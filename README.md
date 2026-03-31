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
|-----------------|-------------- |-------------------------|
|...              |...            |...                      |
|-----------------|---------------|-------------------------|
|500–1000         |Резерв (Мигает)|Код/Сигнал "Мигание"     |
|-----------------|---------------|-------------------------|
|0–500            |Пустой (Горит) |Выход 0В (или DAC 0)     |


















































