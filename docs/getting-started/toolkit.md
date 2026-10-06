# The Toolkit

A reference to the hardware and adapters that show up again and again, grouped by what they do. You will not need all of it. The notes call out what to buy first and what overlaps with what.

## Interface adapters

These bridge your computer's USB port to the low level buses on the target.

### USB to UART adapters

The most used tool in the kit. Converts a serial console to something a terminal program can read.

- **FTDI FT232R** based cables are reliable and well supported.
- **CP2102** and **CH340** based adapters are cheaper and perfectly usable.
- **Requirement:** selectable 3.3 V / 1.8 V / 5 V logic, or at least 3.3 V. A 5 V only adapter will damage most modern targets.

See [Serial Consoles](../software/serial-consoles.md) for the software side.

### Multi-protocol adapters

One device that speaks several buses, ideal for exploration.

| Device | Speaks | Notes |
| --- | --- | --- |
| **Bus Pirate** | UART, SPI, I2C, 1-Wire, JTAG (slow) | Interactive menu, great for learning. Slower than dedicated tools. |
| **FT2232H / FT232H breakout** | UART, SPI, I2C, JTAG, SWD | Fast, works with OpenOCD, `flashrom`, and `pyftdi`. Excellent value. |
| **Tigard** | UART, SPI, I2C, JTAG, SWD | Purpose built for hardware hacking, with voltage selection done well. |
| **Raspberry Pi** | UART, SPI, I2C, bit-banged anything, JTAG | A general purpose GPIO workhorse. See [GPIO Control](../scripting/gpio-control.md). |

### JTAG and SWD probes

For halting the CPU and debugging. See [JTAG](../protocols/jtag.md) and [SWD](../protocols/swd.md).

- **J-Link** (Segger): the gold standard, broad target support, works with OpenOCD and vendor tools.
- **ST-Link**: cheap, excellent for STM32 and any SWD target.
- **FT2232H adapters**: a capable, inexpensive JTAG/SWD interface through OpenOCD.
- **ESP-Prog**: handy for Espressif targets.

### Flash programmers

For reading and writing memory chips directly. See [SPI Flash Dumping](../attacks/spi-flash-dump.md).

- **CH341A**: the ubiquitous cheap SPI/I2C programmer. Watch its voltage, many clones drive 5 V on the data lines, which is risky for 3.3 V chips. A 3.3 V mod or a known-good clone is advised.
- **FT2232H**: drives `flashrom` cleanly at the correct voltage.
- **Dedicated universal programmers** (for example, a XGecu T48/T56): handle parallel NOR, NAND, and odd packages with adapters.

## Clips, sockets, and probes

- **SOIC-8 / SOIC-16 test clips** grab a flash chip's legs without desoldering it.
- **Pomona 5250/5252** clips are the reliable, more expensive option over cheap clones.
- **Chip sockets and adapters** (SOIC to DIP, TSOP, BGA adapters) let a desoldered chip drop into a programmer.
- **Pogo pin jigs** make repeatable contact with test pads and unpopulated headers.
- **Fine-tip probes and grabbers** attach logic analyzer channels to tiny pads.

## Measurement and observation

| Tool | Purpose |
| --- | --- |
| **Multimeter** | Continuity, voltage, identifying ground and power. |
| **Logic analyzer** | Decode digital buses. An 8 channel clone plus [sigrok](../software/logic-analyzers.md) covers a lot. |
| **Oscilloscope** | Analog view: glitches, noise, power rails, timing of fast signals. |
| **USB microscope** | Read chip markings and inspect solder joints. |

## Soldering and rework

- **Temperature controlled iron** with a fine conical and a small chisel tip.
- **Hot air station** for removing and placing surface mount chips.
- **Flux** (a good no-clean paste or pen) makes every joint easier.
- **Solder wick and a solder sucker** for cleanup.
- **Kapton tape** to protect neighboring parts during hot air work.
- **Thin magnet wire** (for example 0.1 mm) for tapping tiny pads.
- **Helping hands or a PCB holder** and a **magnifier**.

## Jumpers and connectors

- Male-to-male, male-to-female, and female-to-female **jumper wires**.
- A **breadboard** for prototyping level shifters and glue circuits.
- An assortment of **Dupont headers** and a crimper if you make your own leads.

## Specialist gear

- **ChipWhisperer** (Lite, Nano, or Husky): an integrated platform for [fault injection](../attacks/fault-injection.md) and [side-channel](../attacks/side-channel.md) work, with excellent tutorials.
- **Software defined radio** (RTL-SDR to start, HackRF or similar to transmit): see [SDR Basics](../radio/sdr-basics.md).
- **Bench power supply** with current readout and limiting.
- **Logic level shifters** (see [Level Shifting](../fundamentals/level-shifting.md)).

## What to buy first

If you are starting from nothing and want one order that covers the most ground:

1. A 3.3 V capable **USB to UART adapter**.
2. A **multimeter** with continuity.
3. A cheap **8 channel logic analyzer**.
4. A **SOIC-8 clip** and a **CH341A** (or an FT2232H breakout, which replaces several items at once).
5. A basic **soldering iron**, flux, and jumper wires.

That kit finds serial consoles, decodes buses, and dumps the most common flash chips, which is the majority of consumer device work.
