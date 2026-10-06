# Fault Injection and Glitching

Fault injection deliberately pushes a chip outside its safe operating conditions for a brief moment, so that a single instruction misbehaves. Done at the right instant, that one glitch can skip a security check, corrupt a comparison, or drop the device into a state it should never reach. It is how attackers defeat [read-out protection](../protocols/swd.md#read-out-protection-again) and [secure boot](bootloader-bypass.md) when no software flaw exists. This page explains the physics, the methods, and the realistic workflow on hardware you own.

## The core idea

A processor assumes a stable supply voltage and a clean clock. Violate that assumption for a few nanoseconds and the CPU can:

- **Skip an instruction**, for example the one that acts on a failed signature check.
- **Corrupt a value** being loaded or compared, so a "not equal" becomes "equal."
- **Mis-fetch an instruction**, executing something unintended.

The attacker's whole job is to land that disturbance at the precise moment a security-critical instruction executes. Everything else is tooling to achieve timing and repeatability.

## The methods

=== "Voltage glitching"

    Briefly drop (or spike) the core supply voltage. The usual technique is a **crowbar**: a fast transistor momentarily shorts the power rail to ground for a controlled number of nanoseconds. Cheap, effective, and the most common starting point.

    - Target the **core** rail, not the I/O rail, for logic faults.
    - Needs a fast switch (a MOSFET driven by an FPGA or a [ChipWhisperer](../getting-started/toolkit.md#specialist-gear)) and control over glitch **width** and **offset** (delay from a trigger).

=== "Clock glitching"

    Insert a too-short or extra clock pulse so the CPU latches a value before it is stable, causing a faulted instruction. Only works on chips driven by an **external** clock you can manipulate (many modern parts use an internal oscillator, which rules this out).

    - Precise and repeatable where applicable.
    - Requires access to the clock line and a glitch generator that can inject a malformed edge.

=== "Electromagnetic (EMFI)"

    A fast, localized magnetic pulse from a small coil induces currents inside the die, faulting nearby logic. Works without electrical contact and can be aimed at a region of the chip.

    - More expensive gear (an EM pulse generator and a probe), and an XY stage to scan positions.
    - Effective against chips with internal clocks and some voltage-glitch countermeasures.

=== "Optical / laser"

    Decapsulate the chip and fault specific transistors with a laser. The most precise and most powerful method, and the most demanding: it needs decapping, a laser station, and die-level targeting. Reserved for well-resourced labs.

## The equipment

- **ChipWhisperer** (Lite, Nano, Husky) is the standard integrated platform: it generates glitches, provides the trigger and timing control, and has thorough tutorials. The best place to learn.
- A **custom crowbar glitcher** (a MOSFET plus an FPGA or even a fast microcontroller) is a cheap DIY entry for voltage glitching.
- An **oscilloscope** to see the power rail and confirm your glitch is landing where you think.
- A **bench supply** whose current readout doubles as a fault indicator.

## The workflow

Glitching is a search for the right parameters, so structure matters more than luck.

1. **Pick the target moment.** Identify the instruction or operation you want to disturb: the signature compare, the read-protection check, a password comparison. [Firmware analysis](firmware-analysis.md) tells you where it is in the boot flow.
2. **Establish a trigger.** You need a repeatable reference point in time. A GPIO that toggles, a recognizable pattern on a bus, a [UART](../protocols/uart.md) byte, or the power signature of the operation itself. The glitch offset is measured from this trigger.
3. **Instrument the outcome.** Define how you know a glitch succeeded: the device prints something it should not, a locked read returns data, a check passes. Automate detecting that.
4. **Sweep the parameters.** Scan glitch **offset** (delay after trigger) and **width** across ranges, many attempts per point, logging results. Success is often rare (a fraction of a percent of attempts), so you repeat and look for the parameter window that works.
5. **Classify each attempt** as normal, crash/reset, or success. Crashes bracket the interesting region: the boundary between "normal" and "crash" is where faults often land.

```
for offset in range(offset_min, offset_max):
    for width in range(width_min, width_max):
        for attempt in range(N):
            arm_glitch(offset, width)
            reset_target()
            result = observe()        # normal | crash | SUCCESS
            log(offset, width, attempt, result)
```

That loop is the essence of every glitching campaign, whatever the tooling.

## What it is used for

- **Defeating [read-out protection](../protocols/swd.md)** to dump internal MCU flash the debug port refuses to read.
- **Bypassing [secure boot](bootloader-bypass.md)** by faulting the signature comparison. See [Bootloader Bypass](bootloader-bypass.md).
- **Skipping a password or PIN check** in a bare-metal device.
- **Reaching a hidden state**, such as a serial loader or debug path normally gated by a check.

## Realistic expectations

Glitching is probabilistic, slow, and device-specific. You will spend most of the time finding a reliable trigger and a parameter window, and many targets include countermeasures: supply filtering, voltage and clock sensors, redundant checks, and randomized timing. It is the right tool only when cheaper software and memory routes are exhausted, which is why it sits near the bottom of the [methodology](../getting-started/methodology.md).

## Defender's view

Make the critical check fault-resistant: perform it twice with different code, verify the result in more than one place, add random delays so the attacker cannot find a stable trigger, and use a chip with glitch sensors where the threat warrants it. A single comparison that decides everything is exactly what fault injection targets.


## Related pages

- [Side-Channel Attacks](side-channel.md)
- [Bootloader Bypass](bootloader-bypass.md)
- [SWD](../protocols/swd.md)
- [JTAG](../protocols/jtag.md)
- [Automating Attacks](../scripting/automation.md)
