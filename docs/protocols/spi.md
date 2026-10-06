# SPI

SPI (Serial Peripheral Interface) is the fast, four-wire bus that connects most external flash chips to a processor. Because the firmware on a huge class of devices lives in a SOIC-8 SPI NOR flash, SPI is the single most important bus for [firmware extraction](../attacks/firmware-extraction.md). Clip onto the chip, speak SPI, and read the whole image.

## How it works

SPI is a synchronous, full-duplex, master-driven bus:

| Signal | Also called | Meaning |
| --- | --- | --- |
| `SCLK` | `CLK` | Clock, driven by the master |
| `MOSI` | `DI`, `SDI` | Master out, slave in |
| `MISO` | `DO`, `SDO` | Master in, slave out |
| `CS#` | `SS`, `CE#` | Chip select, active low |

The master drives the clock and selects a slave by pulling its `CS#` low. On each clock edge one bit shifts out on MOSI and one shifts in on MISO simultaneously. There is no addressing on the wire; the chip select picks the device, and commands are part of the data stream.

SPI runs far faster than [I2C](i2c.md) (tens of MHz), so a full dump is quick, but capturing it with a [logic analyzer](../software/logic-analyzers.md) needs a correspondingly higher sample rate.

## The standard flash pinout

Almost all SPI NOR flash uses the same SOIC-8 pinout:

| Pin | Name | Role |
| --- | --- | --- |
| 1 | `CS#` | Chip select |
| 2 | `DO` / MISO | Data out |
| 3 | `WP#` | Write protect |
| 4 | `GND` | Ground |
| 5 | `DI` / MOSI | Data in |
| 6 | `CLK` | Clock |
| 7 | `HOLD#` | Hold / reset |
| 8 | `VCC` | Power |

Pin 1 is marked by a dot or a bevel. See [Chip Identification](../fundamentals/chip-identification.md) and [Finding and Identifying Pins](../fundamentals/identifying-pins.md).

!!! note "Quad SPI"
    Higher-performance parts add `IO2` and `IO3` to transfer four bits per clock (QSPI). The `WP#` and `HOLD#` pins double as `IO2` and `IO3` in quad mode. For dumping you generally use plain single-bit SPI, which every chip supports.

## Reading the chip

The friendliest target in hardware hacking. Clip a [SOIC-8 test clip](../getting-started/toolkit.md) onto the chip and read it with [`flashrom`](../software/flashrom.md) and a supported programmer:

```bash
# CH341A programmer
flashrom -p ch341a_spi -r dump.bin

# FT2232H based adapter
flashrom -p ft2232_spi:type=2232H,port=A -r dump.bin
```

`flashrom` reads the chip's JEDEC ID, identifies the part and capacity, and reads the whole flash to `dump.bin`. Full procedure, including the in-circuit power trick, on [SPI Flash Dumping](../attacks/spi-flash-dump.md).

## Sniffing an SPI bus

Instead of mastering the chip, you can watch the SoC talk to it at boot. This is useful when the chip is hard to clip, or when you want to see exactly what the CPU reads.

1. Connect a [logic analyzer](../software/logic-analyzers.md) to `CLK`, `MOSI`, `MISO`, and `CS#`, plus ground.
2. Use a sample rate at least four times the SPI clock (so 40 MS/s for a 10 MHz bus).
3. Capture during boot and decode with sigrok/PulseView's SPI decoder.

The decoded stream shows read commands and the data returned, effectively a partial firmware read as the device boots.

## In-circuit versus desoldered

- **In-circuit (clip on a powered-down board).** Fast and non-destructive, but the rest of the board shares the bus and can interfere. Often you must hold the SoC in reset, or the clip's power is not enough and you power the board externally. See the [dumping guide](../attacks/spi-flash-dump.md) for the details.
- **Desoldered.** Remove the chip with hot air and read it in a socket or on an adapter. Always works, but invasive. Reserve it for when in-circuit reads are unreliable.

## Writing and modifying

`flashrom -w newimage.bin` writes a modified image back. On devices that load firmware from SPI without verifying it, this is a direct way to run your own code: dump, modify, write back, boot. Where [secure boot](../attacks/bootloader-bypass.md) verifies the image, a straight rewrite fails and you move to [bootloader](../attacks/bootloader-access.md) or [fault injection](../attacks/fault-injection.md) approaches.

## Defender's view

Storing firmware unencrypted in an external [SPI flash](../attacks/spi-flash-dump.md) means anyone with a clip has your code and any secrets in it. Encrypt sensitive data at rest, verify the boot image, and do not trust the contents of an external, clippable chip as if it were tamper-proof.


## Related pages

- [SPI Flash Dumping](../attacks/spi-flash-dump.md)
- [flashrom](../software/flashrom.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
- [Memory Types](../fundamentals/memory-types.md)
