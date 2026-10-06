# Scripting and Automation

Hardware attacks get powerful when you stop doing them by hand. A [Raspberry Pi](gpio-control.md) or a microcontroller driving the target's pins lets you automate flash reads, bus transactions, and the thousands of repetitions a [glitching](../attacks/fault-injection.md) campaign needs. This section covers controlling pins from code and wiring your tools into repeatable scripts.

<div class="grid cards" markdown>

-   __GPIO Control__

    Drive and read pins from a Raspberry Pi: [UART](../protocols/uart.md), [SPI](../protocols/spi.md), [I2C](../protocols/i2c.md), and bit-banging.

    [:octicons-arrow-right-24: Read](gpio-control.md)

-   __Automating Attacks__

    Turn manual steps into scripts: scripted dumps, glitch sweeps, and harnesses.

    [:octicons-arrow-right-24: Read](automation.md)

</div>

## Why a Raspberry Pi is a hardware-hacking workhorse

A Pi (or similar single-board computer) gives you a full Linux system with directly controllable GPIO pins at 3.3 V logic. From one board you can be a UART console, a [SPI flash](../attacks/spi-flash-dump.md) reader, an I2C master, a [JTAG](../protocols/jtag.md) adapter, and a programmable trigger source, all scripted in Python. It bridges the gap between "poke pins by hand" and "run ten thousand attempts overnight."

!!! warning "Pi GPIO is 3.3 V and not 5 V tolerant"
    The Raspberry Pi's GPIO pins are 3.3 V and can be damaged by 5 V. Match levels and use a [level shifter](../fundamentals/level-shifting.md) when the target is not 3.3 V, and always share a common ground.
