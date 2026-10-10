# MCU Programming

Attacking hardware and building hardware are the same skill seen from two sides. A cheap microcontroller board becomes an [I2C](i2c.md) EEPROM reader, a [UART](uart.md) bridge, or a reset controller with twenty lines of code. This section teaches you to write that code, in **MicroPython** for speed of iteration and in **C/C++** when timing, size, or control matter.

Every peripheral page follows the same shape: what the hardware does, then the **smallest working example** for each framework, side by side in tabs. Pick a tab once and the rest of the site remembers it.

<div class="grid cards" markdown>

-   __Getting Started: MicroPython__

    Flash the firmware, open the REPL, copy files with `mpremote`, and structure `boot.py` and `main.py`.

    [:octicons-arrow-right-24: Read](micropython-setup.md)

-   __Getting Started: C and C++__

    Arduino, PlatformIO, the Pico SDK, ESP-IDF, STM32 HAL, and bare-metal register code. Build and flash each one.

    [:octicons-arrow-right-24: Read](c-cpp-setup.md)

-   __GPIO__

    Outputs, inputs, pull-ups, debouncing, and open-drain pins.

    [:octicons-arrow-right-24: Read](gpio.md)

-   __I2C__

    Init the bus, scan it, read and write registers, dump EEPROMs, and act as an I2C target.

    [:octicons-arrow-right-24: Read](i2c.md)

-   __UART__

    Serial ports on the MCU, line-based protocols, AT commands, and a USB to UART bridge.

    [:octicons-arrow-right-24: Read](uart.md)

-   __ADC, DAC and PWM__

    Read voltages, generate analog levels, and drive PWM for LEDs, servos, and motors.

    [:octicons-arrow-right-24: Read](analog-pwm.md)

-   __Timers, Interrupts and Timing__

    Non-blocking loops, pin interrupts, hardware timers, and measuring pulses.

    [:octicons-arrow-right-24: Read](timers-interrupts.md)

-   __1-Wire and Addressable LEDs__

    DS18B20 temperature sensors and WS2812 (NeoPixel) strips.

    [:octicons-arrow-right-24: Read](one-wire-leds.md)

-   __Bit-Banging and RP2040 PIO__

    Implement a protocol in software, then move it into the RP2040's programmable I/O blocks.

    [:octicons-arrow-right-24: Read](bit-banging-pio.md)

-   __Wi-Fi and Networking__

    Connect an ESP32 or Pico W, make HTTP requests, serve a web page, and use MQTT.

    [:octicons-arrow-right-24: Read](networking.md)

-   __Storage, Sleep and Watchdog__

    Files, non-volatile settings, deep sleep, and recovering from hangs.

    [:octicons-arrow-right-24: Read](storage-power.md)

</div>

## Which language?

| | MicroPython | C/C++ |
| --- | --- | --- |
| Edit, run cycle | Seconds, no compile, live REPL | Compile and flash every change |
| Speed | 10 to 100 times slower than C | Native, cycle-accurate if you need it |
| Timing precision | Milliseconds reliably, microseconds roughly | Down to single clock cycles |
| Memory | Needs about 100 KB RAM, 256 KB+ flash | Runs on a 2 KB AVR |
| Libraries | Small but growing (`mip`) | Huge (Arduino, vendor SDKs) |
| Best for | Probing, prototyping, quick dumpers, scripts | Glitchers, sniffers, anything timing-critical, product code |

A good habit is to **prototype in MicroPython, finish in C** when the prototype is too slow. The bus concepts and register maps are identical; only the syntax changes.

## Recommended boards

| Board | MCU | Why | Languages |
| --- | --- | --- | --- |
| Raspberry Pi Pico / Pico 2 | RP2040 / RP2350 | Cheap, unbrickable (ROM bootloader), [PIO](bit-banging-pio.md) for custom protocols | MicroPython, Pico SDK, Arduino |
| Pico W / Pico 2 W | RP2040 / RP2350 + CYW43439 | Same, with Wi-Fi and BLE | MicroPython, Pico SDK, Arduino |
| ESP32 DevKit | ESP32 / S3 / C3 | Wi-Fi + BLE, lots of RAM, DAC on classic ESP32 | MicroPython, ESP-IDF, Arduino |
| STM32 Nucleo / Blue Pill | STM32F4 / F103 | Industry standard, onboard ST-LINK on Nucleo | STM32 HAL, Arduino, MicroPython (some) |
| Arduino Uno / Nano | ATmega328P | 5 V I/O, huge amount of examples | Arduino, bare-metal AVR |

!!! tip "Start with a Pico"
    If you only buy one board, buy a Raspberry Pi Pico. It cannot be bricked (hold BOOTSEL and it always comes back), it runs both MicroPython and C well, and its PIO blocks can emulate almost any bus. Every example in this section has a Pico variant.

!!! warning "Logic levels"
    The Pico, ESP32, and STM32 are **3.3 V** parts and most of their pins are not 5 V tolerant. The Arduino Uno is 5 V. Mixing them without a [level shifter](../fundamentals/level-shifting.md) damages the 3.3 V side. Always connect grounds first.

## Pin numbers used in the examples

Unless a page says otherwise, the examples use these default pins so you can wire once and run everything:

| Function | Pico (GPx) | ESP32 DevKit (GPIOx) | Arduino Uno | STM32 Nucleo-F401 |
| --- | --- | --- | --- | --- |
| LED | 25 (`"LED"` on Pico W) | 2 | 13 | PA5 |
| I2C SDA / SCL | 4 / 5 (I2C0) | 21 / 22 | A4 / A5 | PB9 / PB8 (I2C1) |
| SPI SCK / MOSI / MISO / CS | 18 / 19 / 16 / 17 (SPI0) | 18 / 23 / 19 / 5 | 13 / 11 / 12 / 10 | PA5 / PA7 / PA6 / PB6 |
| UART TX / RX | 0 / 1 (UART0) | 17 / 16 (UART2) | 1 / 0 (`Serial`) | PA2 / PA3 (USART2, to ST-LINK) |
| ADC input | 26 (ADC0) | 34 | A0 | PA0 |
| PWM output | 15 | 15 | 9 | PA0 (TIM2_CH1) |

The Pico and ESP32 can route most peripherals to many pins; the table lists the conventional choices. On the Nucleo, the user LED shares PA5 with SPI1 SCK, so it flickers during SPI traffic. See [Pinout Cheat Sheet](../reference/pinouts.md) for the full pin maps.
