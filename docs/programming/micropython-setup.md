# Getting Started with MicroPython

MicroPython is a lean Python 3 implementation that runs directly on the microcontroller. There is no operating system and no compile step: you type a line into the REPL over USB and the pin changes state immediately. For hardware hacking this is ideal, because most of the work is exploratory. You want to scan a bus, poke a register, and see what comes back, not wait for a build.

## Flashing the firmware

MicroPython is a firmware image you write once. Download the right build from [micropython.org/download](https://micropython.org/download/) for your exact board.

=== "Raspberry Pi Pico / Pico 2"

    The RP2040 and RP2350 have a USB mass-storage bootloader in ROM, so no tools are needed.

    1. Hold the **BOOTSEL** button and plug in USB. A drive called `RPI-RP2` (or `RP2350`) appears.
    2. Copy the `.uf2` file onto it.
    3. The board reboots into MicroPython and shows up as `/dev/ttyACM0` (Linux) or a COM port (Windows).

    ```bash
    cp RPI_PICO-*.uf2 /run/media/$USER/RPI-RP2/
    ```

    Use the `RPI_PICO_W` build for the Pico W, otherwise Wi-Fi and the onboard LED will not work.

=== "ESP32 family"

    ESP chips are flashed with [esptool](../software/esptool.md) through their serial bootloader.

    ```bash
    pip install esptool

    # wipe old firmware and filesystem first
    esptool.py --chip esp32 --port /dev/ttyUSB0 erase_flash

    # classic ESP32: image goes at 0x1000
    esptool.py --chip esp32 --port /dev/ttyUSB0 --baud 460800 write_flash -z 0x1000 ESP32_GENERIC-*.bin

    # ESP32-S3 / C3 / C6: image goes at 0x0
    esptool.py --chip esp32s3 --port /dev/ttyACM0 write_flash -z 0x0 ESP32_GENERIC_S3-*.bin
    ```

    If the chip does not answer, hold **BOOT** (GPIO0 low) while pressing **EN**/reset to force the download mode. esptool 5 also accepts `esptool` and `write-flash`; the old spellings still work.

=== "STM32 (Pyboard, Nucleo)"

    Boards with a built-in USB DFU bootloader take a `.dfu` file:

    ```bash
    # put the board in DFU mode (BOOT0 high, reset), then:
    dfu-util -a 0 -D NUCLEO_F401RE-*.dfu
    ```

    Nucleo boards can also take the `.hex`/`.bin` over the onboard ST-LINK with `st-flash` or by dragging it onto the ST-LINK mass-storage drive.

## Talking to the board

The board exposes a USB serial port with the **REPL** (read, evaluate, print loop) on it. `mpremote` is the official all-in-one tool:

```bash
pip install mpremote

mpremote                       # auto-connect to the first board, open REPL
mpremote connect /dev/ttyACM0  # pick a specific port
```

Any [serial console](../software/serial-consoles.md) works too (`tio /dev/ttyACM0`, `picocom -b 115200 /dev/ttyACM0`). Useful REPL keys:

| Keys | Action |
| --- | --- |
| ++ctrl+c++ | Interrupt the running program |
| ++ctrl+d++ | Soft reset (re-runs `boot.py` and `main.py`) |
| ++ctrl+e++ | Paste mode: paste a block of code, then ++ctrl+d++ to run it |
| ++ctrl+x++ | Exit `mpremote` |
| ++tab++ | Auto-complete, try `machine.` then ++tab++ |

First test, blink the onboard LED:

```python
>>> from machine import Pin
>>> led = Pin("LED", Pin.OUT)   # Pico / Pico W; use Pin(2, Pin.OUT) on ESP32
>>> led.toggle()
```

## The `mpremote` workflow

```bash
mpremote run probe.py              # run a local file on the board, without saving it
mpremote fs ls                     # list files on the board
mpremote fs cp main.py :main.py    # upload (":" means "on the board")
mpremote fs cp :log.txt .          # download
mpremote fs rm :old.py
mpremote reset                     # hard reset
mpremote mip install ssd1306       # install a package from micropython-lib
mpremote exec "import machine; print(machine.freq())"
```

Commands chain, which makes a fast edit, run loop:

```bash
mpremote fs cp main.py :main.py + reset + repl
```

`mpremote mount .` is the most useful trick of all: it mounts your **local directory** on the board as its working directory. Edit files on your PC, press ++ctrl+d++ in the REPL, and the board runs the new code without copying anything.

## Program structure

On boot, MicroPython runs two files from its internal filesystem, in order:

| File | Purpose |
| --- | --- |
| `boot.py` | Low-level setup: clock speed, Wi-Fi, USB mode. Keep it tiny. |
| `main.py` | Your application. Runs after `boot.py`. |

A typical `main.py` skeleton:

```python
from machine import Pin
import time

led = Pin("LED", Pin.OUT)

def setup():
    print("starting")

def loop():
    led.toggle()
    time.sleep_ms(500)

setup()
while True:
    loop()
```

!!! tip "Escape from a broken `main.py`"
    If `main.py` crashes the board or grabs the USB port, connect with `mpremote` and press ++ctrl+c++ quickly to stop it, then `mpremote fs rm :main.py`. As a last resort, reflash after an `erase_flash` (ESP32) or with the `flash_nuke.uf2` image (Pico).

## Things that differ from desktop Python

- **Modules are smaller.** `time.sleep_ms()`, `time.ticks_ms()` and `time.ticks_diff()` replace `time.time()` math. Use `ticks_diff` because tick counters wrap.
- **The `machine` module is the hardware.** `Pin`, `I2C`, `SPI`, `UART`, `ADC`, `PWM`, `Timer`, `WDT` all live there, with a common API across ports.
- **`bytes` and `bytearray` are your friends.** Bus calls take and return them. Preallocate a `bytearray` and use the `*_into()` variants in loops to avoid allocations.
- **Memory is tight.** Check with `import gc; gc.mem_free()` and call `gc.collect()` before big allocations.
- **`micropython.const()`** turns a constant into a compile-time value, saving RAM for register maps.
- **Port-specific modules** exist: `rp2` (PIO), `esp32` (NVS, partitions), `pyb` (Pyboard).

## Making it fast

When pure Python is too slow, MicroPython has two built-in code emitters:

```python
import micropython

@micropython.native     # roughly 2x faster, same Python semantics
def checksum(buf):
    s = 0
    for b in buf:
        s += b
    return s & 0xFF

@micropython.viper      # much faster, typed, close to C, port dependent
def xor_buf(buf: ptr8, n: int, key: int):
    for i in range(n):
        buf[i] ^= key
```

If that is still not enough, move the timing-critical part into the RP2040's [PIO](bit-banging-pio.md) blocks or switch to [C](c-cpp-setup.md).

## Where next

Continue with [GPIO](gpio.md), then the bus pages: [I2C](i2c.md) and [UART](uart.md).
