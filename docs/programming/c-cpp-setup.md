# Getting Started with C and C++

C is the native language of microcontrollers. Every vendor ships a C SDK, every datasheet describes registers in C terms, and anything timing-critical ends up in C. The trade-off is a compile and flash step for every change.

There are several layers to choose from. They all compile to the same kind of ARM, Xtensa, RISC-V, or AVR machine code; they differ in how much they hide.

| Layer | Example | Hides | Use when |
| --- | --- | --- | --- |
| Arduino | `Wire.begin()` | Almost everything | Quick tools, existing libraries, any board |
| Vendor SDK | Pico SDK, ESP-IDF | Register details, not the hardware model | Serious projects on one chip family |
| Vendor HAL | STM32 HAL + CubeMX | Init code is generated for you | STM32 work |
| Bare metal | `GPIOC->ODR ^= 1 << 13` | Nothing | Learning, tiny parts, exact control |

## Arduino (and PlatformIO)

The Arduino framework runs on far more than Arduino boards: ESP32, RP2040, STM32, nRF52, and SAMD all have Arduino cores. Its API (`pinMode`, `digitalWrite`, `Wire`, `SPI`, `Serial`) is the lowest-effort way to write C++ for a new chip.

A sketch has two functions. The framework calls `setup()` once, then `loop()` forever:

```cpp
void setup() {
  pinMode(LED_BUILTIN, OUTPUT);
  Serial.begin(115200);
}

void loop() {
  digitalWrite(LED_BUILTIN, !digitalRead(LED_BUILTIN));
  Serial.println("tick");
  delay(500);
}
```

=== "arduino-cli"

    ```bash
    arduino-cli config init
    # add third-party cores (ESP32, RP2040)
    arduino-cli config add board_manager.additional_urls \
      https://espressif.github.io/arduino-esp32/package_esp32_index.json \
      https://github.com/earlephilhower/arduino-pico/releases/download/global/package_rp2040_index.json
    arduino-cli core update-index
    arduino-cli core install arduino:avr esp32:esp32 rp2040:rp2040

    arduino-cli board list                       # find the port and FQBN
    arduino-cli compile --fqbn arduino:avr:uno blink
    arduino-cli upload  --fqbn arduino:avr:uno -p /dev/ttyACM0 blink
    arduino-cli monitor -p /dev/ttyACM0 -c baudrate=115200
    ```

    Common FQBNs: `arduino:avr:uno`, `esp32:esp32:esp32`, `rp2040:rp2040:rpipico`.

=== "PlatformIO"

    PlatformIO is a command-line build system (and VS Code extension) that downloads toolchains per project. Everything lives in `platformio.ini`:

    ```ini
    [env:pico]
    platform = raspberrypi
    board = pico
    framework = arduino

    [env:esp32]
    platform = espressif32
    board = esp32dev
    framework = arduino        ; or: espidf
    monitor_speed = 115200

    [env:nucleo]
    platform = ststm32
    board = nucleo_f401re
    framework = stm32cube       ; HAL, or: arduino
    ```

    Code goes in `src/main.cpp` (Arduino projects need `#include <Arduino.h>` at the top).

    ```bash
    pip install platformio
    pio run                       # build all envs
    pio run -e esp32 -t upload    # build + flash one
    pio device monitor
    ```

=== "Arduino IDE"

    Install the IDE, add the board package URLs from the arduino-cli tab under *File, Preferences, Additional boards manager URLs*, install the core from *Boards Manager*, pick the board and port, and press Upload. Fine for beginners, but the CLI options are easier to script and version-control.

!!! note "RP2040 Arduino core"
    There are two Arduino cores for the Pico. The examples here use **earlephilhower/arduino-pico** (`rp2040:rp2040`), which is more complete than the official Mbed-based one.

## Raspberry Pi Pico SDK

The Pico SDK is a C library plus a CMake build. It is thin, well documented, and gives full access to PIO, DMA, and both cores.

```bash
# toolchain (Arch / CachyOS; Debian: gcc-arm-none-eabi libnewlib-arm-none-eabi)
sudo pacman -S cmake arm-none-eabi-gcc arm-none-eabi-newlib
git clone --depth 1 https://github.com/raspberrypi/pico-sdk.git ~/pico-sdk
cd ~/pico-sdk && git submodule update --init
export PICO_SDK_PATH=~/pico-sdk
cp ~/pico-sdk/external/pico_sdk_import.cmake ./your-project/
```

A minimal project has three files:

```cmake title="CMakeLists.txt"
cmake_minimum_required(VERSION 3.13)
include(pico_sdk_import.cmake)
project(blink C CXX ASM)
pico_sdk_init()

add_executable(blink main.c)
target_link_libraries(blink pico_stdlib)   # add hardware_i2c, hardware_spi, ... as needed
pico_enable_stdio_usb(blink 1)             # printf over USB serial
pico_enable_stdio_uart(blink 0)
pico_add_extra_outputs(blink)              # produces blink.uf2
```

```c title="main.c"
#include <stdio.h>
#include "pico/stdlib.h"

int main(void) {
    stdio_init_all();
    const uint LED = 25;              // PICO_DEFAULT_LED_PIN on a plain Pico
    gpio_init(LED);
    gpio_set_dir(LED, GPIO_OUT);
    while (true) {
        gpio_xor_mask(1u << LED);
        printf("tick\n");
        sleep_ms(500);
    }
}
```

```bash
mkdir build && cd build
cmake -DPICO_BOARD=pico ..        # pico, pico_w, pico2, pico2_w
make -j$(nproc)
# hold BOOTSEL, plug in, then:
cp blink.uf2 /run/media/$USER/RPI-RP2/
# or, without touching the button once picotool is installed:
picotool load -x blink.uf2 -f
```

On the Pico W the LED hangs off the Wi-Fi chip, so use `pico_cyw43_arch` and `cyw43_arch_gpio_put(CYW43_WL_GPIO_LED_PIN, 1)` instead.

## ESP-IDF

ESP-IDF is Espressif's official framework: FreeRTOS underneath, a component system, and `idf.py` on top.

```bash
git clone --recursive https://github.com/espressif/esp-idf.git ~/esp-idf
cd ~/esp-idf && ./install.sh esp32,esp32s3
. ~/esp-idf/export.sh                # once per shell

idf.py create-project blink && cd blink
idf.py set-target esp32
idf.py menuconfig                    # optional: flash size, log level, Wi-Fi...
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor # Ctrl+] exits the monitor
```

```c title="main/blink.c"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"

void app_main(void) {
    gpio_reset_pin(GPIO_NUM_2);
    gpio_set_direction(GPIO_NUM_2, GPIO_MODE_OUTPUT);
    int level = 0;
    while (1) {
        gpio_set_level(GPIO_NUM_2, level ^= 1);
        ESP_LOGI("blink", "level=%d", level);
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}
```

`app_main` runs as a FreeRTOS task. Always delay with `vTaskDelay` inside loops, otherwise the idle task starves and the task watchdog resets the chip.

## STM32 HAL and CubeMX

ST's workflow generates initialisation code from a graphical pin and clock configurator (STM32CubeMX, also built into STM32CubeIDE). You configure pins and peripherals, it writes `MX_GPIO_Init()`, `MX_I2C1_Init()` and friends, and you add your code between the `USER CODE BEGIN` / `USER CODE END` comments so regeneration keeps it.

```c title="Core/Src/main.c (user section)"
/* USER CODE BEGIN WHILE */
while (1) {
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);   // LD2 on Nucleo boards
    HAL_Delay(500);
    /* USER CODE END WHILE */
}
```

Flashing goes through the ST-LINK on the Nucleo, or any SWD probe:

```bash
st-flash --reset write build/app.bin 0x08000000
# or with OpenOCD:
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg \
        -c "program build/app.elf verify reset exit"
```

The same [SWD](../protocols/swd.md) connection and [OpenOCD](../software/openocd.md) setup is what you use to *attack* STM32 parts, see [STM32 RDP](../mcu/stm32-rdp.md).

## Bare metal: registers directly

Every layer above eventually writes to memory-mapped registers. Doing it yourself once makes datasheets readable and is the only option on very small parts.

=== "AVR (ATmega328P, Arduino Uno)"

    ```c title="main.c"
    #include <avr/io.h>
    #include <util/delay.h>

    int main(void) {
        DDRB |= _BV(PB5);              // PB5 (Arduino pin 13) as output
        for (;;) {
            PORTB ^= _BV(PB5);         // toggle
            _delay_ms(500);
        }
    }
    ```

    ```bash
    avr-gcc -mmcu=atmega328p -DF_CPU=16000000UL -Os -o main.elf main.c
    avr-objcopy -O ihex main.elf main.hex
    avrdude -c arduino -p m328p -P /dev/ttyACM0 -b 115200 -U flash:w:main.hex
    ```

=== "STM32F103 (Blue Pill)"

    ```c title="main.c"
    #include "stm32f1xx.h"            // CMSIS device header

    int main(void) {
        RCC->APB2ENR |= RCC_APB2ENR_IOPCEN;   // clock the GPIOC block
        GPIOC->CRH &= ~(0xF << 20);           // PC13 config bits are [23:20]
        GPIOC->CRH |=  (0x2 << 20);           // output, 2 MHz, push-pull
        for (;;) {
            GPIOC->ODR ^= (1 << 13);
            for (volatile int i = 0; i < 200000; i++) { }
        }
    }
    ```

    You also need a startup file and linker script (take them from ST's CMSIS package or `libopencm3`). Build with `arm-none-eabi-gcc -mcpu=cortex-m3 -mthumb`.

The pattern is always the same: **enable the peripheral clock, configure the pin mode, then read or write the data register.** Forgetting the clock enable is the most common reason bare-metal code does nothing.

## Seeing output

`printf` debugging works everywhere once stdout is routed somewhere:

| Framework | Where `printf` / log output goes |
| --- | --- |
| Arduino | `Serial.print()` to USB serial |
| Pico SDK | USB CDC (`pico_enable_stdio_usb`) or UART0 |
| ESP-IDF | UART0 via the USB bridge, `idf.py monitor` |
| STM32 HAL | Retarget `_write()` to `HAL_UART_Transmit`, or use [SWO](../protocols/trace.md) |

For real debugging (breakpoints, stepping, memory view), attach an SWD probe and GDB through [OpenOCD](../software/openocd.md). A second Pico flashed with the *debugprobe* firmware is a cheap CMSIS-DAP probe.
