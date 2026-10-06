# Bootloader Bypass

Modern devices increasingly verify their own boot chain: the ROM checks the bootloader, the bootloader checks the kernel, and so on, so that modified firmware refuses to run. This is **secure boot** (or verified boot). Bypassing it, on hardware you are authorized to test, means finding a link in that chain that fails to check what it should, or forcing a check to pass. This page explains the concepts and the realistic attack surface.

## How a verified boot chain works

A chain of trust starts from something immutable and extends outward:

1. **Root of trust.** An on-chip boot ROM holds, or hashes, a public key burned into one-time-programmable fuses. This is immutable.
2. Each stage **verifies the next** with a signature before executing it: ROM verifies the first bootloader, which verifies U-Boot or the kernel, which verifies the root filesystem.
3. A broken signature anywhere **halts the boot**, so a straight firmware modification is rejected.

The attacker's question is always the same: where in this chain is the verification weak, missing, or skippable?

## Where chains actually break

### A stage that does not verify

The most common real-world failure. A vendor signs the kernel but forgets to verify the root filesystem, or verifies the main firmware but not a recovery image, or checks a header but not the payload it describes. If any stage loads the next without a correct check, you modify at that unverified boundary.

Finding it: [extract the firmware](firmware-extraction.md), identify each stage, and in [Ghidra](../reverse-engineering/ghidra-setup.md) locate the verification routine in each loader. A stage with no signature check, or one whose result is ignored, is your entry.

### Verification logic bugs

Even when a check exists, it may be wrong:

- **Return value ignored.** The verify function runs but its result is never tested, so a failure does not stop the boot.
- **Signature not actually bound to the data.** The check validates a signature that does not cover the whole image, letting you alter the unprotected part.
- **Downgrade.** An older, validly signed image with a known vulnerability is still accepted because there is no anti-rollback. You boot the old signed image and exploit it.
- **Debug or test keys** left enabled, or a test mode that disables verification.

### Non-verified boot sources

Some SoCs can be told to boot from an alternate source (serial download mode, USB, an SD card, a UART loader) that is less strictly verified than the normal flash path. If that mode accepts code the normal path would reject, it is a bypass. Vendor "download modes" are a recurring weak point.

## Forcing a check to pass with a glitch

When the logic is sound but you can affect the hardware, a [fault injection](fault-injection.md) at the instant of the signature comparison can make a failed check appear to pass. The CPU is executing a "branch if not equal" after comparing hashes; a well-timed voltage or clock glitch can corrupt that instruction or its operands so execution continues as though the signatures matched. This is how some verified boot chains are defeated when no software flaw exists. It is slow, probabilistic work; see [Fault Injection](fault-injection.md).

## Read-out protection versus secure boot

Keep two related but distinct protections straight:

- **Secure boot** stops the device from *running* unauthorized code.
- **Read-out protection** (RDP, CRP, APPROTECT) stops you from *reading* the firmware over a [debug port](../protocols/jtag.md).

A device can have either, both, or neither. Defeating read-out protection (often via [glitching](fault-injection.md) the protection check) gets you the code; defeating secure boot lets you run your own. An assessment may need both.

## Encrypted firmware interplay

If the image is also **encrypted**, a boot bypass alone may not help you read it, and reading it alone may not help you run code. The bootloader holds the decryption key to do its job, so reversing the bootloader (after dumping it) is frequently the linchpin: it yields both the key and the verification logic. See [Firmware Analysis](firmware-analysis.md#handling-encrypted-firmware).

## A realistic order of attack

1. **Check whether verification exists at all.** Many devices advertise secure boot but leave a stage unverified. Modify at the weakest boundary.
2. **Look for logic bugs** in the verify routines: ignored results, partial coverage, downgrade.
3. **Try alternate boot modes** that may be less strict.
4. **Only then consider glitching** the signature check, which is the most effort.

## Defender's view

Verify every stage, over the whole payload, and act on the result. Disable debug and download modes on production silicon, implement anti-rollback, never ship test keys, and assume a determined attacker will try to glitch the one comparison that matters, so make that comparison redundant and fault-resistant.
