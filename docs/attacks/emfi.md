# Electromagnetic Fault Injection (EMFI)

EMFI induces faults in a running chip with a short, intense magnetic pulse from a coil held over the package. The pulse couples into the die and briefly disturbs logic or memory, corrupting an instruction, a comparison, or a memory read at a chosen moment. It is a branch of [fault injection](fault-injection.md) alongside voltage and clock glitching, with one big practical advantage: it usually needs no decapsulation and no electrical contact with the target. For authorized research on your own hardware.

## Why reach for EM rather than voltage glitching

- **No decap, no modification.** The coil works through the plastic package. You do not have to open the chip or cut power traces, unlike [laser fault injection](decapping.md) or some voltage attacks.
- **Spatial selectivity.** A small coil over a specific region of the die can target one block (the CPU core, the crypto engine, the [boot ROM](../platform/bootrom-exploitation.md) fetch) rather than the whole chip, which pure voltage glitching on the supply cannot do.
- **Works where the supply is clean.** On boards with strong decoupling or an internal regulator, injecting a voltage glitch is hard. An EM pulse sidesteps the power network entirely.

The trade-off is a larger parameter space: on top of timing you now tune position, coil orientation, and pulse energy.

## Setup

- **Pulse generator and coil.** A capacitor discharged through a small hand-wound or commercial coil (a "tip"). Commercial rigs include ChipSHOUTER; the [ChipWhisperer](../software/chipwhisperer.md) ecosystem drives both triggering and EMFI hardware.
- **XY(Z) stage.** To scan the coil across the die and find the sensitive spot. Early work is often manual; a motorized stage makes the search repeatable.
- **Trigger.** A reliable signal marking the moment to pulse, for example a GPIO edge, a [UART](../protocols/uart.md) byte, a power-trace feature, or a point in the [USB](../protocols/usb.md) exchange. Precise triggering is the whole game.

## The attack loop

The EM loop adds two search dimensions (position, energy) on top of the [voltage-glitch loop](fault-injection.md#the-workflow):

1. **Find a trigger** tied to the operation you want to fault (signature check, key comparison, fuse read).
2. **Scan position** across the package to locate a region where pulses produce faults rather than resets.
3. **Sweep timing** around the trigger with a fine delay step.
4. **Sweep pulse energy** so faults are frequent but the chip does not crash or brown out.
5. **Detect success**: a skipped check, a wrong branch, unsigned code accepted, or a dumped value. Expect a low hit rate; automate thousands of attempts.

```python
# ChipWhisperer + EMFI tip (e.g. ChipSHOUTER): add X/Y to the glitch sweep
for x in xrange:                      # motorized stage grid over the die
    for y in yrange:
        stage.move(x, y)
        for off in offsets:           # timing after trigger
            for power in powers:      # pulse energy / gain
                emfi.pulse.power = power
                scope.glitch.ext_offset = off
                reset_target(); scope.arm(); go(); scope.capture()
                if success_detector():        # unsigned code ran / value leaked
                    log_hit(x, y, off, power)
```

---

## Attack 1 — secure-boot bypass

Skip the [signature verification](chain-of-trust.md) in a [boot ROM](../platform/bootrom-exploitation.md) so unsigned code runs. Trigger on the verify window; scan the coil over the CPU-fetch region; success = unsigned next-stage executes. The go-to when there is no public ROM software bug and the supply is too clean for voltage glitching.

## Attack 2 — read-protection / fuse bypass

Fault the read of a lock [eFuse](../fundamentals/efuse-otp.md) so a protected device re-exposes [JTAG](../protocols/jtag.md) or flash. Scan over the fuse/efuse sense logic; trigger on the post-reset fuse-load window (same principle as [nRF52 APPROTECT](../mcu/nrf52-approtect.md), but contactless).

## Attack 3 — crypto differential fault analysis (DFA)

Corrupt one intermediate byte in an RSA or AES operation, capture the faulty output, and recover the key with DFA math (one good/faulty ciphertext pair recovers an AES-128 key with published DFA). Trigger on the crypto operation (power-trace feature works well); scan over the crypto engine.

## Attack 4 — FPGA config disturbance

Disturb [bitstream authentication](../fpga/bitstream-encryption.md) or force a device out of encrypted-only mode during configuration. Scan over the config logic; trigger on the config-load phase.

## Defender's view

Harden against the same effects as other [fault injection](fault-injection.md#defenders-view): verify security-critical decisions redundantly and in different code paths, use double checks with random delays so a single pulse cannot flip the whole outcome, and add light sensors, EM/glitch detectors, or active meshes on high-value parts. Because EM is spatial, physically separating and shielding the most sensitive blocks (crypto engine, fuse logic) raises the cost of finding and hitting them. Assume a determined attacker with a coil can reach the die through the package.

## Related pages

- [Fault Injection and Glitching](fault-injection.md)
- [Side-Channel Attacks](side-channel.md)
- [Decapping and Invasive Attacks](decapping.md)
- [Boot ROM Exploitation](../platform/bootrom-exploitation.md)
- [eFuse and OTP Memory](../fundamentals/efuse-otp.md)
- [ChipWhisperer](../software/chipwhisperer.md)
- [Chain of Trust and Secure Boot](chain-of-trust.md)
