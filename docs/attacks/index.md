# Attacks and Techniques

This section ties the protocols and tools into complete workflows: how to actually get firmware off a device, how to get a shell, and what to do when the easy paths are locked down. The pages are ordered roughly from least to most invasive, which is also the order you should try them.

!!! warning "Authorized use only"
    Every technique here is for devices you own or are explicitly authorized to test. Many are destructive and all void warranties. See the [home page](../index.md) note on scope.

<div class="grid cards" markdown>

-   __Reconnaissance__

    Learn the device before you touch it.

    [:octicons-arrow-right-24: Read](recon.md)

-   __Firmware Extraction__

    The decision tree for getting firmware by any available route.

    [:octicons-arrow-right-24: Read](firmware-extraction.md)

-   __SPI Flash Dumping__

    Clip on and read a NOR flash with [flashrom](../software/flashrom.md).

    [:octicons-arrow-right-24: Read](spi-flash-dump.md)

-   __NAND and eMMC Dumping__

    Larger storage, in-system and off-board.

    [:octicons-arrow-right-24: Read](nand-emmc-dump.md)

-   __Firmware Analysis__

    Carve, search, and reverse the image you extracted.

    [:octicons-arrow-right-24: Read](firmware-analysis.md)

-   __Bootloader Access (U-Boot)__

    Interrupt the bootloader and take control of boot.

    [:octicons-arrow-right-24: Read](bootloader-access.md)

-   __Getting a Shell__

    Turn a console into interactive access.

    [:octicons-arrow-right-24: Read](getting-a-shell.md)

-   __Escaping Restricted Shells__

    Break out of a locked-down menu or jailed shell.

    [:octicons-arrow-right-24: Read](restricted-shell-escape.md)

-   __Bootloader Bypass__

    Defeat [secure boot](bootloader-bypass.md) and verified boot chains.

    [:octicons-arrow-right-24: Read](bootloader-bypass.md)

-   __Fault Injection and Glitching__

    Skip a check with voltage or clock faults.

    [:octicons-arrow-right-24: Read](fault-injection.md)

-   __Side-Channel Attacks__

    Recover keys from power and timing.

    [:octicons-arrow-right-24: Read](side-channel.md)

-   __Chain of Trust and Secure Boot__

    How verified boot chains, TrustZone, and eFuse roots are built and broken.

    [:octicons-arrow-right-24: Read](chain-of-trust.md)

-   __TPM Sniffing__

    Capture keys off an unencrypted TPM bus, including BitLocker.

    [:octicons-arrow-right-24: Read](tpm-sniffing.md)

-   __Firmware Modification__

    Repack, patch, defeat signatures, and flash back.

    [:octicons-arrow-right-24: Read](firmware-modification.md)

</div>

For chip-specific read-out protection and its bypasses, see [MCU Targets](../mcu/index.md).
