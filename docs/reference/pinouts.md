# Pinout Cheat Sheet

The standard pinouts you will reach for repeatedly. Always confirm against the specific datasheet, but these cover the common cases. For how to *find* and *identify* pins on an unknown board, see [Finding and Identifying Pins](../fundamentals/identifying-pins.md).

## SPI NOR flash (SOIC-8 / SOP-8)

The near-universal serial flash pinout. Pin 1 is marked by a dot or bevel.

```
        +--\/--+
  CS# 1 |      | 8 VCC
   DO 2 |      | 7 HOLD#
  WP# 3 |      | 6 CLK
  GND 4 |      | 5 DI
        +------+
```

| Pin | Name | Alias | Role |
| --- | --- | --- | --- |
| 1 | `CS#` | `CE#` | Chip select, active low |
| 2 | `DO` | `MISO`, `SO` | Data out (chip to master) |
| 3 | `WP#` | `IO2` | Write protect / quad IO2 |
| 4 | `GND` | | Ground |
| 5 | `DI` | `MOSI`, `SI` | Data in (master to chip) |
| 6 | `CLK` | `SCK` | Clock |
| 7 | `HOLD#` | `RESET#`, `IO3` | Hold / reset / quad IO3 |
| 8 | `VCC` | | Power (usually 3.3 V, some 1.8 V) |

See [SPI](../protocols/spi.md) and [SPI Flash Dumping](../attacks/spi-flash-dump.md).

## I2C EEPROM (24Cxx, SOIC-8)

```
         +--\/--+
   A0  1 |      | 8 VCC
   A1  2 |      | 7 WP
   A2  3 |      | 6 SCL
  GND  4 |      | 5 SDA
         +------+
```

| Pin | Name | Role |
| --- | --- | --- |
| 1 to 3 | `A0`,`A1`,`A2` | Address select (set the device address 0x50 to 0x57) |
| 4 | `GND` | Ground |
| 5 | `SDA` | Serial data |
| 6 | `SCL` | Serial clock |
| 7 | `WP` | Write protect |
| 8 | `VCC` | Power |

See [I2C](../protocols/i2c.md).

## ARM 20-pin JTAG header

The legacy 0.1 inch (2.54 mm) 2x10 connector. Looking at the header, pin 1 is marked.

| Pin | Signal | Pin | Signal |
| --- | --- | --- | --- |
| 1 | VTref | 2 | VCC |
| 3 | nTRST | 4 | GND |
| 5 | TDI | 6 | GND |
| 7 | TMS / SWDIO | 8 | GND |
| 9 | TCK / SWCLK | 10 | GND |
| 11 | RTCK | 12 | GND |
| 13 | TDO | 14 | GND |
| 15 | nRESET | 16 | GND |
| 17 | DBGRQ | 18 | GND |
| 19 | VCC | 20 | GND |

Note how `TMS`/`SWDIO` (pin 7) and `TCK`/`SWCLK` (pin 9) are shared between JTAG and SWD. See [JTAG](../protocols/jtag.md) and [SWD](../protocols/swd.md).

## ARM Cortex Debug 10-pin header

The modern 0.05 inch (1.27 mm) 2x5 connector, SWD-oriented.

| Pin | Signal | Pin | Signal |
| --- | --- | --- | --- |
| 1 | VTref | 2 | SWDIO / TMS |
| 3 | GND | 4 | SWCLK / TCK |
| 5 | GND | 6 | SWO / TDO |
| 7 | key (no pin) | 8 | TDI |
| 9 | GNDDetect | 10 | nRESET |

## Raspberry Pi 40-pin header (relevant pins)

BCM numbering, as used by [GPIO scripts](../scripting/gpio-control.md).

| Function | BCM | Physical pin |
| --- | --- | --- |
| 3.3 V power | | 1, 17 |
| 5 V power | | 2, 4 |
| Ground | | 6, 9, 14, 20, 25, 30, 34, 39 |
| UART TXD | GPIO14 | 8 |
| UART RXD | GPIO15 | 10 |
| I2C1 SDA | GPIO2 | 3 |
| I2C1 SCL | GPIO3 | 5 |
| SPI0 MOSI | GPIO10 | 19 |
| SPI0 MISO | GPIO9 | 21 |
| SPI0 SCLK | GPIO11 | 23 |
| SPI0 CE0 | GPIO8 | 24 |

!!! warning
    Pi GPIO is 3.3 V and not 5 V tolerant. See [Level Shifting](../fundamentals/level-shifting.md).

## USB-to-UART adapter (typical)

Common FTDI/CP210x/CH340 breakout pins. Connect crossing TX and RX to the target.

| Adapter pin | Connect to target |
| --- | --- |
| GND | GND |
| TX / TXD | RX |
| RX / RXD | TX |
| VCC / 3V3 | (usually leave unconnected) |
| CTS / RTS | (usually unused) |

Set the adapter's voltage jumper to match the target (3.3 V default). See [UART](../protocols/uart.md) and [Serial Consoles](../software/serial-consoles.md).

## Common baud rates

Try in this order when a rate is unknown: **115200**, 57600, 38400, 19200, 9600, then 230400, 460800, 1500000. See [UART](../protocols/uart.md#determining-the-baud-rate).

## Common logic voltages

| Voltage | Where |
| --- | --- |
| 5 V | Older logic, Arduino Uno, some sensors |
| 3.3 V | The embedded default: most SoC I/O, SPI flash, UART headers |
| 1.8 V | Modern SoC I/O, high-density flash, eMMC |
| 1.2 V / 1.1 V | SoC core rails, DDR |

See [Logic Levels and Level Shifting](../fundamentals/level-shifting.md).
