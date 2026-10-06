# GPIO Control

Driving a target's pins from code is the foundation of automation. A Raspberry Pi exposes general-purpose I/O plus hardware UART, SPI, and I2C controllers, all at 3.3 V, and all scriptable from Python. This page covers the practical ways to control those pins and when to use each.

!!! warning "Voltage and ground"
    Pi GPIO is 3.3 V and **not** 5 V tolerant. Use a [level shifter](../fundamentals/level-shifting.md) for anything other than 3.3 V targets, and tie grounds together first. See [Lab Setup](../getting-started/lab-setup.md).

## The Pi header, briefly

The 40-pin header carries power (3.3 V and 5 V), many grounds, and GPIO pins, several of which have dedicated hardware functions:

| Function | Pins (BCM numbering) |
| --- | --- |
| UART | GPIO14 (TXD), GPIO15 (RXD) |
| SPI0 | GPIO10 (MOSI), GPIO9 (MISO), GPIO11 (SCLK), GPIO8 (CE0) |
| I2C1 | GPIO2 (SDA), GPIO3 (SCL) |
| General GPIO | most of the rest |

Enable the hardware interfaces with `raspi-config` (or `dtparam` in `/boot/config.txt`) before using SPI, I2C, or the UART.

## Basic GPIO from Python

Modern Raspberry Pi OS uses the `libgpiod` interface via the `gpiozero` or `lgpio` libraries (the old `RPi.GPIO` still works on many setups). `gpiozero` is the simplest:

```python
from gpiozero import LED, Button
from time import sleep

# drive a pin high/low (e.g. hold a target in reset)
reset = LED(17)          # BCM 17
reset.on()               # 3.3 V
sleep(0.1)
reset.off()              # 0 V -- pulse the reset line

# read a pin (e.g. sense a target status line)
status = Button(27, pull_up=False)
print("high" if status.is_pressed else "low")
```

Driving a pin to hold a target in reset is a constant need: it is how you [isolate a chip](../attacks/spi-flash-dump.md#the-in-circuit-power-problem) while reading flash, and how you reset a target between [glitch attempts](automation.md).

## UART from the Pi

Use the hardware UART to be a [serial console](../software/serial-consoles.md) or to script interaction:

```python
import serial
ser = serial.Serial("/dev/serial0", 115200, timeout=1)
ser.write(b"\n")                    # nudge the console
print(ser.read(200))                # read whatever comes back
```

Wire Pi TXD to target RXD, Pi RXD to target TXD, and ground to ground, exactly as any [UART](../protocols/uart.md) connection. Scripting the UART lets you, for example, hammer the "any key" window to [interrupt U-Boot](../attacks/bootloader-access.md) reliably.

## SPI from the Pi

The Pi's SPI controller reads flash and talks to SPI peripherals. `flashrom` uses it directly:

```bash
flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=10000 -r dump.bin
```

For custom SPI transactions, `spidev` from Python:

```python
import spidev
spi = spidev.SpiDev()
spi.open(0, 0)                      # bus 0, device CE0
spi.max_speed_hz = 1_000_000
# send a SPI flash JEDEC ID command (0x9F) and read 3 response bytes
resp = spi.xfer2([0x9F, 0, 0, 0])
print([hex(b) for b in resp[1:]])   # manufacturer + device id
```

## I2C from the Pi

Master an I2C bus to read an [EEPROM](../protocols/i2c.md) or sensor:

```bash
i2cdetect -y 1                      # scan for devices that acknowledge
i2cget -y 1 0x50 0x00               # read one byte
```

```python
from smbus2 import SMBus
with SMBus(1) as bus:
    # read 256 bytes from a 24C02-style EEPROM at 0x50
    data = bytes(bus.read_byte_data(0x50, reg) for reg in range(256))
```

## Bit-banging arbitrary protocols

When no hardware controller fits (an odd clock polarity, a non-standard bus, or a [JTAG](../protocols/jtag.md) brute-force), you drive the pins manually in software. This is **bit-banging**: set clock and data pins by hand in the right sequence.

```python
from gpiozero import OutputDevice, InputDevice
clk = OutputDevice(11)
mosi = OutputDevice(10)
miso = InputDevice(9)

def shift_byte(out_byte):
    in_byte = 0
    for i in range(7, -1, -1):          # MSB first
        mosi.value = (out_byte >> i) & 1
        clk.on()                        # rising edge: slave samples
        in_byte = (in_byte << 1) | miso.value
        clk.off()
    return in_byte
```

Bit-banging is slow and timing-approximate on a non-real-time OS, but it is universal: anything with a defined sequence of edges can be driven this way. For JTAG, projects like `jtagenum` bit-bang candidate pins to [find the pinout](../fundamentals/identifying-pins.md#identifying-jtag-and-swd).

## pigpio for precise timing

When bit-banging needs better timing (tighter pulses, hardware-timed waveforms), the `pigpio` daemon provides microsecond-level control and hardware PWM, which plain `gpiozero` cannot guarantee under Linux scheduling. It is the tool for generating a precise pulse, for example a [glitch trigger](../attacks/fault-injection.md) or a clock, from a Pi.

## From control to automation

Once you can drive reset, read a status line, and talk a bus from code, you can wrap the whole attack in a loop: reset the target, perform an action, observe the result, repeat. That is the subject of [Automating Attacks](automation.md).
