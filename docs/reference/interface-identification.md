# Interface Identification

A field guide for turning observations into a guess about what interface you are looking at: a symptom-to-interface map, the tells of each bus, and a UART baud bruteforce table. This condenses the reasoning on [Finding and Identifying Pins](../fundamentals/identifying-pins.md) into a lookup you can scan at the bench.

## Symptom to probable interface

| What you observe | Probable interface | Confirm by |
| --- | --- | --- |
| 3 to 4 pin header near the SoC; one pin bursts with activity at power-on | [UART](../protocols/uart.md) | Idle-high line, readable ASCII at a [standard baud](#uart-baud-bruteforce) |
| Group of pins labeled TCK/TMS/TDI/TDO or a 2x10 / 2x5 header | [JTAG](../protocols/jtag.md) | Valid IDCODE via JTAGulator / OpenOCD scan |
| Two pins labeled SWDIO/SWCLK near a Cortex-M chip | [SWD](../protocols/swd.md) | DAP responds in OpenOCD |
| 8-pin chip (SOIC-8) near the SoC, clock + two data + CS lines | [SPI](../protocols/spi.md) flash | JEDEC ID via flashrom |
| Two lines with pull-up resistors to a small 8-pin chip | [I2C](../protocols/i2c.md) | Devices acknowledge on `i2cdetect` |
| Single data line, pull-up, to a sensor/ID/authentication chip | [1-Wire](../protocols/one-wire.md) | ROM IDs appear on the bus |
| Two lines (MDC/MDIO) between a MAC and an Ethernet PHY | [MDIO](../protocols/mdio.md) | PHY ID registers read back |
| Multiplexed 4-bit bus at 33 MHz near a TPM/super-I/O | [LPC](../protocols/lpc.md) | LFRAME#/LAD decode; TPM frames |
| BGA with CLK/CMD/DAT lines, large storage | [eMMC](../protocols/emmc.md) | Enumerates as a block device |
| TSOP-48 chip, many parallel lines | [Parallel NAND](../attacks/nand-emmc-dump.md) | Datasheet pinout, programmer read |
| USB connector or data pair to a controller | [USB](../protocols/usb.md) | `lsusb` enumerates descriptors |
| Two-wire differential, ~60 ohm across the pair, in a vehicle | [CAN](../automotive/can-bus.md) | Frames on `candump` at 500 kbit/s |

## The tells of each bus

- **UART**: one line (TX) erupts the instant power is applied; it idles high and drops for start bits. The easiest signal to spot with a [logic analyzer](../software/logic-analyzers.md).
- **Clock-bearing buses** (SPI, JTAG, I2C, LPC, MDIO): look for a line that toggles at a steady frequency during activity. That is a clock, and a clock means a synchronous bus.
- **SPI vs I2C**: SPI has four active lines (CLK, two data, CS) and no pull-ups on the data lines; I2C has exactly two lines, both with pull-up resistors, and is much slower.
- **JTAG vs SWD**: four or five signals is JTAG; two signals (plus optional reset) next to an ARM Cortex-M is SWD. They share silicon, and many headers expose both.
- **CAN**: a differential pair reading roughly 60 ohm across it with the device off (two 120 ohm terminators in parallel) is the giveaway.

## UART baud bruteforce

When you have found a [UART](../protocols/uart.md) TX line but do not know its speed, try these rates in order. Garbage characters mean the wrong rate; readable ASCII in the boot log means you found it.

| Order | Baud | Notes |
| --- | --- | --- |
| 1 | 115200 | By far the most common on modern SoCs |
| 2 | 57600 | |
| 3 | 38400 | |
| 4 | 9600 | Common on older/8-bit and some industrial gear |
| 5 | 19200 | |
| 6 | 230400 | Faster SoCs |
| 7 | 460800 | |
| 8 | 1500000 | Some high-speed SoC consoles |

Measure instead of guessing when you can: capture TX on a [logic analyzer](../software/logic-analyzers.md), measure the narrowest pulse (one bit), and the baud rate is its inverse. About 8.68 microseconds is 115200; about 104 microseconds is 9600. See [UART](../protocols/uart.md#determining-the-baud-rate).

### Automating the sweep

Scripts cycle the port through each rate and score the output for printable ASCII, locking onto the rate that yields readable text. The same idea underlies dedicated baudrate-detection tools. A manual loop with a [serial terminal](../software/serial-consoles.md) works just as well: open at each rate, power-cycle the target, and watch for a legible boot log.

## The identification procedure in one line

Find [ground](../fundamentals/identifying-pins.md#step-1-find-ground), find [power](../fundamentals/identifying-pins.md#step-2-find-power), capture at boot to see which lines are active, match the pattern above, then confirm with the bus's own tool. Cheapest and most likely first: check for a UART before anything invasive.

## Related pages

- [Finding and Identifying Pins](../fundamentals/identifying-pins.md)
- [Pinout Cheat Sheet](pinouts.md)
- [Flash Chip Reference](flash-chips.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
- [UART](../protocols/uart.md)
