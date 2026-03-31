# DashboardSignalEmulator

Эмуляция управляющих сигналов для панели приборов.

- Тахометр
- Уровень топлива
- Вольтметр
- Спидометр

Для эмуляции изменяющихся сигналов используются переменные резисторы 10(100)кОм.

Тахометр значения от 0 до 8000.

АЦП(ADC) 12 bit (0-4095)

Схема подключения:
- Левый контакт резистора GND
- Средний контакт резистора к GPIO(ADC пину)
- Правый контакт резистора 3.3v

Проблемы:
- Нелинейность. АЦП у ESP32 не самый точный, значения в самом начале и в конце шкалы могут «проседать».
- Шумы. Если цифры сильно прыгают, стоит добавить конденсатор на 0.1 мкФ между сигнальным пином и землей или усреднять значения программно.

ADC Oneshot (АЦП) в ESP-IDF.
Основные шаги:
1. Инициализация модуля (Unit): Создаем обработчик для ADC1.
2. Настройка канала (Channel): Указываем пин и аттенюацию (ослабление). Для полного диапазона 0–3.3В используйте ADC_ATTEN_DB_12.
3. Чтение и маппинг: Считываем значение и пересчитываем его пропорцией.

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_adc/adc_oneshot.h"
#include "esp_log.h"

#define POT_ADC_CHANNEL ADC_CHANNEL_6 // GPIO 34 для ESP32
#define MAX_VAL 8000

void app_main(void) {
    // 1. Инициализация ADC Unit
    adc_oneshot_unit_handle_t adc1_handle;
    adc_oneshot_unit_init_cfg_t init_config1 = {
        .unit_id = ADC_UNIT_1,
    };
    ESP_ERROR_CHECK(adc_oneshot_new_unit(&init_config1, &adc1_handle));

    // 2. Конфигурация канала (разрядность по умолчанию 12 бит: 0-4095)
    adc_oneshot_chan_cfg_t config = {
        .bitwidth = ADC_BITWIDTH_DEFAULT,
        .atten = ADC_ATTEN_DB_12, // Позволяет читать до ~3.1-3.3В
    };
    ESP_ERROR_CHECK(adc_oneshot_config_channel(adc1_handle, POT_ADC_CHANNEL, &config));

    int adc_raw;
    while (1) {
        // 3. Чтение сырого значения
        ESP_ERROR_CHECK(adc_oneshot_read(adc1_handle, POT_ADC_CHANNEL, &adc_raw));

        // 4. Масштабирование (Mapping) 0..4095 -> 0..8000
        // Используем long long для промежуточного вычисления, чтобы избежать переполнения
        int mapped_value = (adc_raw * MAX_VAL) / 4095;

        printf("Raw: %d | Mapped: %d\n", adc_raw, mapped_value);

        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```
Убедитесь, что используете пины ADC1 (GPIO 32-39), так как ADC2 нельзя использовать одновременно с работающим Wi-Fi.

Добавить программный фильтр (Moving Average), чтобы значение 8000 не «дрожало» из-за шумов резистора.

АЦП ESP32 имеет нелинейность. Если нужны точные 8000 при максимальном повороте, может потребоваться калибровка.

Сигнал уровня топлива — это медленно меняющийся уровень (аналог).
Сигнал тахометра — это частотный импульсный сигнал (обычно импульсы (Square Wave), частота которых зависит от оборотов).

У ESP32 ЦАП (DAC) 8-битный (0–255). Выход ЦАП выдает от 0 до ~3.3В.

В ESP-IDF LEDC (PWM control) аппаратно генерирует стабильную частоту без нагрузки на процессор.

```c
#include "driver/ledc.h"

#define LEDC_GPIO              18    // Пин, имитирующий выход тахометра
#define LEDC_TIMER             LEDC_TIMER_0
#define LEDC_MODE              LEDC_LOW_SPEED_MODE
#define LEDC_CHANNEL           LEDC_CHANNEL_0
#define LEDC_DUTY_RES          LEDC_TIMER_10_BIT // 10 бит разрешение
#define LEDC_DUTY              (512) // 50% заполнения (меандр)

void init_tachometer() {
    ledc_timer_config_t ledc_timer = {
        .speed_mode       = LEDC_MODE,
        .timer_num        = LEDC_TIMER,
        .duty_resolution  = LEDC_DUTY_RES,
        .freq_hz          = 10,  // Начальная частота
        .clk_cfg          = LEDC_AUTO_CLK
    };
    ledc_timer_config(&ledc_timer);

    ledc_channel_config_t ledc_channel = {
        .speed_mode     = LEDC_MODE,
        .channel        = LEDC_CHANNEL,
        .timer_sel      = LEDC_TIMER,
        .intr_type      = LEDC_INTR_DISABLE,
        .gpio_num       = LEDC_GPIO,
        .duty           = 0, // Изначально выключен
        .hpoint         = 0
    };
    ledc_channel_config(&ledc_channel);
}

// Вызывать при изменении значения резистора
void update_tachometer(int rpm) {
    if (rpm < 100) { // Если обороты слишком малы, выключаем сигнал
        ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, 0);
    } else {
        // Допустим, 8000 RPM = 266 Гц (для 4-цилиндрового двигателя)
        uint32_t freq = rpm / 30;
        ledc_set_freq(LEDC_MODE, LEDC_TIMER, freq);
        ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, 512); // 50% заполнения
    }
    ledc_update_duty(LEDC_MODE, LEDC_CHANNEL);
}
```

Тахометр эмулируем через частоту (PWM/LEDC).

Уровень топлива эмулируем через уровни напряжения (DAC).

Частота: freq = rpm / 60 (для 1 имп/об) или rpm / 30 (для 2 имп/об).
Диапазон: Если накрутили 8000, ledc_set_freq выставит ~133 Гц или ~266 Гц.


Топливный бак (Ступенчатые значения)
Для бака лучше всего использовать массив пороговых значений. Мы делим диапазон потенциометра (0–8000) на участки.



|Значение (0-8000)|Состояние бака |Выход на вторую ESP      |
|-----------------|---------------|-------------------------|
|7001–8000        |Полный (1/1)   |Выход 3.3В (или DAC 255) |
|...              |...            |...                      |
|500–1000         |Резерв (Мигает)|Код/Сигнал "Мигание"     |
|0–500            |Пустой (Горит) |Выход 0В (или DAC 0)     |

```c
// Определяем пороги для бака
#define FUEL_FULL    7000
#define FUEL_HALF    4000
#define FUEL_LOW     1000
#define FUEL_EMPTY   400

void process_signals(int pot_value) {
    // --- ТАХОМЕТР ---
    if (pot_value > 50) { // Игнорируем шум в нуле
        uint32_t freq = pot_value / 30; // Например, 8000 -> 266 Hz
        ledc_set_freq(LEDC_LOW_SPEED_MODE, LEDC_TIMER_0, freq);
        ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 512); // 50% меандр
    } else {
        ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 0);
    }
    ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);

    // --- БАК (Ступенчатая логика) ---
    if (pot_value > FUEL_FULL) {
        // Логика "Полный бак"
    } else if (pot_value < FUEL_EMPTY) {
        // Логика "Горит лампа"
    } else if (pot_value < FUEL_LOW) {
        // Логика "Мигает лампа"
    }
    // и так далее...
}
```

Цифровой сигнал (выделите 3–4 GPIO и подавайте на них комбинации (High/Low) как на переключатель).
Аналоговый сигнал (используйте dac_oneshot_output_voltage). Для каждого порога выдавайте фиксированное напряжение (например: 0.5В, 1.0В, 1.5В и т.д.).

На ESP32 лучше всего использовать ADC1 (пины GPIO 32–39), так как они не конфликтуют с Wi-Fi.

Схема подключения:
- Резистор 1 (Тахометр):
   - 3.3V
   - GPIO 34 (ADC1_CH6)
   - GND
- Резистор 2 (Бак):
   - 3.3V
   - GPIO 35 (ADC1_CH7)
   - GND

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_adc/adc_oneshot.h"

// Каналы для GPIO 34 и 35
#define TACHO_CHAN ADC_CHANNEL_6
#define FUEL_CHAN  ADC_CHANNEL_7

void app_main(void) {
    // 1. Общая настройка модуля ADC1
    adc_oneshot_unit_handle_t adc1_handle;
    adc_oneshot_unit_init_cfg_t init_config1 = {
        .unit_id = ADC_UNIT_1,
    };
    ESP_ERROR_CHECK(adc_oneshot_new_unit(&init_config1, &adc1_handle));

    // 2. Настройка обоих каналов (аттенюация 12dB для диапазона 0-3.3В)
    adc_oneshot_chan_cfg_t config = {
        .bitwidth = ADC_BITWIDTH_DEFAULT,
        .atten = ADC_ATTEN_DB_12,
    };
    ESP_ERROR_CHECK(adc_oneshot_config_channel(adc1_handle, TACHO_CHAN, &config));
    ESP_ERROR_CHECK(adc_oneshot_config_channel(adc1_handle, FUEL_CHAN, &config));

    int raw_tacho, raw_fuel;

    while (1) {
        // Читаем тахометр
        ESP_ERROR_CHECK(adc_oneshot_read(adc1_handle, TACHO_CHAN, &raw_tacho));
        int rpm = (raw_tacho * 8000) / 4095;

        // Читаем бак
        ESP_ERROR_CHECK(adc_oneshot_read(adc1_handle, FUEL_CHAN, &raw_fuel));
        int fuel_level = (raw_fuel * 8000) / 4095; // Тоже в попугаях 0-8000

        // Вывод для проверки
        printf("Тахометр: %d RPM | Бак: %d\n", rpm, fuel_level);

        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

Для тахометра передавайте переменную rpm напрямую в функцию управления частотой (LEDC).
Для Бака используйте конструкцию if / else if или switch, чтобы превратить число fuel_level в нужные вам 6–8 состояний.
Например: if (fuel_level > 7000) state = FULL;)


Нелинейность и шумы на краях диапазона решается тремя способами:
- Аппаратная калибровка (API Calibration V2)
В ESP-IDF 5.x есть встроенная система калибровки, которая учитывает индивидуальные отклонения каждого чипа (записанные в eFuse). Она превращает «сырые» попугаи (0–4095) в честные милливольты (0–3100 мВ). Без калибровки ESP32 «слепнет» уже на 3.0В–3.1В, показывая 4095, хотя на резисторе еще есть запас.

- Программный фильтр (Скользящее среднее)
Чтобы значения не прыгали (например, 7998... 8002... 7995), мы берем среднее из последних 10–20 чтений.

- «Мертвые зоны» (Clamping)
Чтобы гарантированно получать 0 в начале и 8000 в конце, мы искусственно расширяем границы.

```c
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"

// Константы
#define SAMPLE_COUNT 16   // Количество выборок для усреднения
#define TARGET_MAX   8000 // Наш максимум

// Функция для маппинга с "запасом" по краям
int smooth_map(int val, int in_min, int in_max, int out_min, int out_max) {
    if (val <= in_min + 50) return out_min; // Мертвая зона в начале
    if (val >= in_max - 50) return out_max; // Мертвая зона в конце
    return (val - in_min) * (out_max - out_min) / (in_max - in_min);
}

void app_main(void) {
    // 1. Инициализация ADC (как раньше) ...

    // 2. Настройка калибровки (Новое в IDF 5.x)
    adc_cali_handle_t cali_handle = NULL;
    adc_cali_line_fitting_config_t cali_config = {
        .unit_id = ADC_UNIT_1,
        .atten = ADC_ATTEN_DB_12,
        .bitwidth = ADC_BITWIDTH_DEFAULT,
    };
    adc_cali_create_scheme_line_fitting(&cali_config, &cali_handle);

    while (1) {
        uint32_t avg_raw = 0;

        // 3. Усреднение (Фильтр)
        for (int i = 0; i < SAMPLE_COUNT; i++) {
            int raw;
            adc_oneshot_read(adc1_handle, TACHO_CHAN, &raw);
            avg_raw += raw;
        }
        avg_raw /= SAMPLE_COUNT;

        // 4. Получаем милливольты (линейные данные)
        int voltage_mv = 0;
        adc_cali_raw_to_voltage(cali_handle, avg_raw, &voltage_mv);

        // 5. Маппинг (0-3100 мВ -> 0-8000)
        // 3100мВ - это примерный предел ESP32 при 12dB
        int final_value = smooth_map(voltage_mv, 100, 3100, 0, TARGET_MAX);

        printf("Voltage: %d mV | Final: %d\n", voltage_mv, final_value);
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}
```
1. Снизу (около 0): smooth_map отсекает шумы. Если на пине 50мВ, мы считаем это за 0.
2. Сверху (около 8000): Из-за калибровки мы маппим не до 4095, а до ~3100 мВ. Это убирает «полку» в конце хода резистора.
3. Дрожание: SAMPLE_COUNT (16 или 32) делает движение стрелки тахометра или бака плавным.

Аппаратная калибровка в ESP-IDF работает «под капотом», но вам нужно правильно её инициализировать в коде. Она не требует от вас вращения резистора — она использует заводские данные, записанные в память eFuse каждого конкретного чипа ESP32 при производстве.

На заводе Espressif измеряет опорное напряжение (Vref) вашего чипа и записывает поправки в eFuse.
- Без калибровки: Вы получаете «попугаи» (0–4095), которые у одного чипа значат 3.1В, а у другого 3.25В.
- С калибровкой: Библиотека esp_adc/adc_cali.h считывает эти поправки и пересчитывает «попугаи» в реальные милливольты (мВ).

Для чипов ESP32 (оригинальных) используется схема line_fitting. Для новых (C3, S3) — curve_fitting.

Пример инициализации калибровки:
```c
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"

adc_cali_handle_t cali_handle = NULL;

// 1. Создаем схему калибровки
adc_cali_line_fitting_config_t cali_config = {
    .unit_id = ADC_UNIT_1,
    .atten = ADC_ATTEN_DB_12,           // Должно совпадать с настройкой канала!
    .bitwidth = ADC_BITWIDTH_DEFAULT,
};

// 2. Проверяем, поддерживается ли калибровка в eFuse, и создаем обработчик
esp_err_t ret = adc_cali_create_scheme_line_fitting(&cali_config, &cali_handle);

if (ret == ESP_OK) {
    printf("Калибровка успешно инициализирована\n");
}
```

Теперь вместо того, чтобы гадать, что значит число 3800, вы вызываете функцию пересчета:
```c
int raw_value;
int voltage_mv;

// Сначала читаем "сырые" данные
adc_oneshot_read(adc1_handle, TACHO_CHAN, &raw_value);

// Переводим их в милливольты с учетом заводской калибровки
adc_cali_raw_to_voltage(cali_handle, raw_value, &voltage_mv);

// Теперь маппим уже милливольты (0 - 3100мВ) в (0 - 8000)
int rpm = (voltage_mv * 8000) / 3100;
```

1. Нижний край: У ESP32 АЦП не видит ничего ниже ~100 мВ. Калибровка честно скажет вам 0 или около того, а программная «мертвая зона» (о которой мы говорили выше) уберет шум.

2. Верхний край: При аттенюации 12дБ АЦП «насыщается» (показывает максимум 4095) уже при напряжении около 3.1В, хотя питание у нас 3.3В.
  - Без калибровки: Вы докрутили резистор на 90%, а в коде уже 8000. Последние 10% хода ничего не меняют.
  - С калибровкой: Вы будете точно знать, когда достигнут предел в 3100 мВ, и сможете растянуть шкалу программно именно до этой точки.

Если adc_cali_create_scheme_line_fitting вернет ошибку, это значит, что в чип не записали Vref. В этом случае библиотека использует «типичное значение» (1100 мВ), что все равно точнее, чем просто делить на 4095.


Калибровка — это не поиск «точек слепоты» (краев), а построение графика точности внутри рабочего диапазона.
АЦП — это линейка. На заводе знают, что эта «линейка» у ESP32 немного кривая:
- Смещение (Offset): Она может начинать считать не с 0, а, например, с 50 мВ.
- Наклон (Gain): Вместо того чтобы на каждый 1 вольт прибавлять 1000 делений, она прибавляет 1050.

Что именно записывают в eFuse:
В память чипа записывают коэффициенты (наклон и смещение). Когда вы вызываете adc_cali_raw_to_voltage(), драйвер берет ваше «кривое» сырое число и по формуле выпрямляет его в реальные милливольты.


А что с «краями» (где он перестает различать)?
Это называется насыщением (Saturation), и калибровка это не исправляет, она просто дает вам знать, где этот предел наступил.
- Нижний предел: ESP32 физически не видит напряжения ниже ~100 мВ. Хоть обкалибруйтесь, всё, что ниже, будет «шумом» или нулем.
- Верхний предел: При максимальном усилении (12dB) АЦП «зашкаливает» на уровне ~3100 мВ, хотя питание платы 3300 мВ.

Как мы это используем для ваших 8000:
Зная благодаря калибровке, что «потолок» наступил на 3100 мВ, мы в коде говорим: «Все, что выше 3050 мВ — это железно 8000». Это и убирает проблему, когда вы крутите ручку до упора, а значения на 7900 начинают бешено прыгать.

Резюме: Калибровка делает шкалу линейной (чтобы 4000 было ровно посередине между 0 и 8000), а проблему краев мы закрываем программным «зажимом» (clamping) в коде.


Итог. мертвые зоны по краям не исчезнут, но данные будут четко 0 и четко 8000. В электронике и разработке интерфейсов это называется «мертвыми зонами» (Deadbands).

- При повороте ручки до упора влево он гарантированно видел 0, а не прыгающие 1, 0, 3, 0.
- При повороте до упора вправо он гарантированно видел 8000, а не застрявшее на 7942 значение из-за того, что АЦП «насытился» раньше, чем резистор дошел до конца.

Мы берем диапазон, который ESP32 видит уверенно (например, от 100 мВ до 3050 мВ), и программно растягиваем его на ваши 0–8000.
- Все, что ниже 100 мВ, принудительно превращаем в 0.
- Все, что выше 3050 мВ, принудительно превращаем в 8000.

```c
int get_calibrated_value(int voltage_mv) {
    const int V_MIN = 100;  // С какого напряжения начинаем считать (нижняя мертвая зона)
    const int V_MAX = 3050; // На каком напряжении уже отдаем максимум (верхняя мертвая зона)
    const int TARGET = 8000;

    if (voltage_mv <= V_MIN) return 0;
    if (voltage_mv >= V_MAX) return TARGET;

    // Линейная интерполяция между V_MIN и V_MAX
    return (voltage_mv - V_MIN) * TARGET / (V_MAX - V_MIN);
}
```

- Начало хода: Вы начинаете крутить резистор, первые пару градусов ничего не происходит (ноль стабилен), затем значение плавно растет.
- Конец хода: Вы еще не докрутили резистор до самого упора (осталось чуть-чуть), но значение уже встало в «железные» 8000 и не шелохнется.


Проблема фильтра (Скользящее среднее)
У фильтра есть инерция. Если вы резко крутанете ручку в упор, фильтр будет еще несколько миллисекунд «дотягивать» среднее значение до 8000.
Результат: Вы уже физически в упоре, а на экране цифры быстро бегут: 7850... 7920... 7988... 8000.
Решение: Мы ставим проверку после фильтра. Если усредненное значение попало в «мертвую зону» (например, выше 3050 мВ), мы мгновенно выдаем ровно 8000, игнорируя остатки вычислений фильтра.

Мертвая зона — это наша «домашняя» калибровка
Заводская калибровка делает шкалу линейной (честной), а «мертвая зона» делает её комфортной.
Заводская: Говорит нам: «Друг, 4095 попугаев — это на самом деле 3124 милливольта».
Наша (Мертвая зона): Мы говорим: «Нам плевать, что там 3124 или 3100. Как только напряжение перевалило за 3050, пиши 8000. А всё, что ниже 100, пиши 0».


```c
// 1. Сначала фильтруем (сглаживаем дрожание)
int avg_voltage = get_moving_average(raw_voltage);

// 2. Затем применяем наши границы (Мертвые зоны)
int final_val;
if (avg_voltage >= 3050) {
    final_val = 8000; // "Железный" максимум
} else if (avg_voltage <= 100) {
    final_val = 0;    // "Железный" минимум
} else {
    // Только если мы внутри нормального диапазона, считаем пропорцию
    final_val = (avg_voltage - 100) * 8000 / (3050 - 100);
}
```

Тахометр: При 8000 оборотах стрелка на эмулируемом приборе не должна дрожать на 7988. Она должна «прилипнуть» к ограничителю.

Бак: У резисторов на 100кОм часто по краям напыление стирается или имеет плохой контакт. Мертвая зона в 100–150 мВ полностью скрывает этот дефект.


Использование структур (struct) позволит нам настроить каждый датчик индивидуально. Например, для тахометра нам важна скорость реакции, а для бака — максимальная стабильность (чтобы стрелка не дрожала от малейшего касания резистора).
Мы создадим структуру, которая хранит настройки «мертвых зон» и текущее сглаженное значение.
```c
#include <stdio.h>
#include "esp_adc/adc_oneshot.h"
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"

// Структура для описания датчика
typedef struct {
    adc_channel_t channel; // Пин (канал)
    int v_min;             // Нижняя мертвая зона (мВ)
    int v_max;             // Верхняя мертвая зона (мВ)
    int target_max;        // Максимум шкалы (например, 8000)
    float filter_alpha;    // Коэффициент плавности (0.01 - медленно, 0.5 - быстро)
    float current_val;     // Хранилище для фильтра
} Potentiometer_t;

// Инициализируем наши "приборы"
Potentiometer_t tacho = { .channel = ADC_CHANNEL_6, .v_min = 120, .v_max = 3050, .target_max = 8000, .filter_alpha = 0.2 };
Potentiometer_t fuel  = { .channel = ADC_CHANNEL_7, .v_min = 150, .v_max = 3000, .target_max = 8000, .filter_alpha = 0.05 };

// Функция обработки
int process_pot(Potentiometer_t *p, adc_oneshot_unit_handle_t ad_handle, adc_cali_handle_t cal_handle) {
    int raw, mv;
    adc_oneshot_read(ad_handle, p->channel, &raw);
    adc_cali_raw_to_voltage(cal_handle, raw, &mv);

    // 1. Экспоненциальное сглаживание (проще и эффективнее среднего)
    p->current_val = (mv * p->filter_alpha) + (p->current_val * (1.0 - p->filter_alpha));

    // 2. Применение мертвых зон и маппинг
    if (p->current_val <= p->v_min) return 0;
    if (p->current_val >= p->v_max) return p->target_max;

    return (int)((p->current_val - p->v_min) * p->target_max / (p->v_max - p->v_min));
}

void app_main(void) {
    // ... здесь инициализация ADC1 и калибровки (как обсуждали раньше) ...

    while (1) {
        int rpm_val = process_pot(&tacho, adc1_handle, cali_handle);
        int fuel_val = process_pot(&fuel, adc1_handle, cali_handle);

        printf("RPM: %d | FUEL: %d\n", rpm_val, fuel_val);

        vTaskDelay(pdMS_TO_TICKS(50));
    }
}
```

Индивидуальный фильтр (filter_alpha):
- У тахометра (0.2) — стрелка будет "живой", быстро прыгая за рукой.
- У бака (0.05) — значение будет меняться очень плавно и лениво, полностью игнорируя мелкий дребезг контактов резистора.

Разные зоны:
- Если резистор тахометра качественный, ему можно поставить v_min = 80.
- Если резистор бака старый и в конце "шумит", ставим ему v_max = 2900, чтобы он гарантированно выдавал "полный бак" чуть раньше физического упора.

Масштабируемость:
- Если завтра понадобится добавить датчик температуры масла — просто создаете еще одну структуру в одну строчку.


Для бака нам нужно превратить плавное значение 0–8000 в дискретные состояния (пустой, 1/4, 1/2 и т.д.). Главная проблема здесь — «дребезг» на границах: если бак заполнен ровно наполовину, стрелка или лампа могут начать быстро переключаться туда-сюда из-за микро-шумов.
Чтобы этого избежать, мы используем гистерезис (окно задержки).


1. Определяем состояния бака
Создадим перечисление (enum), чтобы код был читаемым:
```c
typedef enum {
    FUEL_EMPTY_ALARM = 0, // Лампа горит (совсем пусто)
    FUEL_RESERVE_BLINK,   // Лампа мигает
    FUEL_1_4,
    FUEL_1_2,
    FUEL_3_4,
    FUEL_FULL             // Полный бак
} fuel_level_t;
```

2. Реализация функции со «ступеньками» и гистерезисом
Мы будем менять состояние только тогда, когда значение перешло порог на определенную величину (например, +/- 100 единиц).

```c
fuel_level_t calculate_fuel_state(int current_val) {
    static fuel_level_t last_state = FUEL_EMPTY_ALARM;
    const int HYS = 150; // Гистерезис (окно в 150 единиц)

    // Логика порогов (значения 0-8000)
    if (current_val > 7000 + HYS) last_state = FUEL_FULL;
    else if (current_val < 7000 - HYS && current_val > 5000 + HYS) last_state = FUEL_3_4;
    else if (current_val < 5000 - HYS && current_val > 3000 + HYS) last_state = FUEL_1_2;
    else if (current_val < 3000 - HYS && current_val > 1500 + HYS) last_state = FUEL_1_4;
    else if (current_val < 1500 - HYS && current_val > 600 + HYS)  last_state = FUEL_RESERVE_BLINK;
    else if (current_val < 600 - HYS) last_state = FUEL_EMPTY_ALARM;

    return last_state;
}
```

3. Как это работает на практике
- Вы крутите резистор вверх. Как только значение стало 7151, бак переключается в статус FUEL_FULL.
- Вы чуть-чуть задели резистор обратно, значение стало 6950. Статус не изменится! Он останется «Полным», пока вы не опуститесь ниже 6850.
- Это создает эффект «четкого щелчка» между делениями, как на настоящей приборной панели.


4. Итоговое применение в app_main
Теперь в основном цикле вы получаете и точное значение для тахометра, и стабильную «ступеньку» для бака:
```c
while (1) {
    // 1. Получаем чистые данные 0-8000
    int rpm_raw = process_pot(&tacho, adc1_handle, cali_handle);
    int fuel_raw = process_pot(&fuel, adc1_handle, cali_handle);

    // 2. Превращаем бак в ступеньки
    fuel_level_t fuel_status = calculate_fuel_state(fuel_raw);

    // 3. Выводим результат
    printf("Тахометр: %d RPM | Статус бака: %d\n", rpm_raw, fuel_status);

    // Здесь можно добавить switch(fuel_status) для управления светодиодами
    // или отправки данных на вторую ESP32

    vTaskDelay(pdMS_TO_TICKS(100));
}
```

Для имитации «мигающей лампы» (состояние FUEL_RESERVE_BLINK) вам понадобится простой таймер или проверка системного времени, чтобы менять состояние светодиода каждые 500 мс.
Если мы говорим о «фиксированном мигании» для режима резерва топлива, проще всего реализовать это через проверку системного времени (esp_timer_get_time). Это не блокирует основной цикл и не требует сложных прерываний.

Как реализовать мигание без задержек (Non-blocking):
Добавим логику, которая меняет состояние переменной каждые 500 мс.

```c
#include "driver/gpio.h"
#include "esp_timer.h"

#define LED_FUEL_PIN 2  // Пин лампы топлива

void update_fuel_led(fuel_level_t status) {
    static bool led_state = false;
    static int64_t last_toggle = 0;
    int64_t now = esp_timer_get_time(); // Время в микросекундах

    switch (status) {
        case FUEL_EMPTY_ALARM:
            gpio_set_level(LED_FUEL_PIN, 1); // Горит постоянно
            break;

        case FUEL_RESERVE_BLINK:
            // Мигаем раз в 500мс (500000 мкс)
            if (now - last_toggle > 500000) {
                led_state = !led_state;
                gpio_set_level(LED_FUEL_PIN, led_state);
                last_toggle = now;
            }
            break;

        default:
            gpio_set_level(LED_FUEL_PIN, 0); // В остальных случаях выключена
            break;
    }
}
```

Почему это удобно:
- Фиксированная частота: Лампа будет мигать ровно, даже если основной цикл while(1) притормозит на других задачах.
- Простота: Вы просто вызываете update_fuel_led(fuel_status) в основном цикле после обработки резистора.
- Гибкость: Если решите использовать BSP функции (например, bsp_led_set_rgb), логику switch можно легко адаптировать под них.

Итоговая архитектура системы:
- Резистор 1 (Tacho): ADC -> Фильтр -> Мертвые зоны -> 0-8000 -> LEDC (Частота).
- Резистор 2 (Fuel): ADC -> Фильтр -> Мертвые зоны -> 0-8000 -> Гистерезис (Ступеньки) -> Логика LED (Горит/Мигает/Выкл).


В мире встраиваемых систем (Embedded) использование vTaskDelay() или delay() для мигания — это плохой тон, потому что на это время весь код «засыпает».

Почему проверка времени (esp_timer_get_time) лучше:
- Многозадачность без RTOS-сложностей: Ваш цикл while(1) крутится максимально быстро (например, каждые 10-20 мс). Он успевает плавно обновлять тахометр, фильтровать шумы и проверять состояние бака.
- Точность: Вы просто спрашиваете у процессора: «Эй, уже прошло 500мс с момента последнего переключения светодиода?». Если нет — код идет дальше по своим делам, не тратя ни микросекунды на ожидание.
- Стабильность: Частота мигания не будет зависеть от того, сколько времени занимают другие вычисления (чтение АЦП или вывод в Serial).


```c
while (1) {
    // 1. Моментальное чтение и фильтрация (не ждем)
    int rpm = process_pot(&tacho, adc1_handle, cali_handle);
    int fuel_raw = process_pot(&fuel, adc1_handle, cali_handle);

    // 2. Моментальное обновление частоты тахометра
    update_tachometer_pwm(rpm);

    // 3. Логика бака с гистерезисом (простая математика)
    fuel_level_t fuel_status = calculate_fuel_state(fuel_raw);

    // 4. Умное управление светодиодом (через проверку времени)
    update_fuel_led_non_blocking(fuel_status);

    // 5. Минимальная пауза для "дыхания" FreeRTOS
    vTaskDelay(pdMS_TO_TICKS(20));
}
```

При таком подходе:
- Тахометр откликается мгновенно (высокий FPS).
- Бак переключается четко (гистерезис).
- Лампа мигает ровно (таймер).
- Процессор свободен для других задач (например, если захотите добавить Wi-Fi).


Вы правы, vTaskDelay() действительно работает через планировщик FreeRTOS и не замораживает процессор целиком.
Но есть нюанс: vTaskDelay() замораживает выполнение именно ВАШЕГО кода (конкретного Task).

В чем ловушка для вашей задачи:
Если вы напишете мигалку для бака внутри основного цикла вот так:
```c
// ПЛОХОЙ ПРИМЕР
while(1) {
    read_tacho(); // Читаем тахометр
    if (reserve) {
        gpio_set_level(LED, 1);
        vTaskDelay(pdMS_TO_TICKS(500)); // Ждем полсекунды
        gpio_set_level(LED, 0);
        vTaskDelay(pdMS_TO_TICKS(500)); // Еще полсекунды
    }
}
```

Проблема: Пока идет этот vTaskDelay(500), функция read_tacho() не вызывается. Если вы в этот момент резко крутанете ручку тахометра, ESP32 «увидит» это только через секунду, когда мигалка закончит свой цикл. Тахометр будет жутко тормозить.


Почему проверка времени (esp_timer_get_time) лучше:
Она позволяет вашему циклу работать молниеносно (например, 100 раз в секунду).
- На 1-й итерации: «Время мигать? Нет. Обновил тахометр. Уснул на 10мс».
- На 2-й итерации: «Время мигать? Нет. Обновил тахометр. Уснул на 10мс».
...
- На 50-й итерации: «О, прошло 500мс! Переключил диод. Обновил тахометр. Уснул на 10мс».
Итог: Тахометр обновляется каждые 10мс (плавно), а лампа мигает свои 500мс (ровно). Вы не «крадете» время у тахометра ради лампочки.



Две независимые структуры для датчиков, заводская калибровка, экспоненциальный фильтр (чтобы не было инерции среднего), мертвые зоны, гистерезис для ступенек бака и неблокирующее мигание лампы.

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "driver/ledc.h"
#include "esp_adc/adc_oneshot.h"
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"
#include "esp_timer.h"

// --- НАСТРОЙКИ GPIO ---
#define PIN_TACHO_ADC    ADC_CHANNEL_6 // GPIO 34
#define PIN_FUEL_ADC     ADC_CHANNEL_7 // GPIO 35
#define PIN_TACHO_OUT    GPIO_NUM_18   // Выход частоты для другой ESP
#define PIN_FUEL_LED     GPIO_NUM_2    // Лампа остатка топлива

// --- СТРУКТУРЫ И ТИПЫ ---
typedef enum {
    FUEL_EMPTY = 0, FUEL_RESERVE_BLINK, FUEL_1_4, FUEL_1_2, FUEL_3_4, FUEL_FULL
} fuel_level_t;

typedef struct {
    adc_channel_t channel;
    int v_min;         // Мертвая зона снизу (мВ)
    int v_max;         // Мертвая зона сверху (мВ)
    int target_max;    // Шкала (0-8000)
    float alpha;       // Коэффициент фильтра (0.05 - плавно, 0.3 - быстро)
    float current_mv;  // Состояние фильтра
} sensor_t;

// --- ГЛОБАЛЬНЫЕ ОБЪЕКТЫ ---
static adc_oneshot_unit_handle_t adc1_handle;
static adc_cali_handle_t cali_handle = NULL;

sensor_t tacho = { .channel = PIN_TACHO_ADC, .v_min = 120, .v_max = 3050, .target_max = 8000, .alpha = 0.2 };
sensor_t fuel  = { .channel = PIN_FUEL_ADC,  .v_min = 150, .v_max = 3000, .target_max = 8000, .alpha = 0.05 };

// --- ФУНКЦИИ ОБРАБОТКИ ---

// Чтение, калибровка, фильтрация и маппинг в одном флаконе
int process_sensor(sensor_t *s) {
    int raw, mv;
    adc_oneshot_read(adc1_handle, s->channel, &raw);
    adc_cali_raw_to_voltage(cali_handle, raw, &mv);

    // Экспоненциальное сглаживание
    s->current_mv = (mv * s->alpha) + (s->current_mv * (1.0f - s->alpha));

    // Применение мертвых зон
    if (s->current_mv <= s->v_min) return 0;
    if (s->current_mv >= s->v_max) return s->target_max;

    // Линейный маппинг внутри рабочего диапазона
    return (int)((s->current_mv - s->v_min) * s->target_max / (s->v_max - s->v_min));
}

// Логика ступенек бака с гистерезисом
fuel_level_t get_fuel_status(int val) {
    static fuel_level_t state = FUEL_EMPTY;
    const int H = 150; // Окно гистерезиса

    if (val > 7000 + H) state = FUEL_FULL;
    else if (val < 7000 - H && val > 5000 + H) state = FUEL_3_4;
    else if (val < 5000 - H && val > 3000 + H) state = FUEL_1_2;
    else if (val < 3000 - H && val > 1500 + H) state = FUEL_1_4;
    else if (val < 1500 - H && val > 600 + H)  state = FUEL_RESERVE_BLINK;
    else if (val < 600 - H) state = FUEL_EMPTY;

    return state;
}

// Неблокирующее управление лампой (мигание через системное время)
void update_fuel_visuals(fuel_level_t status) {
    static int64_t last_blink = 0;
    static bool led_on = false;
    int64_t now = esp_timer_get_time();

    if (status == FUEL_EMPTY) {
        gpio_set_level(PIN_FUEL_LED, 1);
    } else if (status == FUEL_RESERVE_BLINK) {
        if (now - last_blink > 500000) { // 500мс
            led_on = !led_on;
            gpio_set_level(PIN_FUEL_LED, led_on);
            last_blink = now;
        }
    } else {
        gpio_set_level(PIN_FUEL_LED, 0);
    }
}

// Генерация частоты тахометра (LEDC)
void update_tacho_signal(int rpm) {
    if (rpm < 50) {
        ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 0);
    } else {
        uint32_t freq = rpm / 30; // Пример: 8000 RPM -> ~266 Hz
        ledc_set_freq(LEDC_LOW_SPEED_MODE, LEDC_TIMER_0, freq);
        ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 512); // 50% меандр
    }
    ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
}

// --- ИНИЦИАЛИЗАЦИЯ ПЕРИФЕРИИ ---
void init_hardware() {
    // 1. ADC & Cali
    adc_oneshot_unit_init_cfg_t init_cfg = { .unit_id = ADC_UNIT_1 };
    adc_oneshot_new_unit(&init_cfg, &adc1_handle);
    adc_oneshot_chan_cfg_t ch_cfg = { .bitwidth = ADC_BITWIDTH_DEFAULT, .atten = ADC_ATTEN_DB_12 };
    adc_oneshot_config_channel(adc1_handle, PIN_TACHO_ADC, &ch_cfg);
    adc_oneshot_config_channel(adc1_handle, PIN_FUEL_ADC, &ch_cfg);

    adc_cali_line_fitting_config_t cali_cfg = { .unit_id = ADC_UNIT_1, .atten = ADC_ATTEN_DB_12, .bitwidth = ADC_BITWIDTH_DEFAULT };
    adc_cali_create_scheme_line_fitting(&cali_cfg, &cali_handle);

    // 2. GPIO
    gpio_reset_pin(PIN_FUEL_LED);
    gpio_set_direction(PIN_FUEL_LED, GPIO_MODE_OUTPUT);

    // 3. LEDC (Tacho PWM)
    ledc_timer_config_t lt = { .speed_mode = LEDC_LOW_SPEED_MODE, .timer_num = LEDC_TIMER_0, .duty_resolution = LEDC_TIMER_10_BIT, .freq_hz = 10, .clk_cfg = LEDC_AUTO_CLK };
    ledc_timer_config(&lt);
    ledc_channel_config_t lc = { .speed_mode = LEDC_LOW_SPEED_MODE, .channel = LEDC_CHANNEL_0, .timer_sel = LEDC_TIMER_0, .gpio_num = PIN_TACHO_OUT, .duty = 0, .hpoint = 0 };
    ledc_channel_config(&lc);
}

void app_main(void) {
    init_hardware();

    while (1) {
        // Читаем датчики (очень быстро)
        int rpm_val = process_sensor(&tacho);
        int fuel_val = process_sensor(&fuel);

        // Обновляем сигналы
        update_tacho_signal(rpm_val);
        fuel_level_t f_status = get_fuel_status(fuel_val);
        update_fuel_visuals(f_status);

        // Отладочный вывод
        printf("RPM: %4d | FUEL: %4d (Status: %d)\n", rpm_val, fuel_val, f_status);

        vTaskDelay(pdMS_TO_TICKS(20)); // Цикл крутится 50 раз в секунду
    }
}
```

Что этот код делает:
1. RPM (Тахометр): На выходе GPIO 18 генерирует частоту, пропорциональную повороту резистора. Стрелка будет двигаться плавно благодаря alpha = 0.2.
2. FUEL (Бак):
  - Превращает 0–8000 в 6 состояний.
  - Если уровень в резерве — лампа на GPIO 2 мигает.
  - Если совсем пусто — горит постоянно.
  - Если больше резерва — гаснет.
  - Переключение между делениями четкое за счет гистерезиса.
3. Надежность: Вы можете крутить оба резистора одновременно — они не мешают друг другу, так как код неблокирующий.


Использование отдельной задачи для мигания — это классический подход в RTOS, и он абсолютно правильный, если вы хотите разделить логику «мозгов» (чтение датчиков) и «индикации».

- Нагрузка: Задача, которая большую часть времени спит в vTaskDelay(), потребляет 0% ресурсов процессора. Процессор «просыпается» только на долю миллисекунды, чтобы переключить пин.
- Вытеснение: Если у задачи мигания приоритет ниже (например, 1), а у основной логики выше (например, 5), то система всегда поставит в приоритет чтение тахометра. Светодиод мигнет чуть позже (на микросекунду), что человеческому глазу незаметно.


Как это реализовать правильно (через очереди или события):
Чтобы задача мигания знала, в каком режиме работать, мы будем передавать ей статус бака.

```c
#include "freertos/queue.h"

// Очередь для передачи статуса бака в задачу мигания
static QueueHandle_t fuel_status_queue;

void fuel_led_task(void *pvParameters) {
    fuel_level_t current_status = FUEL_FULL;
    bool led_state = false;

    while (1) {
        // Проверяем, пришел ли новый статус из основной задачи (не ждем долго)
        xQueueReceive(fuel_status_queue, &current_status, 0);

        if (current_status == FUEL_EMPTY) {
            gpio_set_level(PIN_FUEL_LED, 1);
            vTaskDelay(pdMS_TO_TICKS(100)); // Просто спим
        }
        else if (current_status == FUEL_RESERVE_BLINK) {
            led_state = !led_state;
            gpio_set_level(PIN_FUEL_LED, led_state);
            vTaskDelay(pdMS_TO_TICKS(500)); // Период мигания
        }
        else {
            gpio_set_level(PIN_FUEL_LED, 0);
            vTaskDelay(pdMS_TO_TICKS(100)); // Спим, пока бак полный
        }
    }
}
```

```c
void app_main(void) {
    init_hardware();

    // Создаем очередь на 1 элемент
    fuel_status_queue = xQueueCreate(1, sizeof(fuel_level_t));

    // Запускаем задачу на другом ядре (или просто с низким приоритетом)
    // Приоритет 1 (низкий), ядро 1
    xTaskCreatePinnedToCore(fuel_led_task, "fuel_led", 2048, NULL, 1, NULL, 1);

    while (1) {
        int rpm = process_sensor(&tacho);
        int fuel_raw = process_sensor(&fuel);

        fuel_level_t status = get_fuel_status(fuel_raw);

        // Отправляем статус в задачу мигания (перезаписываем старый)
        xQueueOverwrite(fuel_status_queue, &status);

        update_tacho_signal(rpm);
        vTaskDelay(pdMS_TO_TICKS(20));
    }
}
```


Преимущества такого разделения:
- Чистота кода: Основной цикл не знает о «миллисекундах мигания», он просто говорит: «Сейчас бак в резерве».
- Надежность: Если основной цикл зависнет на тяжелых вычислениях, RTOS всё равно переключит контекст на задачу светодиода, и он продолжит мигать.
- Масштабируемость: В задачу светодиода можно потом добавить звук (зуммер) или отправку данных по сети, не трогая быстрый цикл тахометра.


Как переделать тахометр на реальную частоту:
Железо: GPIO ESP32 выдает 3.3В. Если вход тахометра в другом устройстве/авто рассчитан на 5В или 12В, вам понадобится простейший каскад на одном транзисторе (например, 2N2222 или BC547) по схеме «открытый коллектор».
Частота (Формула): Сейчас мы использовали rpm / 30. Это соответствует 2 импульсам на один оборот коленвала (стандарт для 4-цилиндрового двигателя). Если у вас другой двигатель, просто поменяйте делитель:
- 1 имп/об: freq = rpm / 60
- 2 имп/об: freq = rpm / 30
- 3 имп/об: freq = rpm / 20



```c
#include "driver/dac_oneshot.h"
// ... внутри цикла ...
// Мапим 0-8000 в 0-255 (8 бит ЦАП)
uint8_t dac_val = (fuel_val * 255) / 8000;
dac_oneshot_output_voltage(dac_handle, dac_val);
```


Первая ESP32 (Машина):
- Считывает ваш физический резистор (тот, что вы крутите рукой).
- Превращает положение ручки в число 0–8000 (используя фильтры и калибровку, которые мы написали).
- Генерирует на своем выходе (GPIO 18) реальную частоту (импульсы), пропорциональную этому числу.
- Эта частота по проводу улетает на вторую плату.


Вторая ESP32 (Приборная панель):
- Не знает про ваш резистор. Она «видит» только импульсы на своем входном пине.
- Считает эти импульсы (через прерывания или счетчик PCNT).
- Пересчитывает частоту обратно в обороты и рисует их на экране.


Для стабильного чтения частоты на второй плате лучше всего использовать периферию PCNT (Pulse Counter). Она считает импульсы аппаратно, не нагружая процессор.
```c
#include "driver/pulse_cnt.h"

// Настройка счетчика
pcnt_unit_config_t unit_config = {
    .high_limit = 10000,
    .low_limit = -1,
};
pcnt_unit_handle_t pcnt_unit = NULL;
pcnt_new_unit(&unit_config, &pcnt_unit);

pcnt_chan_config_t chan_config = {
    .edge_gpio_num = 4, // Пин, куда пришел провод от первой ESP
    .level_gpio_num = -1,
};
pcnt_channel_handle_t pcnt_chan = NULL;
pcnt_new_channel(pcnt_unit, &chan_config, &pcnt_chan);

// Установка действия: считать по положительному фронту
pcnt_channel_set_edge_action(pcnt_chan, PCNT_CHANNEL_EDGE_ACTION_INCREASE, PCNT_CHANNEL_EDGE_ACTION_HOLD);

pcnt_unit_enable(pcnt_unit);
pcnt_unit_start(pcnt_unit);

// Раз в секунду (или чаще) читаем:
int pulses = 0;
pcnt_unit_get_count(pcnt_unit, &pulses);
pcnt_unit_clear_count(pcnt_unit);
// Обороты = pulses * 60 / (количество импульсов на оборот)
```


В микроконтроллерах АЦП (ADC) — это один физический модуль, который просто переключается между разными пинами (каналами). Его «железное» разрешение всегда одинаковое (для ESP32 это 12 бит, то есть 0–4095).
Разделить их на уровне «железа» (один датчик в 8000, другой в 6) невозможно, поэтому мы делаем это программно сразу после чтения.
Мы читаем оба канала одним и тем же методом, но мапим (масштабируем) их в разные диапазоны.

```c
// 1. Читаем сырые данные (0-4095)
int raw_tacho, raw_fuel;
adc_oneshot_read(adc1_handle, TACHO_CHAN, &raw_tacho);
adc_oneshot_read(adc1_handle, FUEL_CHAN, &raw_fuel);

// 2. Масштабируем программно под разные нужды:

// Тахометр: нам нужна высокая точность для генерации частоты
int rpm = (raw_tacho * 8000) / 4095;

// Бак: нам достаточно 6 делений
int fuel_steps = (raw_fuel * 6) / 4095;
```

Почему ваш подход 0–8000 для обоих — это даже лучше:
Если вы приведете оба датчика к диапазону 0–8000, это даст вам:
- Универсальность: Вы используете одну и ту же функцию фильтрации и калибровки для обоих.
- Точность для бака: Имея число 0–8000, вы можете сделать очень плавный гистерезис. Если бы у вас было всего 0–6, вы бы не смогли отфильтровать дребезг резистора (любое колебание сразу перекидывало бы деление бака).
- Гибкость: Если завтра вы решите, что в баке должно быть не 6 делений, а 10 или 12, вам не придется переписывать логику чтения АЦП — вы просто поменяете пороги в функции get_fuel_status.

Итог по архитектуре:
Ваша логика верна:
- Чтение: Оба в 0–8000 (максимальное разрешение).
- Обработка Тacho: Используем 0–8000 напрямую для формирования частоты ledc.
- Обработка Fuel: Используем 0–8000 как вход для функции «ступенек», которая на выходе выдаст номер текущего деления (0–5).


Чтобы диапазон 0–8000 ровно делился на 6 зон, проще всего разбить его на отрезки по 1333 единицы. Однако для реального бака зоны обычно не равны: «резерв» всегда меньше, чем «полный бак».
Вот как это реализовать с ровным делением и добавлением гистерезиса, чтобы стрелка не дрожала на границах зон.
1. Математика деления
Если делить 8000 на 6 зон:
- 0–1333: Пусто
- 1334–2666: 1/4
- ... и так далее.
Но лучше использовать центральные точки и пороги.


```c
#define FUEL_MAX 8000
#define FUEL_ZONES 6
#define HYSTERESIS 150  // Защитное "окно", чтобы деление не прыгало

int get_fuel_step(int value) {
    static int last_step = 0;

    // Вычисляем размер одного шага (8000 / 6 = 1333)
    const int step_size = FUEL_MAX / FUEL_ZONES;

    // Рассчитываем текущий шаг на основе сырого значения
    int current_step = value / step_size;

    // Ограничитель, чтобы не выйти за массив (0-5)
    if (current_step >= FUEL_ZONES) current_step = FUEL_ZONES - 1;

    // Логика гистерезиса:
    // Переключаем шаг только если отклонение больше порога HYSTERESIS
    // Это предотвращает "дребезг" индикатора на границе зон
    int threshold_low = (last_step * step_size) - HYSTERESIS;
    int threshold_high = ((last_step + 1) * step_size) + HYSTERESIS;

    if (value > threshold_high || value < threshold_low) {
        last_step = current_step;
    }

    return last_step;
}

```

3. Почему это удобно:
- Чистые индексы: Функция вернет число от 0 до 5. Вам легко использовать это в массиве картинок бака на экране: draw_image(icons[step]).
- Никакого мерцания: Если резистор стоит ровно на отметке 1333, и значение из-за шума прыгает 1332-1334, гистерезис удержит старое состояние, пока вы явно не крутанете ручку дальше.
- Легко менять: Если решите сделать 8 зон, просто поменяйте #define FUEL_ZONES 8, и вся математика пересчитается сама.


4. Дополнительно: Определение "Резерва"
Обычно "Резерв" (мигание) — это когда step == 0 или step == 1.

```c
int fuel_step = get_fuel_step(fuel_8000_val);

if (fuel_step == 0) {
    // Совсем пусто - горит лампа
} else if (fuel_step == 1) {
    // Резерв - мигает лампа
} else {
    // Все ок - лампа выключена, рисуем уровень
}
```


