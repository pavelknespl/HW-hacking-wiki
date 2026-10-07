# Attacks and Techniques

This section ties the protocols and tools into complete workflows: how to actually get firmware off a device, how to get a shell, and what to do when the easy paths are locked down. The pages are ordered roughly from least to most invasive, which is also the order you should try them.

!!! warning "Authorized use only"
    Every technique here is for devices you own or are explicitly authorized to test. Many are destructive and all void warranties. See the [home page](../index.md) note on scope.

## Recon and firmware acquisition

Learn the target, then get its firmware off by whatever route is open.

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

</div>

## Gaining access

Turn a boot interface or console into control of the running system.

<div class="grid cards" markdown>

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

</div>

## Fault and physical attacks

When software paths are locked down, attack the silicon and the physics.

<div class="grid cards" markdown>

-   __Fault Injection and Glitching__

    Skip a check with voltage or clock faults.

    [:octicons-arrow-right-24: Read](fault-injection.md)

-   __Side-Channel Attacks__

    Recover keys from power and timing.

    [:octicons-arrow-right-24: Read](side-channel.md)

-   __Electromagnetic Fault Injection__

    Induce faults with an EM pulse, no decap and no contact needed.

    [:octicons-arrow-right-24: Read](emfi.md)

-   __Cold Boot Attacks__

    Recover keys from DRAM remanence after power loss.

    [:octicons-arrow-right-24: Read](cold-boot.md)

-   __DMA Attacks__

    Read and patch RAM over Thunderbolt/PCIe with PCILeech.

    [:octicons-arrow-right-24: Read](dma-attacks.md)

-   __Secure Elements__

    Attacking the system around a key-storage chip.

    [:octicons-arrow-right-24: Read](secure-elements.md)

-   __Decapping and Invasive__

    Die-level: decap, microprobing, FIB, optical fault injection.

    [:octicons-arrow-right-24: Read](decapping.md)

</div>

## Boot security and persistence

Understand and defeat the trust chain, and the stores that survive a reboot.

<div class="grid cards" markdown>

-   __Chain of Trust and Secure Boot__

    How verified boot chains, TrustZone, and eFuse roots are built and broken.

    [:octicons-arrow-right-24: Read](chain-of-trust.md)

-   __TPM Sniffing__

    Capture keys off an unencrypted TPM bus, including BitLocker.

    [:octicons-arrow-right-24: Read](tpm-sniffing.md)

-   __Firmware Modification__

    Repack, patch, defeat signatures, and flash back.

    [:octicons-arrow-right-24: Read](firmware-modification.md)

-   __NVRAM and U-Boot Environment__

    Recover credentials, keys, and boot logic from config stores.

    [:octicons-arrow-right-24: Read](nvram-uboot-env.md)

</div>

For chip-specific read-out protection and its bypasses, see [MCU Targets](../mcu/index.md). For PC and mobile platforms, see [Platform Security](../platform/index.md).
