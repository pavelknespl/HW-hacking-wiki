# Logic Levels and Level Shifting

Digital signals are only meaningful relative to a voltage. A device that calls 3.3 V a logical one will misread a 5 V signal as noise at best and burn out an input at worst. Matching logic levels, and bridging them when they differ, is the quiet discipline that keeps captures clean and chips alive.

## What a logic level is

A digital line represents 1 and 0 as two voltage bands. Each family defines:

- **VOH / VOL:** the output high and low voltages a driver produces.
- **VIH / VIL:** the input high and low thresholds a receiver requires.

For two chips to communicate, the driver's VOH must clear the receiver's VIH, and neither side may exceed the other's absolute maximum input voltage.

## The common embedded voltages

| Rail | Where you see it | Notes |
| --- | --- | --- |
| **5 V** | Older logic, Arduino Uno, some sensors | Increasingly rare on signal lines. Dangerous to modern 1.8 V parts. |
| **3.3 V** | The embedded default: most SoCs' I/O, [SPI flash](../attacks/spi-flash-dump.md), [UART](../protocols/uart.md) headers | Your adapters should default here. |
| **1.8 V** | Modern SoC I/O, high-density flash and [eMMC](../protocols/emmc.md), newer cores | Common and easy to overlook. Check the datasheet. |
| **1.2 V / 1.1 V** | SoC core rails, DDR | Rarely a signal interface you connect to directly. |

!!! danger "The number one way to kill a target"
    Connecting a 5 V adapter to a 3.3 V or 1.8 V device drives current into protection diodes and can destroy an input in an instant. Always measure or look up the target's I/O voltage first, and set your adapter to match. See [Lab Setup](../getting-started/lab-setup.md#voltage-is-the-first-thing-to-get-right).

## When you can get away without shifting

- **Same voltage both sides.** A 3.3 V adapter to a 3.3 V target needs no shifting. This covers most consumer work.
- **3.3 V driving a 5 V input.** Many 5 V parts accept 3.3 V as a valid high because their VIH sits below 3.3 V. This often works but is not guaranteed; check the receiver's VIH.
- **5 V-tolerant inputs.** Some 3.3 V chips explicitly tolerate 5 V on specific pins. Only the datasheet can tell you.

## When you must shift

- **5 V driving a 3.3 V or 1.8 V input.** The high level exceeds the receiver's absolute maximum. Shift down, always.
- **3.3 V driving a 1.8 V input.** Same problem at smaller scale. Shift down.
- **1.8 V driving a 3.3 V input.** The 1.8 V high may not reach the 3.3 V VIH, so the receiver misreads ones. Shift up.

## How to shift levels

=== "Bidirectional MOSFET shifter"

    The standard for [I2C](../protocols/i2c.md) and other open-drain, bidirectional buses. A small board (often based on the BSS138 FET) with a low-voltage side, a high-voltage side, and the two reference voltages. Wire the low rail, the high rail, grounds, and the signal pairs.

    - Works for I2C, and for push-pull buses at modest speeds ([SPI](../protocols/spi.md), UART at common rates).
    - Needs both reference voltages supplied.
    - Cheap, widely available, the default choice for exploration.

=== "Dedicated level-translator IC"

    Chips like the **TXB0108** (auto-direction, push-pull) or **TXS0108** (open-drain) handle multiple channels at higher speed and are better for fast SPI or [JTAG](../protocols/jtag.md).

    - Pick auto-direction parts for buses where direction changes (SPI data), and be careful: some auto-direction parts misbehave on open-drain buses like I2C, where a MOSFET shifter is safer.

=== "Resistor divider (receive only, slow)"

    To bring a 5 V output down to 3.3 V on a line you only read, a two-resistor divider works for low speeds. For 5 V to 3.3 V, roughly 1.8 kO in series and 3.3 kO to ground.

    - One direction only. Do not use it on a line the target must also read.
    - Bandwidth limited by the resistor values and line capacitance. Fine for slow UART, poor for fast SPI.

=== "Series resistor (quick and dirty)"

    A single resistor (for example 1 kO to 3.3 kO) in series limits current into a receiver's protection diodes. It is a field expedient, not a clean solution, and can distort fast signals. Prefer a real shifter when it matters.

## Grounds must be common

Level shifting assumes every device shares a ground reference. Without a common ground, "3.3 V" on one board is not the same 3.3 V on another, levels float, and captures become nonsense. Tie all grounds together first, every time. See [Lab Setup](../getting-started/lab-setup.md).

## A practical checklist

1. Measure the target's I/O voltage with a multimeter, or read it from the datasheet.
2. Set your adapter to match if it can.
3. If voltages differ, choose a shifter: MOSFET board for I2C and general use, a translator IC for fast SPI/JTAG, a divider only for a read-only slow line.
4. Tie all grounds together.
5. Verify with a [logic analyzer](../software/logic-analyzers.md) that the shifted signal swings cleanly between the expected rails before trusting it.


## Related pages

- [Lab Setup and Safety](../getting-started/lab-setup.md)
- [UART](../protocols/uart.md)
- [I2C](../protocols/i2c.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
