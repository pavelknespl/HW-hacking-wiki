# Platform Security

The chips covered in [MCU Targets](../mcu/index.md) are small microcontrollers. This section is about the big platforms: PC firmware (UEFI/BIOS), the hidden coprocessors inside modern CPUs (Intel ME, AMD PSP), and mobile SoC boot and recovery modes (Qualcomm EDL, MediaTek, Android bootloaders). These are higher-complexity targets with their own tooling, but the hardware-hacking fundamentals still apply: find the firmware, read it, understand the trust chain, and look for the weak link.

<div class="grid cards" markdown>

-   __UEFI and BIOS__

    Dump PC firmware from the SPI chip and analyze it with chipsec and UEFITool.

    [:octicons-arrow-right-24: Read](uefi-bios.md)

-   __Intel ME and AMD PSP__

    The always-on coprocessors beside the main CPU, and what you can learn about them.

    [:octicons-arrow-right-24: Read](intel-me-amd-psp.md)

-   __Qualcomm EDL and MediaTek__

    Mobile SoC emergency download and recovery modes.

    [:octicons-arrow-right-24: Read](mobile-soc-edl.md)

-   __Android Bootloaders__

    fastboot, unlocking, verified boot, and where firmware lives on a phone.

    [:octicons-arrow-right-24: Read](android-bootloader.md)

</div>

## How these relate to the rest of the wiki

Everything here reduces to familiar steps. A PC BIOS lives in an [SPI flash](../protocols/spi.md) you can [clip and dump](../attacks/spi-flash-dump.md). A phone's firmware sits in [eMMC or UFS](../protocols/emmc.md) reachable through a recovery mode. The boot chains are instances of the [chain of trust](../attacks/chain-of-trust.md). The difference is scale and tooling, not kind. If this section feels advanced, the [fundamentals](../fundamentals/index.md) and [attacks](../attacks/index.md) sections are the groundwork it builds on.

## Related pages

- [MCU Targets](../mcu/index.md)
- [Chain of Trust and Secure Boot](../attacks/chain-of-trust.md)
- [TPM Sniffing](../attacks/tpm-sniffing.md)
- [SPI Flash Dumping](../attacks/spi-flash-dump.md)
