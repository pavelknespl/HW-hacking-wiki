# I2C

I2C (Inter-Integrated Circuit) is a two-wire bus that lets many low-speed chips share the same pair of lines. On a target board it typically connects EEPROMs, sensors, power management ICs, and small configuration memories. For a hardware hacker, I2C is where small but valuable secrets often sit: MAC addresses, calibration data, serial numbers, and sometimes security flags.

## How it works

Two lines, both open-drain and pulled high by resistors, shared by every device on the bus:

| Signal | Meaning |
| --- | --- |
| [`SDA`](../reference/pin-reference.md#sda) | Serial data |
| [`SCL`](../reference/pin-reference.md#scl) | Serial clock |

Plus a common ground. Each device has a 7-bit (sometimes 10-bit) **address**. A master starts a transaction by pulling SDA low while SCL is high (a start condition), sends the target address and a read/write bit, and the addressed device acknowledges. Data then flows a byte at a time, each acknowledged, until a stop condition.

Key consequences for attackers:

- **Addressable.** You can enumerate the bus by probing every address and noting which ones acknowledge.
- **Open-drain and bidirectional.** [Level shifting](../fundamentals/level-shifting.md) must respect that; use a [MOSFET-based shifter](../fundamentals/level-shifting.md), not a push-pull translator.
- **Slow.** Standard mode is 100 kHz, fast mode 400 kHz. Easy to sniff with a cheap [logic analyzer](../software/logic-analyzers.md).

## Finding the bus

- Look for the characteristic **pair of pull-up resistors** near a small 8-pin chip; the two lines they pull up are often SDA and SCL.
- EEPROMs in the `24Cxx` family are I2C by definition. Their pinout is standard. See [Chip Identification](../fundamentals/chip-identification.md).
- The SoC's I2C pins are named in its datasheet and frequently routed to test points.

## Sniffing traffic

Passive sniffing reveals what moves across the bus at boot, which often includes an EEPROM read of configuration.

1. Connect a [logic analyzer](../software/logic-analyzers.md)'s two channels to `SDA` and `SCL`, plus ground.
2. Capture during power-up.
3. Decode with sigrok/PulseView's I2C decoder to get addresses and byte values. See [Logic Analyzers](../software/logic-analyzers.md).

## Actively reading a chip

To dump an EEPROM yourself, become the bus master with a [multi-protocol adapter](../getting-started/toolkit.md#multi-protocol-adapters) (Bus Pirate, FT2232H with `pyftdi`, or a [Raspberry Pi](../scripting/gpio-control.md)).

On a Raspberry Pi with I2C enabled:

```bash
# scan the bus for devices that acknowledge
i2cdetect -y 1

# read a single byte from device 0x50, register 0x00
i2cget -y 1 0x50 0x00

# dump a 24C256 EEPROM (32 KB) at address 0x50
for reg in $(seq 0 255); do
  i2cget -y 1 0x50 $reg
done
```

For larger EEPROMs a proper tool handles 16-bit addressing. `eeprog`, `i2c-tools`, or a Python script using `smbus2` can read the whole device in one pass. A dedicated programmer with a clip is the most reliable route for a standalone chip.

!!! warning "Contention on a live bus"
    If you master a bus while the SoC is also driving it, the two masters collide. Either hold the SoC in reset, read the chip out of circuit, or sniff passively instead of mastering. On a powered board, passive sniffing is the safe default.

## What you find

- **Configuration EEPROMs** holding device settings, sometimes including an "unlocked" or "debug" flag you can flip.
- **Identity data**: MAC addresses, serial numbers, region codes.
- **Calibration and secrets** that firmware reads at boot.
- Occasionally **credentials or keys** stored lazily in an external EEPROM.

Pair an I2C dump with [firmware analysis](../attacks/firmware-analysis.md): the firmware code that reads the EEPROM tells you what each byte means.


## Related pages

- [Pin Reference](../reference/pin-reference.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
- [Memory Types](../fundamentals/memory-types.md)
- [Logic Levels and Level Shifting](../fundamentals/level-shifting.md)
- [SPI](spi.md)
