# Cold Boot Attacks

DRAM does not lose its contents instantly when power is removed. For seconds, and much longer when cooled, the bits persist. A cold boot attack exploits this remanence to recover whatever was in memory, most valuably **encryption keys** for full-disk encryption, from a machine that was running or suspended. For authorized testing and research on your own systems.

## Why it works

Dynamic RAM stores each bit as charge in a tiny capacitor that must be refreshed constantly. Cut the power and the charge decays, but not immediately:

- At room temperature, contents remain largely intact for **several seconds**, degrading gradually.
- **Cooling the chips** (canned air inverted, giving a cold blast, or colder) slows decay dramatically, extending the window to **minutes**, long enough to move the module or reboot into a capture tool.

So a key sitting in RAM, for example a disk-encryption key that software was careful never to write to disk, is nonetheless sitting in a medium you can read after power loss.

## The attack

1. **Target a running or suspended machine** whose disk is decrypted (the key is in RAM). A locked screen does not help the defender here; the key is still resident.
2. **Drop power and capture fast**, two common variants:
    - **Reboot into a tiny imager**: cut power and immediately boot a minimal USB/PXE tool that dumps RAM before the OS clears it. Works when boot is fast and memory is not scrubbed.
    - **Transplant the DIMM**: with the chips chilled, physically move the module to an attacker-controlled machine and image it there.
3. **Recover keys from the image**: disk-encryption keys (and their key schedules) have recognizable structure. Tools scan a RAM dump for **AES key schedules** and reconstruct the key even from partially decayed memory, because the schedule is redundant.

## What it defeats

- **Full-disk encryption** (BitLocker without a PIN, LUKS, FileVault) when the machine is powered on, suspended, or recently off, because the key lives in RAM.
- Any secret an application holds in cleartext memory.

It complements [TPM sniffing](tpm-sniffing.md): both recover a disk key a physical attacker is not supposed to get, one from the TPM bus, one from RAM.

## Modern obstacles

- **Memory encryption**: Intel TME / SGX and AMD SME/SEV encrypt DRAM contents, so a cold-boot image is ciphertext. Where enabled, this largely closes the attack.
- **Soldered, on-package memory** (many laptops, phones) cannot be transplanted, limiting you to the fast-reboot variant.
- **Faster, scrubbing boot**: firmware that clears RAM on boot (MemoryOverwriteRequest) shortens the reboot window.
- **Suspend-to-RAM policies**: forcing a key eviction on lock or suspend removes the target from memory.

## Defender's view

- Use **memory encryption** (TME/SME) where available; it is the real fix.
- Require a **pre-boot PIN/passphrase** and configure the disk encryption to **evict keys on suspend/lock** rather than keeping them resident.
- Prefer **hibernate (suspend-to-disk)** over suspend-to-RAM for sensitive machines, and enable firmware memory scrubbing.
- Treat a powered-on encrypted machine as effectively unlocked to a physical attacker.

## Related pages

- [TPM Sniffing](tpm-sniffing.md)
- [DMA Attacks](dma-attacks.md)
- [Side-Channel Attacks](side-channel.md)
- [Memory Types](../fundamentals/memory-types.md)
- [Chain of Trust and Secure Boot](chain-of-trust.md)
