# Decapping and Invasive Attacks

When every non-invasive and semi-invasive path is exhausted, the last resort is to open the chip itself: remove the packaging, expose the silicon die, and read or manipulate it directly. These invasive techniques, decapsulation, microprobing, and focused ion beam editing, are the high end of hardware attacks: powerful, expensive, destructive, and the domain of well-equipped labs. This page explains what they are and when they come into play, for research context.

## Where this sits

Invasive attacks are the bottom of the [methodology](../getting-started/methodology.md) escalation. You reach for them only when:

- The firmware is in **internal flash** with [read-out protection](../mcu/stm32-rdp.md) that resisted [glitching](fault-injection.md).
- A **key** lives in on-chip fuses or secure storage unreachable by [side-channel](side-channel.md) or software.
- The target justifies the cost (research, high-value forensics), because these methods can run from thousands to a great deal more and often destroy the part.

## Decapsulation

Removing the package to expose the die:

- **Chemical**: hot fuming nitric or sulfuric acid dissolves the epoxy over the die, leaving the silicon intact. The common lab method. Dangerous chemistry requiring a fume hood, protective equipment, and care; not a casual bench activity.
- **Mechanical / laser**: milling or lasering away the package, sometimes before a chemical finish.

The result is a working chip with its die visible under a microscope, which you can then image or probe.

## Attacks once the die is exposed

- **Attack 1 — optical imaging / RE**: photograph the die layer by layer (delayering between shots) to reconstruct the circuit, including **mask ROM** contents, which read optically because the bits are physically encoded. How some ROM-stored secrets and algorithms are recovered.
- **Attack 2 — microprobing**: land fine needles (or use a probe station) on internal buses to read data as the chip runs — e.g. tap an internal memory bus to capture firmware or keys in transit inside the chip, below any external protection.
- **Attack 3 — optical (laser) fault injection**: a focused laser pulse flips bits or faults specific transistors with far more precision than [voltage glitching](fault-injection.md) or [EMFI](emfi.md), defeating protections that resist electrical faults. Same [glitch loop](fault-injection.md#the-workflow) logic, but the "position" axis is micron-scale on the bare die.
- **Attack 4 — Focused Ion Beam (FIB) edit**: cut and reconnect traces on the die — sever a security fuse's sense line, or reconnect a disabled debug path. Circuit edit at the nanometer scale.

## Reading fuses and secure storage

eFuse/OTP values and secure-storage cells can sometimes be read optically or electrically once the die is exposed and delayered, recovering the [root-of-trust](chain-of-trust.md) keys or unlock bits that gate everything else. This is the invasive counterpart to glitching a fuse read.

## The realities

- **Destructive**: decapping and delayering consume the part. Have spares and expect one-shot attempts.
- **Expensive and specialized**: probe stations, FIB, and microscopy are lab equipment, often outsourced to a specialist service rather than owned.
- **Countermeasures exist**: modern secure chips add active shield meshes (detecting probing), light sensors (detecting decap/laser), scrambled and buried buses, and glue logic specifically to make invasive attacks harder.

For almost all practical work you will never need this; it matters to know it exists, because it defines the ceiling of what a determined, well-funded attacker can do, and therefore what "secure" really means for a given chip.

## Defender's view

For threat models that include well-resourced invasive attackers, choose silicon with certified physical countermeasures (active shields, sensors, secure elements rated against invasive attack), store keys in [secure elements](secure-elements.md) designed to resist decap and probing, and assume that any unprotected on-die secret can eventually be read by a lab with enough budget.

## Related pages

- [Fault Injection and Glitching](fault-injection.md)
- [Side-Channel Attacks](side-channel.md)
- [Secure Elements](secure-elements.md)
- [MCU Targets](../mcu/index.md)
- [Chain of Trust and Secure Boot](chain-of-trust.md)
