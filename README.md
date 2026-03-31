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

























