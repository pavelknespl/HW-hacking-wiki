# ADC, DAC and PWM

Digital pins only know high and low. To measure a voltage (a battery, a potentiometer, a sensor, a supply rail on a target board) you need the **ADC**. To produce something in between, you either use a **DAC**, which few MCUs have, or **PWM**, which every MCU has: a fast square wave whose average voltage is set by its duty cycle.

## ADC: reading a voltage

An ADC converts the voltage on a pin into a number between 0 and its maximum (4095 for a 12-bit ADC). The voltage is `raw / max * Vref`, where `Vref` is usually the 3.3 V supply.

| MCU | Resolution | Range | Notes |
| --- | --- | --- | --- |
| RP2040 / RP2350 | 12-bit | 0 to 3.3 V | GP26 to GP29 (ADC0 to ADC3), ADC4 is the internal temperature sensor |
| ESP32 | 12-bit | 0 to about 3.1 V with 11/12 dB attenuation | Noticeably non-linear; ADC2 pins do not work while Wi-Fi is on |
| ATmega328P | 10-bit | 0 to 5 V (AVCC) | A0 to A5 |
| STM32F4 | 12-bit | 0 to 3.3 V | Fast, good linearity |

=== "MicroPython"

    ```python
    from machine import ADC, Pin

    adc = ADC(Pin(26))                     # Pico ADC0
    raw = adc.read_u16()                   # always scaled to 0..65535 on every port
    volts = raw * 3.3 / 65535
    print(f"{volts:.3f} V")

    # ESP32: set attenuation for the full range, read calibrated microvolts
    # adc = ADC(Pin(34), atten=ADC.ATTN_11DB)
    # print(adc.read_uv() / 1e6, "V")

    # Pico internal temperature sensor
    t = ADC(4).read_u16() * 3.3 / 65535
    print("chip temp:", 27 - (t - 0.706) / 0.001721, "C")
    ```

=== "Arduino"

    ```cpp
    void setup() {
      Serial.begin(115200);
      analogReadResolution(12);            // ESP32, RP2040, SAMD, STM32; AVR is fixed at 10
    }

    void loop() {
      int raw = analogRead(A0);
      float volts = raw * 3.3 / 4095.0;    // Uno: raw * 5.0 / 1023.0
      Serial.println(volts, 3);
      // ESP32 shortcut with factory calibration:
      // Serial.println(analogReadMilliVolts(34));
      delay(200);
    }
    ```

=== "Pico SDK"

    ```c
    #include "hardware/adc.h"

    adc_init();
    adc_gpio_init(26);           // disable digital functions on GP26
    adc_select_input(0);         // ADC0 = GP26
    uint16_t raw = adc_read();   // 12-bit, 0..4095
    float volts = raw * 3.3f / 4095.0f;
    ```

    Link `hardware_adc`. For the temperature sensor call `adc_set_temp_sensor_enabled(true)` and select input 4.

=== "ESP-IDF"

    ```c
    #include "esp_adc/adc_oneshot.h"

    adc_oneshot_unit_handle_t adc;
    adc_oneshot_unit_init_cfg_t unit = { .unit_id = ADC_UNIT_1 };
    adc_oneshot_new_unit(&unit, &adc);

    adc_oneshot_chan_cfg_t ch = {
        .atten = ADC_ATTEN_DB_12,           // full range, about 0..3.1 V
        .bitwidth = ADC_BITWIDTH_DEFAULT,
    };
    adc_oneshot_config_channel(adc, ADC_CHANNEL_6, &ch);   // GPIO34 = ADC1_CH6

    int raw;
    adc_oneshot_read(adc, ADC_CHANNEL_6, &raw);
    ```

    Use the `esp_adc/adc_cali.h` calibration API to convert raw values to millivolts accurately.

=== "STM32 HAL"

    ```c
    // CubeMX: ADC1, channel 0 (PA0), single conversion
    extern ADC_HandleTypeDef hadc1;

    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, 10);
    uint32_t raw = HAL_ADC_GetValue(&hadc1);
    HAL_ADC_Stop(&hadc1);
    float volts = raw * 3.3f / 4095.0f;
    ```

### Measuring more than 3.3 V

Never feed a pin more than its supply voltage. Scale higher voltages down with a resistor divider:

```
  Vin ──[ R1 ]──┬──[ R2 ]── GND
                │
              ADC pin        Vadc = Vin * R2 / (R1 + R2)
```

With R1 = 100 kΩ and R2 = 33 kΩ, 12 V becomes about 2.98 V. Multiply the reading by `(R1 + R2) / R2` to get the real voltage back. High-value resistors keep the load on the measured circuit low; add a 100 nF capacitor from the ADC pin to ground if readings are noisy.

### Getting stable readings

- **Average.** Take 16 or 64 samples and average them. It is cheap and removes most noise.
- **Short, quiet wiring.** Keep analog wires away from PWM and switching lines.
- **Know the reference.** If the 3.3 V supply sags, every reading shifts. Some MCUs have an internal reference you can measure against.
- **ESP32 caveats.** Use calibrated reads (`read_uv()`, `analogReadMilliVolts()`, `adc_cali`) and stay inside roughly 0.15 to 2.5 V for the most linear region.

## PWM: average voltage from a square wave

PWM switches a pin between high and low at a fixed **frequency**. The **duty cycle** is the fraction of time it is high. At 25% duty on a 3.3 V pin, the average is 0.825 V. LEDs and motors respond to the average, so PWM dims and controls speed without wasting power.

| Use | Typical frequency | Duty |
| --- | --- | --- |
| LED dimming | 500 Hz to 20 kHz | 0 to 100% brightness |
| DC motor (via driver) | 10 to 25 kHz (above hearing) | speed |
| Hobby servo | 50 Hz | 1.0 ms to 2.0 ms pulse = 0 to 180 degrees |
| Buzzer tone | the note's frequency | 50% |

=== "MicroPython"

    ```python
    from machine import PWM, Pin

    pwm = PWM(Pin(15))
    pwm.freq(1000)                 # 1 kHz
    pwm.duty_u16(32768)            # 50% (0..65535)

    # servo: 50 Hz, set the pulse width directly
    servo = PWM(Pin(16))
    servo.freq(50)
    servo.duty_ns(1_500_000)       # 1.5 ms = centre position

    pwm.deinit()                   # stop
    ```

=== "Arduino"

    ```cpp
    // AVR, RP2040, STM32: analogWrite on a PWM-capable pin, 8-bit by default
    analogWrite(9, 128);            // 50%

    // ESP32 (core 3.x): LEDC API with explicit frequency and resolution
    ledcAttach(15, 1000, 10);       // pin, 1 kHz, 10-bit
    ledcWrite(15, 512);             // 50%
    ```

    For servos, the `Servo` library (`ESP32Servo` on ESP32) handles the 50 Hz timing: `servo.attach(16); servo.write(90);`.

=== "Pico SDK"

    ```c
    #include "hardware/pwm.h"

    gpio_set_function(15, GPIO_FUNC_PWM);
    uint slice = pwm_gpio_to_slice_num(15);
    pwm_set_clkdiv(slice, 125.0f);      // 125 MHz / 125 = 1 MHz counter (RP2040)
    pwm_set_wrap(slice, 999);           // 1 MHz / 1000 = 1 kHz
    pwm_set_gpio_level(15, 500);        // 50%
    pwm_set_enabled(slice, true);
    ```

    The output frequency is `clk_sys / clkdiv / (wrap + 1)`. The RP2350 runs at 150 MHz by default, so use a divider of 150 there. Link `hardware_pwm`.

=== "ESP-IDF"

    ```c
    #include "driver/ledc.h"

    ledc_timer_config_t t = {
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .timer_num = LEDC_TIMER_0,
        .duty_resolution = LEDC_TIMER_10_BIT,
        .freq_hz = 1000,
        .clk_cfg = LEDC_AUTO_CLK,
    };
    ledc_timer_config(&t);

    ledc_channel_config_t c = {
        .gpio_num = 15,
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .channel = LEDC_CHANNEL_0,
        .timer_sel = LEDC_TIMER_0,
        .duty = 512,                    // 50% of 1024
    };
    ledc_channel_config(&c);

    // change duty later
    ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 256);
    ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
    ```

=== "STM32 HAL"

    ```c
    // CubeMX: TIM2 channel 1 as "PWM Generation CH1" on PA0.
    // With an 84 MHz timer clock: Prescaler 83 -> 1 MHz, Period 999 -> 1 kHz.
    extern TIM_HandleTypeDef htim2;

    HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);
    __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, 500);   // 50%
    ```

!!! warning "Motors and relays need a driver"
    A GPIO pin can supply a few milliamps. Drive motors, solenoids, and relays through a MOSFET or a driver IC (DRV8833, L298N, TB6612), and put a flyback diode across inductive loads.

## DAC: a real analog output

A DAC outputs a true constant voltage instead of a square wave.

=== "MicroPython"

    ```python
    from machine import DAC, Pin
    dac = DAC(Pin(25))          # classic ESP32: GPIO25 or GPIO26 (ESP32-S2: 17, 18)
    dac.write(128)              # 8-bit: about 1.65 V
    ```

=== "Arduino"

    ```cpp
    dacWrite(25, 128);          // classic ESP32 / S2
    ```

=== "STM32 HAL"

    ```c
    // CubeMX: DAC channel 1 (PA4) on parts that have one (F4, G4, L4...)
    extern DAC_HandleTypeDef hdac;
    HAL_DAC_Start(&hdac, DAC_CHANNEL_1);
    HAL_DAC_SetValue(&hdac, DAC_CHANNEL_1, DAC_ALIGN_12B_R, 2048);   // about 1.65 V
    ```

The RP2040, ESP32-S3, ESP32-C3, and ATmega328P have no DAC. Use PWM followed by an RC low-pass filter instead. With a 10 kΩ resistor and a 1 µF capacitor (cutoff about 16 Hz) and a PWM frequency of 20 kHz or more, the output is a fairly smooth DC level. It cannot supply current, so buffer it with an op-amp if it drives a load.
