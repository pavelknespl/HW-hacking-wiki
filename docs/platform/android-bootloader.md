# Android Bootloaders

An Android phone is an ARM SoC running a layered, verified boot chain, with a well-defined low-level interface, **fastboot**, that a hardware hacker can reach over USB. Understanding the bootloader, how unlocking works, and where firmware lives is the entry point for most phone security work. For authorized work on devices you own.

## The boot chain

Android boot is a textbook [chain of trust](../attacks/chain-of-trust.md):

```
Boot ROM -> Primary bootloader (SoC, e.g. Qualcomm XBL) -> ABL/aboot (fastboot)
          -> boot/init_boot (kernel + ramdisk) -> system (verified by AVB/dm-verity)
```

Each stage verifies the next, rooted in keys fused into the SoC. **Android Verified Boot (AVB / vbmeta)** and **dm-verity** protect the kernel and system partitions, so modified images are rejected unless the bootloader is unlocked.

## fastboot

The **ABL** (Android Bootloader) exposes fastboot over USB, entered with a key combo or `adb reboot bootloader`:

```bash
fastboot devices
fastboot getvar all           # device, bootloader, unlock state, partitions
fastboot oem device-info      # lock status on many devices
```

fastboot is how you query state, flash partitions (when unlocked), and boot images. A **locked** bootloader refuses to flash or boot unsigned images; that lock is the main protection.

## Unlocking

Official unlocking (where the vendor allows it) flips the bootloader to an unlocked state:

```bash
fastboot flashing unlock      # or: fastboot oem unlock
```

Important consequences:

- Unlocking **wipes user data** by design (so unlocking cannot be used to steal data from a found phone).
- An unlocked bootloader lets you flash a custom boot image (patched kernel, Magisk for root) and boot unsigned images.
- Many carrier/vendor devices **disallow** unlocking, or gate it behind a vendor token, which is the obstacle.

## Where firmware lives, and how to get it

Phone firmware sits in [eMMC or UFS](../protocols/emmc.md) as many partitions (boot, vendor, system, modem, etc.). Routes to it:

- **Official factory images**: many vendors publish full firmware packages; the easiest source for [analysis](../attacks/firmware-analysis.md).
- **fastboot (unlocked)**: read/boot images; flashing a patched boot image is the standard root path.
- **Recovery / download mode below the bootloader**: [Qualcomm EDL or MediaTek BROM](mobile-soc-edl.md) can read storage even when fastboot is locked, which is why those modes matter so much.
- **Chip-off** [eMMC/UFS](../attacks/nand-emmc-dump.md) as a last resort.

## The attack surface

- **Locked vs unlockable**: whether the vendor permits unlocking, and whether the unlock is enforced properly, is the first question.
- **Downgrade / anti-rollback**: flashing an older signed image with a known bug, blocked by rollback counters when implemented.
- **Secure boot / AVB flaws**: a stage that fails to verify the next, an accepted unsigned partition, or a vbmeta misconfiguration, is the classic [bootloader bypass](../attacks/bootloader-bypass.md).
- **Below-bootloader modes**: [EDL/BROM](mobile-soc-edl.md) bypass fastboot's lock entirely, the reason they are a prime target.
- **Encrypted userdata**: modern Android encrypts user data with keys bound to the hardware keystore, so even raw storage access yields ciphertext without the keys.

## The workflow

1. `fastboot getvar all` to read bootloader, unlock state, and partition layout.
2. Obtain firmware (factory image, or dump via fastboot/[EDL](mobile-soc-edl.md)).
3. Analyze partitions with [binwalk](../software/binwalk.md) and [Ghidra](../reverse-engineering/ghidra-setup.md); the kernel, init, and vendor binaries are the meat.
4. If the bootloader is locked and unofficial, evaluate [below-bootloader modes](mobile-soc-edl.md) or known AVB/bootloader bypasses for that exact model.

## Defender's view

Keep the bootloader locked, enforce AVB and rollback protection, bind data encryption to a hardware keystore so storage access alone is useless, and secure the [download modes](mobile-soc-edl.md) that sit below the bootloader. Tie unlocking to a data wipe so it cannot exfiltrate user data.

## Related pages

- [Qualcomm EDL and MediaTek](mobile-soc-edl.md)
- [Chain of Trust and Secure Boot](../attacks/chain-of-trust.md)
- [Bootloader Bypass](../attacks/bootloader-bypass.md)
- [eMMC and SD](../protocols/emmc.md)
- [Firmware Analysis](../attacks/firmware-analysis.md)
