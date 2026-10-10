# GPIO

General-purpose I/O pins are the simplest peripheral and the one every other page builds on. A pin is either an **output** that the MCU drives high or low, or an **input** that reads the voltage someone else applies. Holding a target in reset, sensing a status line, pressing a button, or [bit-banging](bit-banging-pio.md) a protocol all come down to these two operations.

## Output

=== "MicroPython"

    ```python
    from machine import Pin

    led = Pin(25, Pin.OUT)      # Pico; Pin("LED") on Pico W, Pin(2) on ESP32
    led.value(1)                # high
    led.off()                   # low (same as value(0))
    led.toggle()                # rp2, esp32 and stm32 ports

    rst = Pin(15, Pin.OUT, value=1)   # set the level at the same time as the mode
    ```

=== "Arduino"

    ```cpp
    const int LED = LED_BUILTIN;

    void setup() {
      pinMode(LED, OUTPUT);
      digitalWrite(LED, HIGH);
    }

    void loop() {
      digitalWrite(LED, !digitalRead(LED));   // toggle
      delay(500);
    }
    ```

=== "Pico SDK"

    ```c
    #include "pico/stdlib.h"

    gpio_init(25);
    gpio_set_dir(25, GPIO_OUT);
    gpio_put(25, 1);
    gpio_xor_mask(1u << 25);        // toggle
    gpio_put_masked(0xF << 6, 0x5 << 6);   // write GP6..GP9 in one go
    ```

=== "ESP-IDF"

    ```c
    #include "driver/gpio.h"

    gpio_config_t io = {
        .pin_bit_mask = 1ULL << GPIO_NUM_2,
        .mode = GPIO_MODE_OUTPUT,
    };
    gpio_config(&io);
    gpio_set_level(GPIO_NUM_2, 1);
    ```

=== "STM32 HAL"

    ```c
    // pin configured as GPIO_Output in CubeMX, so MX_GPIO_Init() did the setup
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET);
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
    ```

## Input and pull resistors

An input pin with nothing connected **floats**: it picks up noise and reads random values. A pull-up or pull-down resistor gives it a defined idle level. Every MCU has weak internal pulls (roughly 30 to 50 kΩ) you can enable in software.

The usual button wiring is pin to button to ground, with the internal **pull-up** enabled. The pin reads `1` when released and `0` when pressed.

=== "MicroPython"

    ```python
    from machine import Pin

    btn = Pin(14, Pin.IN, Pin.PULL_UP)
    if btn.value() == 0:
        print("pressed")
    ```

=== "Arduino"

    ```cpp
    pinMode(14, INPUT_PULLUP);
    if (digitalRead(14) == LOW) {
      Serial.println("pressed");
    }
    ```

=== "Pico SDK"

    ```c
    gpio_init(14);
    gpio_set_dir(14, GPIO_IN);
    gpio_pull_up(14);
    if (!gpio_get(14)) {
        printf("pressed\n");
    }
    ```

=== "ESP-IDF"

    ```c
    gpio_config_t io = {
        .pin_bit_mask = 1ULL << GPIO_NUM_14,
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
    };
    gpio_config(&io);
    if (gpio_get_level(GPIO_NUM_14) == 0) {
        printf("pressed\n");
    }
    ```

=== "STM32 HAL"

    ```c
    // CubeMX: PC13 as GPIO_Input (Nucleo blue button has an external pull-up)
    if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET) {
        // pressed
    }
    ```

!!! note "ESP32 input-only pins"
    On the classic ESP32, GPIO34 to GPIO39 are input-only and have **no internal pulls**. Add an external resistor if you use them for buttons.

## Debouncing

Mechanical contacts bounce for a few milliseconds, so one press looks like several edges. The simplest fix is to accept a new state only after it has been stable for a while:

=== "MicroPython"

    ```python
    import time
    from machine import Pin

    btn = Pin(14, Pin.IN, Pin.PULL_UP)
    last = btn.value()
    changed_at = time.ticks_ms()
    stable = last

    while True:
        v = btn.value()
        if v != last:
            last = v
            changed_at = time.ticks_ms()
        if v != stable and time.ticks_diff(time.ticks_ms(), changed_at) > 20:
            stable = v
            if stable == 0:
                print("press")
    ```

=== "Arduino"

    ```cpp
    const int BTN = 14;
    int last = HIGH, stable = HIGH;
    unsigned long changedAt = 0;

    void setup() { pinMode(BTN, INPUT_PULLUP); Serial.begin(115200); }

    void loop() {
      int v = digitalRead(BTN);
      if (v != last) { last = v; changedAt = millis(); }
      if (v != stable && millis() - changedAt > 20) {
        stable = v;
        if (stable == LOW) Serial.println("press");
      }
    }
    ```

For edge-triggered handling with interrupts, see [Timers, Interrupts and Timing](timers-interrupts.md).

## Open-drain outputs

A push-pull output drives both high and low. An **open-drain** output only pulls low and lets go otherwise, so an external (or internal) pull-up provides the high level. This is how [I2C](i2c.md), reset lines, and wired-OR interrupt lines work, and it is the safe way to drive a target's reset pin: you can pull it low but you never fight the target's own pull-up with a hard 3.3 V.

=== "MicroPython"

    ```python
    rst = Pin(15, Pin.OPEN_DRAIN, value=1)   # released (pulled up by the target)
    rst.value(0)                             # assert reset
    rst.value(1)                             # release
    ```

=== "Arduino"

    ```cpp
    // portable trick: switch between "output low" and "input" (high-Z)
    void resetAssert()  { pinMode(15, OUTPUT); digitalWrite(15, LOW); }
    void resetRelease() { pinMode(15, INPUT); }
    // ESP32 and some cores also accept pinMode(15, OUTPUT_OPEN_DRAIN)
    ```

=== "Pico SDK"

    ```c
    // emulate open-drain: keep the output value at 0, toggle the direction
    gpio_init(15);
    gpio_put(15, 0);
    gpio_set_dir(15, GPIO_OUT);   // assert (drive low)
    gpio_set_dir(15, GPIO_IN);    // release (high-Z)
    ```

=== "ESP-IDF"

    ```c
    gpio_config_t io = {
        .pin_bit_mask = 1ULL << GPIO_NUM_15,
        .mode = GPIO_MODE_INPUT_OUTPUT_OD,
        .pull_up_en = GPIO_PULLUP_ENABLE,
    };
    gpio_config(&io);
    gpio_set_level(GPIO_NUM_15, 0);   // assert
    gpio_set_level(GPIO_NUM_15, 1);   // release
    ```

=== "STM32 HAL"

    ```c
    // CubeMX: GPIO mode "Output Open Drain"
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);  // assert
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);    // release
    ```

## Electrical limits

- **Current.** A GPIO pin can source or sink a few mA (RP2040 defaults to 4 mA drive, configurable up to 12 mA; ESP32 up to about 40 mA absolute max). Drive LEDs through a resistor and anything bigger through a transistor or MOSFET.
- **Voltage.** Never put more than the supply voltage on a 3.3 V pin. See [Logic Levels and Level Shifting](../fundamentals/level-shifting.md).
- **Boot strapping pins.** Some pins are read at reset to choose boot mode (ESP32 GPIO0, 2, 12, 15; STM32 BOOT0). Pulling them the wrong way stops the board from booting. These same pins are interesting on [targets](../platform/soc-recovery-modes.md), for the same reason.
