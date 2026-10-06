# Flash Chip Reference

A quick reference for identifying SPI flash from its JEDEC ID and markings, and for decoding capacity. When you clip onto an unknown chip, [`flashrom`](../software/flashrom.md) reads this ID for you; this page is for when you want to decode it yourself or recognize a part from its markings. See also [Chip Identification](../fundamentals/chip-identification.md) and [Memory Types](../fundamentals/memory-types.md).

## Reading the JEDEC ID

Every SPI NOR chip answers the **RDID** command (`0x9F`) with three bytes: a manufacturer ID, and two bytes of device ID (typically memory type and capacity).

```
master -> 0x9F
chip   -> [manufacturer] [memory type] [capacity]
```

- [`flashrom`](../software/flashrom.md) prints the decoded result: `Found Winbond flash chip "W25Q128.V" (16384 kB)`.
- From a [Raspberry Pi over SPI](../scripting/gpio-control.md) you can issue `0x9F` and read the three bytes directly.

## Manufacturer IDs (byte 1)

The first byte identifies the maker. The common ones:

| JEDEC ID | Manufacturer | Marking prefix |
| --- | --- | --- |
| `0xEF` | Winbond | `W25` |
| `0xC2` | Macronix | `MX25` |
| `0xC8` | GigaDevice | `GD25` |
| `0x20` | Micron / Numonyx / ST | `N25Q`, `MT25Q`, `M25P` |
| `0x01` | Spansion / Cypress / Infineon | `S25FL` |
| `0x1C` | Eon | `EN25` |
| `0x1F` | Adesto / Atmel / Microchip | `AT25`, `AT45` |
| `0xBF` | SST / Microchip | `SST25` |
| `0x9D` | ISSI | `IS25` |
| `0x85` | Puya | `P25Q` |
| `0x68` | Boya | `BY25` |
| `0x89` | Intel | |

If the first byte is `0x7F`, it is a JEDEC continuation code: the real manufacturer ID is in a later bank, which older or unusual parts use.

## Capacity (last byte)

For most SPI NOR, the third RDID byte encodes capacity as a power of two bytes, so you can read the size straight off it:

| Byte | Capacity | In bits |
| --- | --- | --- |
| `0x13` | 512 KB | 4 Mbit |
| `0x14` | 1 MB | 8 Mbit |
| `0x15` | 2 MB | 16 Mbit |
| `0x16` | 4 MB | 32 Mbit |
| `0x17` | 8 MB | 64 Mbit |
| `0x18` | 16 MB | 128 Mbit |
| `0x19` | 32 MB | 256 Mbit |
| `0x20` | 64 MB | 512 Mbit |

Example: a chip answering `EF 40 18` is a Winbond (`EF`), standard SPI (`40`), 16 MB (`18`) part, that is, a W25Q128.

## Common chips at a glance

| Part | Maker | Capacity | Voltage | Package |
| --- | --- | --- | --- | --- |
| W25Q32 | Winbond | 4 MB | 3.3 V | SOIC-8 |
| W25Q64 | Winbond | 8 MB | 3.3 V | SOIC-8 |
| W25Q128 | Winbond | 16 MB | 3.3 V | SOIC-8 |
| MX25L6406E | Macronix | 8 MB | 3.3 V | SOIC-8 |
| MX25L12835F | Macronix | 16 MB | 3.3 V | SOIC-8 |
| GD25Q64 | GigaDevice | 8 MB | 3.3 V | SOIC-8 |
| N25Q128 | Micron | 16 MB | 3.3 V | SOIC-8 |
| S25FL128S | Cypress | 16 MB | 3.3 V | SOIC-16 |
| W25Q128JW | Winbond | 16 MB | **1.8 V** | SOIC-8 / WSON |
| MX25U series | Macronix | varies | **1.8 V** | SOIC-8 / WSON |

!!! warning "Voltage, always"
    The `W25Q128JV` is a 3.3 V part; the `W25Q128JW` is a **1.8 V** part with a nearly identical marking. Powering a 1.8 V chip at 3.3 V can damage it. Confirm the exact suffix against the datasheet before connecting. See [Level Shifting](../fundamentals/level-shifting.md).

## When the ID is unknown to flashrom

If `flashrom` reads an ID it does not recognize, it refuses to operate for safety. You can look up the raw three bytes in the manufacturer's datasheet (search the ID and "JEDEC"), then force the closest matching chip definition with `-c`, but only once you are sure of the real size, or an over/under-read corrupts your dump. See [flashrom](../software/flashrom.md#force-a-chip-model).

## Related pages

- [Chip Identification](../fundamentals/chip-identification.md)
- [Memory Types](../fundamentals/memory-types.md)
- [SPI Flash Dumping](../attacks/spi-flash-dump.md)
- [flashrom](../software/flashrom.md)
- [Pinout Cheat Sheet](pinouts.md)
