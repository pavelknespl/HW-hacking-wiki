# Finding and Identifying Pins

An unlabeled row of pads is the most common puzzle in hardware hacking. This page shows how to turn a mystery header into a known [UART](../protocols/uart.md) or [JTAG](../protocols/jtag.md) pinout using a multimeter, a [logic analyzer](../software/logic-analyzers.md), and a little reasoning. The same approach identifies individual signal pins on an unmarked chip.

## Step 1: Find ground

Ground is your anchor for everything else.

- Set the multimeter to **continuity**. Ground connects, with a beep and near-zero resistance, to large copper areas, shield cans, and the negative terminal of the power input.
- Test each unknown pad against a known ground. The one that beeps is `GND`.
- Confirm there is often more than one ground pin on a header. Any of them works as your reference.

## Step 2: Find power

- With the device powered, measure each remaining pad to ground with the meter on **DC volts**.
- A pad sitting steadily at 3.3 V (or 1.8 V, or 5 V) is likely `VCC`. On a UART header you usually leave this pin disconnected and power the device normally. Driving it can conflict with the onboard regulator.

## Step 3: Watch the pins during boot

Signal pins reveal themselves by how they behave when the device starts.

Connect a [logic analyzer](../software/logic-analyzers.md) to the unknown pads (sharing ground), power the device, and capture the first few seconds.

- **UART TX** bursts into activity immediately: a steady stream of pulses as the boot log prints. It idles **high** and drops low for start bits. This is the easiest signal to spot.
- **UART RX** is usually quiet at boot, idling high, because nothing is typing yet.
- **Clock lines** ([SPI](../protocols/spi.md) `CLK`, JTAG `TCK`, [I2C](../protocols/i2c.md) `SCL`) show regular, periodic toggling.
- **Floating or unused pins** drift or sit at odd voltages.

## Identifying a UART

UART is the highest-value find, so prioritize it. Signs of a UART header:

- A group of **3 or 4 pins**: `GND`, `TX`, `RX`, and sometimes `VCC`.
- One pin (TX) that erupts in activity the instant the device powers on.
- Voltage levels that swing between 0 V and the logic rail.

To confirm and read it:

1. Identify `GND` (step 1) and the busy pin (`TX`, step 3).
2. Determine the baud rate. If you captured with a logic analyzer, measure the width of the narrowest pulse; its inverse is the baud rate. Common values are 115200, 57600, and 9600. See [UART](../protocols/uart.md) for a baud table and auto-detection tricks.
3. Connect your adapter: adapter `RX` to target `TX`, adapter `TX` to target `RX`, and ground to ground. Open a [serial console](../software/serial-consoles.md).

!!! tip "JTAGulator and UART bruteforcing"
    Tools like the **JTAGulator** automate pin identification by driving and sensing combinations across a header. In software, `jtagenum` running on a microcontroller or [Raspberry Pi](../scripting/gpio-control.md) brute-forces JTAG pinouts. For UART, scripts that cycle through common baud rates while watching for printable ASCII quickly lock onto the right speed.

## Identifying JTAG and SWD

JTAG uses four or five signals: `TCK`, `TMS`, `TDI`, `TDO`, and optional `TRST`. [SWD](../protocols/swd.md) uses two: `SWDIO` and `SWCLK`. Both are harder to identify by eye than UART because they are quiet unless a debugger is talking.

Approaches:

- **Look for a standard header.** Many boards use a 2x5 or 2x10 0.05 inch header, or the 10/20-pin ARM layout. A recognizable connector short-circuits the guesswork.
- **Continuity to the SoC.** JTAG pins trace back to specific balls on the processor. If the datasheet gives those ball locations, continuity testing maps the header.
- **Brute force with JTAGulator or jtagenum.** These drive the IEEE 1149.1 state machine across candidate pins and detect a valid TAP by reading the IDCODE. A returned IDCODE confirms both the pinout and that JTAG is alive.

See [JTAG](../protocols/jtag.md) and [SWD](../protocols/swd.md) for the full procedures.

## Identifying SPI flash pins

A SOIC-8 flash has a fixed, standard pinout, so identification is mostly confirming the chip orientation:

| Pin | Name | Role |
| --- | --- | --- |
| 1 | `CS#` | Chip select, active low |
| 2 | `DO` (MISO) | Data out |
| 3 | `WP#` | Write protect |
| 4 | `GND` | Ground |
| 5 | `DI` (MOSI) | Data in |
| 6 | `CLK` | Clock |
| 7 | `HOLD#` | Hold / reset |
| 8 | `VCC` | Power |

Pin 1 is marked by a dot or a bevel on the chip body. With that corner identified, the rest follows the table. See [SPI Flash Dumping](../attacks/spi-flash-dump.md).

## A reasoned example

Suppose you find an unlabeled 4-pad row near a router's SoC:

1. Continuity test: pad 1 beeps to the shield. Pad 1 is `GND`.
2. Powered voltage check: pad 4 reads a steady 3.3 V. Pad 4 is `VCC`.
3. Logic capture at boot: pad 3 bursts with activity, pad 2 stays idle high. Pad 3 is `TX`, pad 2 is `RX`.
4. Pulse width of the narrowest bit on pad 3 is about 8.68 microseconds, which is 115200 baud.

You now wire adapter `RX` to pad 3, adapter `TX` to pad 2, ground to pad 1, leave pad 4 alone, open a terminal at 115200, and read the boot log. That is the whole method in miniature.


## Related pages

- [Pin Reference](../reference/pin-reference.md)
- [UART](../protocols/uart.md)
- [JTAG](../protocols/jtag.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
- [Pinout Cheat Sheet](../reference/pinouts.md)
- [PCB Recon](pcb-recon.md)
