# Glasgow Interface Explorer

Glasgow is an open-source hardware tool built for exactly the situation a hardware hacker is usually in: an unknown board, unlabeled pins, and no idea yet what speaks what. It is an FPGA-based multi-tool whose pins are software-defined, so one board can be a [UART](../protocols/uart.md), a [SPI](../protocols/spi.md)/[I2C](../protocols/i2c.md) master, a [JTAG](../protocols/jtag.md) adapter, a memory reader, or a logic analyzer, with safe and configurable I/O voltages per bank. Pair it with [Finding and Identifying Pins](../fundamentals/identifying-pins.md).

## Why it is useful

- **Per-pin, per-bank voltage.** Each I/O bank sets its own voltage and can sense the target's level, which avoids the [level-shifting](../fundamentals/level-shifting.md) guesswork that kills 1.8 V targets.
- **Applets, not firmware flashing.** Functionality comes from "applets" that reconfigure the FPGA on demand. You switch from UART sniffing to SPI flash reading with a command, not a reflash.
- **Built for exploration.** It is designed to poke at hardware you do not understand yet, which is most real targets.

## Typical uses

```bash
# scan for a UART baud rate on a suspected console
glasgow run uart-analyzer -V 3.3 ...

# read a SPI flash chip
glasgow run memory-25x -V 3.3 read 0 0x100000 flash.bin

# talk I2C and probe addresses
glasgow run i2c-initiator -V 3.3 ...
```

Applets cover UART, SPI (including [SPI flash](../protocols/spi.md) with the `memory-25x` applet), I2C, JTAG, SWD, and more; the set grows with the project. The `memory-25x` applet makes Glasgow a capable [SPI flash dumper](../attacks/spi-flash-dump.md), and the JTAG applets help when probing an [FPGA](../fpga/index.md) or SoC TAP.

## Where it fits

Glasgow shines in the recon phase: you have a header, you do not know the protocol or voltage, and you want to try several safely. For dedicated fast JTAG/SPI work a plain [FTDI adapter like Tigard](tigard.md) or a [logic analyzer](logic-analyzers.md) may be simpler, and for pure pinout brute force a [JTAGulator](jtagulator.md) is purpose-built. Glasgow is the versatile first thing to reach for when the board is a mystery.

## Related pages

- [Finding and Identifying Pins](../fundamentals/identifying-pins.md)
- [Logic Levels and Level Shifting](../fundamentals/level-shifting.md)
- [Tigard](tigard.md)
- [Logic Analyzers (sigrok)](logic-analyzers.md)
- [SPI Flash Dumping](../attacks/spi-flash-dump.md)
- [UART](../protocols/uart.md)
- [JTAG](../protocols/jtag.md)
