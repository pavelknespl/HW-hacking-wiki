# Lab Setup and Safety

You do not need a laboratory. A corner of a desk, decent light, and about a hundred dollars of gear will carry you through most consumer device assessments. This page describes a bench that grows with you and the handful of safety habits that keep both you and your targets alive.

## The three tiers

=== "Starter (around 50 to 100 USD)"

    Enough to find a UART, read a flash chip, and watch simple buses.

    - A USB to multi-protocol adapter such as a **Bus Pirate**, **FT2232H breakout**, or **CH341A** programmer
    - A cheap **USB logic analyzer** (an 8 channel clone works with [sigrok](../software/logic-analyzers.md))
    - A **multimeter** with continuity beep
    - **Jumper wires**, **SOIC-8 test clip**, and a **breadboard**
    - A basic soldering iron

=== "Intermediate (around 300 to 600 USD)"

    Comfortable, reliable work on a wide range of boards.

    - A quality **JTAG/SWD probe** such as a J-Link or an FT2232H based adapter
    - A **temperature controlled soldering station** and **hot air rework** station
    - A proper **bench power supply** with current limiting
    - A real **logic analyzer** (Saleae clone or a Saleae) at higher sample rates
    - A **binocular microscope** or a USB microscope
    - A **flash programmer** that handles SPI, I2C, and parallel chips

=== "Advanced"

    For fault injection, side-channel work, and dense boards.

    - An **oscilloscope** (bandwidth of 100 MHz or more)
    - A **ChipWhisperer** for glitching and power analysis, or a custom crowbar glitcher
    - A **hot air / preheater / IR** rework setup for BGA work
    - A **soldering microscope** with a boom arm
    - An **X-ray** service or shop access for hidden vias (usually outsourced)

## The essential bench tools explained

| Tool | Why you want it |
| --- | --- |
| Multimeter | Identify ground, VCC, and continuity. The single most used tool on the bench. |
| Logic analyzer | Capture and decode UART, SPI, I2C, and JTAG traffic. Turns guesses into facts. |
| USB to UART adapter | The workhorse for serial consoles. Must support selectable voltage (see below). |
| SOIC test clip | Read a flash chip without desoldering it. Saves hours. |
| Bench power supply | Power a board externally and watch its current draw. Current limiting prevents cooked chips. |
| Soldering station | Attach wires to tiny pads and remove chips cleanly. |

## Voltage is the first thing to get right

The fastest way to destroy a target is to connect a 5 V adapter to a 3.3 V or 1.8 V device. Embedded logic levels are commonly **3.3 V**, often **1.8 V**, occasionally **5 V**, and sometimes **1.2 V** on newer SoCs.

Rules that will save your hardware:

- Confirm the target's I/O voltage with a multimeter or the chip datasheet **before** connecting anything.
- Buy adapters with a selectable voltage jumper and set it to match the target.
- When voltages differ between devices, use a [level shifter](../fundamentals/level-shifting.md). Do not rely on luck.
- Share a common ground between every device on the bench. Floating grounds give garbage captures and can damage parts.

!!! danger "Never connect two power sources"
    If the target is powered by its own supply, do **not** also feed VCC from your adapter. Connect only the data lines and the shared ground. Driving two supplies into one rail can destroy either device.

## Powering the target

You have two sane options:

1. **Let the device power itself** from its normal supply and connect only ground and signal lines to your tools. Safest default.
2. **Power it from a bench supply** with the current limit set just above the device's idle draw. The current reading becomes a free diagnostic: a sudden spike often marks the moment a fault injection lands or a chip faults.

## Personal and electrical safety

- **Mains voltage kills.** Many consumer devices contain a mains power supply section. Treat anything connected to the wall, and large capacitors near it, as dangerous even when unplugged. Capacitors hold charge. If you are not confident, power the low-voltage logic board separately from a bench supply and leave the mains section alone.
- **Ventilate.** Solder fumes and flux are not good for you. Use a fume extractor or a small fan and work in open air.
- **Lead and chemicals.** Wash your hands after soldering. Keep food away from the bench.
- **ESD.** A grounding strap is cheap insurance for sensitive parts, though most robust embedded boards tolerate casual handling.
- **Eyes.** Flux spatters and clipped leads fly. Safety glasses during soldering and desoldering are worth the minor annoyance.

## Keep a lab notebook

Photograph every board before and after modification. Label the chips. Record which pin you connected to what, what voltage you measured, and what each command returned. When a device bricks or a capture confuses you, good notes are the difference between a five minute recovery and starting over.
