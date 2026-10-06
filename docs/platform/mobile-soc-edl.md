# Qualcomm EDL and MediaTek

Mobile and embedded SoCs keep a low-level **recovery / emergency download mode** in ROM that accepts code and firmware over USB when normal boot fails. On Qualcomm this is **EDL** (Emergency Download, also called Sahara/Firehose); on MediaTek it is the **BROM** / Preloader download mode. These modes are how phones are unbricked and reflashed at the factory, and they are a powerful, sometimes under-protected entry point. For authorized work on your own devices.

## Qualcomm EDL (Sahara / Firehose)

EDL is a mode in the Qualcomm boot ROM. Entered by a hardware condition (a test point shorted to ground at boot, a special USB cable, or a command like `adb reboot edl`), the device enumerates as a Qualcomm USB device and speaks the **Sahara** protocol.

The twist is the **programmer (Firehose)**: EDL itself cannot do much until the host uploads a small signed loader (an XML-driven "Firehose programmer", a `prog_emmc_firehose_*.mbn`). Once loaded, the programmer exposes read/write access to the device storage ([eMMC/UFS](../protocols/emmc.md)):

```bash
# open-source EDL client
edl printgpt                 # read the partition table
edl r boot boot.img          # read the boot partition
edl w aboot aboot.img        # write a partition (if the loader allows)
```

(`edl` here is the community `bkerler/edl` tool; `QFIL` is the vendor equivalent.)

Security points:

- The Firehose programmer is **signed**, so you generally need the correct loader for the SoC. Leaked and vendor loaders circulate, and that is often the gating factor.
- Some loaders enforce authentication or partition restrictions; others are permissive. Permissive or leaked loaders have enabled full storage dumps (and real-world forensic and unlock tooling) on many devices.
- EDL gives **raw storage access below the Android bootloader**, so it can bypass protections that live at the [fastboot/bootloader](android-bootloader.md) level.

## MediaTek BROM / Preloader

MediaTek SoCs expose a **BROM download mode** entered by holding a key combo or grounding a test point while connecting USB (before the Preloader runs). Tools like **mtkclient** (and the vendor **SP Flash Tool**) talk to it:

```bash
mtkclient r boot boot.bin    # read a partition via BROM
mtkclient printgpt
```

MediaTek's BROM has been a notably soft target: the **"Kamakiri"** class of exploits defeated BROM download-mode authentication on many chips, enabling storage read/write and even recovery of the hardware-unique keys used for storage encryption on affected SoCs. The exact result is highly chip-and-revision dependent.

## Why these modes matter

- They are an [extraction path](../attacks/firmware-extraction.md) that sits **below** the OS and often below the Android bootloader's protections, so they can dump storage even when fastboot is locked.
- They are how bricked devices are revived, so they are intentionally hard to disable entirely.
- Entry often needs only a **test point** on the board (an EDL or BROM pad shorted to ground at boot), which is pure hardware recon. See [PCB Recon](../fundamentals/pcb-recon.md).

## The workflow

1. **Enter the mode**: find the test point or key combo, or use a software trigger if available.
2. **Identify the SoC** and obtain the matching loader (Firehose for Qualcomm) or exploit (for MediaTek BROM).
3. **Read the partition table**, then dump the partitions you want.
4. **Analyze** the extracted images as usual ([binwalk](../software/binwalk.md), [Ghidra](../reverse-engineering/ghidra-setup.md)); storage may be encrypted, which pushes you toward key recovery.

## Defender's view

Treat download modes as a real attack surface: require authenticated, device-bound loaders; do not reuse keys across a fleet; bind storage encryption to keys the download mode cannot extract; and patch BROM/Preloader vulnerabilities (where the SoC allows). Physical access to a board's download-mode pads is realistic, so the mode must authenticate, not merely be hidden.

## Related pages

- [Android Bootloaders](android-bootloader.md)
- [Firmware Extraction](../attacks/firmware-extraction.md)
- [eMMC and SD](../protocols/emmc.md)
- [PCB Recon](../fundamentals/pcb-recon.md)
- [Chain of Trust and Secure Boot](../attacks/chain-of-trust.md)
