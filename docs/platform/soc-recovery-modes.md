# SoC Recovery and Download Modes

Almost every application processor has a factory-programmed download mode: a way for the [boot ROM](bootrom-exploitation.md) to receive code over [USB](../protocols/usb.md) or [UART](../protocols/uart.md) when normal boot fails. It exists for manufacturing and un-bricking, but for a hardware hacker it is two things at once: a reliable path to **dump storage and RAM**, and an **attack surface** running at the highest privilege. This page is a tour of the common vendors' modes and how to enter them. For authorized testing of your own hardware.

## Why these modes matter

- **They are a free dump path.** Many download agents can read [eMMC/UFS](../protocols/emmc.md) or RAM back to the host, so you image storage without desoldering.
- **They run before or around secure boot.** The [boot ROM](bootrom-exploitation.md) code handling them is privileged and parser-heavy, so bugs here are high value.
- **They are hard to disable.** Removing them entirely breaks manufacturing, so vendors gate rather than remove, and the gate is sometimes weak.

Entering them generally means forcing the SoC to find no valid boot device: hold a strap pin, short a clock or data line on the boot flash at reset, or press a key combination. See [finding and identifying pins](../fundamentals/identifying-pins.md).

## Allwinner: FEL mode

Allwinner SoCs (sunxi) have **FEL**, a USB recovery mode in the boot ROM:

- Entered when no valid bootloader is found, or via a FEL button/pin on many boards.
- The host tool is `sunxi-fel` from **sunxi-tools**. It can read and write SRAM/DRAM, execute code, and dump memory.

```bash
sunxi-fel version
sunxi-fel read 0x0 0x8000 sram_dump.bin   # read 32 KiB of SRAM
sunxi-fel exe 0x2000                       # execute loaded code
```

FEL is a favorite because it is open, well documented, and gives easy code execution on cheap SBCs and devices.

## Rockchip: maskrom and loader mode

Rockchip SoCs expose **maskrom mode** (boot ROM USB) and a loader mode:

- Enter maskrom by shorting the data line of the boot flash at reset so the ROM cannot load, or via a maskrom button.
- Tools: `rkdeveloptool` and `rkflashtool`. You push a DRAM init blob and loader, then read/write the [eMMC](../protocols/emmc.md).

```bash
rkdeveloptool rd 3                 # reboot to maskrom
rkdeveloptool rl 0 0x10000 dump.img  # read sectors from flash
```

## Amlogic: USB boot mode

Amlogic SoCs (used in many TV boxes) have a USB boot mode reached by shorting specific points or via a reset button, driven by tools like `update` from the Amlogic burn toolkit. It loads a bootloader over USB and can access storage.

## NXP i.MX: Serial Download Protocol (SDP)

NXP i.MX application processors expose **SDP** over USB (and UART) in the boot ROM:

- Selected by boot-mode strap pins (set to "serial downloader").
- The host tool is **`uuu`** (mfgtools) or the older `imx_usb_loader`. It loads a bootloader into RAM and can script full flashing.

```bash
uuu -b spl u-boot-with-spl.imx    # load and run U-Boot over SDP
```

## MediaTek: BROM and the Preloader

MediaTek SoCs have a **BROM** (boot ROM) mode and a Preloader stage, both reachable over USB:

- Entered by holding keys or shorting while connecting USB, often with very tight timing.
- The host normally sends a **Download Agent (DA)**, which then talks to storage.
- **mtkclient** (with the **kamakiri** BROM exploit on vulnerable SoCs) bypasses DA authentication and SLA/DAA protections to read and write [eMMC/UFS](../protocols/emmc.md) and dump the preloader. This is a boot-ROM-level break, in the spirit of [checkm8](bootrom-exploitation.md).

## Qualcomm: EDL

Qualcomm's **Emergency Download (EDL / 9008)** mode is covered in depth on the [Qualcomm EDL and MediaTek](mobile-soc-edl.md) page. Entered via test points or a crashed bootloader, it loads a signed **programmer (firehose)** and exposes raw storage. Leaked or unsigned firehose programmers are the usual way in.

## The general recipe

1. **Identify the SoC** from [markings](../fundamentals/chip-identification.md) and find its download mode in the datasheet or community tools.
2. **Enter the mode**: strap pin, boot-flash short at reset, or key combo. See [identifying pins](../fundamentals/identifying-pins.md).
3. **Attach the host tool** for that vendor and confirm the device enumerates on [USB](../protocols/usb.md).
4. **Dump or inject**: image [eMMC/UFS](../protocols/emmc.md) and RAM, load a bootloader, or push code to execute.
5. **Escalate** if the mode is authenticated, via a [boot ROM exploit](bootrom-exploitation.md) or [glitch](../attacks/fault-injection.md).

## Defender's view

Authenticate the download agent/programmer and the code the ROM accepts (signed firehose, signed DA, SLA/DAA), gate download mode behind an [authenticated unlock](android-bootloader.md) or a production [eFuse](../fundamentals/efuse-otp.md), and do not rely on the mode being obscure: the entry straps and tools are community knowledge. Treat storage reachable through recovery as reachable by an attacker, and encrypt data at rest accordingly.

## Related pages

- [Boot ROM Exploitation](bootrom-exploitation.md)
- [Qualcomm EDL and MediaTek](mobile-soc-edl.md)
- [Android Bootloaders](android-bootloader.md)
- [Firmware Extraction](../attacks/firmware-extraction.md)
- [eMMC and SD](../protocols/emmc.md)
- [USB](../protocols/usb.md)
