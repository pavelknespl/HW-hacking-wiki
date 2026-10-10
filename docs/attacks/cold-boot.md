# Cold Boot Attacks

DRAM does not lose its contents instantly when power is removed. For seconds, and much longer when cooled, the bits persist. A cold boot attack exploits this remanence to recover whatever was in memory, most valuably **encryption keys** for full-disk encryption, from a machine that was running or suspended. For authorized testing and research on your own systems.

## Why it works

Dynamic RAM stores each bit as charge in a tiny capacitor that must be refreshed constantly. Cut the power and the charge decays, but not immediately:

- At room temperature, contents remain largely intact for **several seconds**, degrading gradually.
- **Cooling the chips** (canned air inverted, giving a cold blast, or colder) slows decay dramatically, extending the window to **minutes**, long enough to move the module or reboot into a capture tool.

So a key sitting in RAM, for example a disk-encryption key that software was careful never to write to disk, is nonetheless sitting in a medium you can read after power loss.

## Attack 1 — fast-reboot imager

**Target:** a running/suspended machine with a decrypted disk; firmware that boots fast and does not scrub RAM.
**Gear:** a USB stick with a minimal RAM-dumping image (the original PoC used `msramdump`/`bios_memimage`; a tiny Linux or iPXE that images physical memory works too).

Steps:

1. Confirm the disk key is resident (machine on, suspended, or recently off; a locked screen does not evict it).
2. Cut power and immediately boot the USB imager before the OS/firmware clears RAM.
3. Dump physical memory to the stick.

## Attack 2 — DIMM transplant (chilled)

**Target:** socketed DRAM.
**Gear:** inverted canned air (cold blast) or colder; a second attacker-controlled machine.

Steps:

1. Chill the modules to slow decay (extends the window to minutes).
2. Move the DIMM to the capture machine and image it there.
3. Note: soldered/on-package memory (most laptops, phones) cannot be transplanted — use Attack 1 instead.

## Recovering the key from the image

Disk-encryption keys and their **key schedules** have recognizable, redundant structure, so tools reconstruct them even from partially decayed memory:

```bash
aeskeyfind mem.raw           # scan a RAM image for AES key schedules
rsakeyfind mem.raw           # find RSA private keys
# for BitLocker specifically:
bitlocker2john -i mem.raw    # extract the VMK/FVEK material, then mount with dislocker
```

Feed a recovered VMK/FVEK to `dislocker` (BitLocker), `cryptsetup` (LUKS), etc. to mount the volume offline.

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
