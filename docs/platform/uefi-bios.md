# UEFI and BIOS

A PC's firmware (UEFI, still colloquially "the BIOS") lives in a [SPI flash](../protocols/spi.md) chip on the motherboard, exactly like a router's firmware lives in its flash. That means the same tools apply: dump the chip, carve it, and analyze it. PC firmware is a large, structured image with its own format and its own security model (Secure Boot, the [ME/PSP](intel-me-amd-psp.md), flash write protections), which this page introduces. For authorized research on your own hardware.

## Where the firmware is

The SPI flash is usually an 8-pin SOIC or WSON chip near the chipset, marked `25` series like any [SPI NOR](../reference/flash-chips.md), typically 8 to 32 MB. You get its contents two ways:

- **Software, from the running OS**: `flashrom` with the internal programmer can read the chip through the chipset.

    ```bash
    sudo flashrom -p internal -r bios.bin
    ```

    This often works for reads but may be blocked for writes, or blocked entirely, by the flash protections below.

- **Hardware, with a clip**: a [CH341A or FT2232H](../attacks/spi-flash-dump.md) on the SPI chip reads it regardless of what the running system allows. The reliable method when software reads are locked, and the only method on a bricked board.

## The image is structured

A UEFI image is not a flat blob. It is divided into regions by the **Intel Flash Descriptor**:

- **Descriptor**: defines the regions and their read/write permissions.
- **BIOS region**: the UEFI firmware volumes (DXE/PEI modules, the actual code).
- **ME region**: the [Intel Management Engine](intel-me-amd-psp.md) firmware.
- **GbE / other regions**: network controller config, etc.

Tools understand this layout:

```bash
# parse and extract UEFI structure
UEFITool bios.bin           # GUI: browse volumes, modules, extract
uefi-firmware-parser -e bios.bin
```

**UEFITool** browses and extracts firmware volumes and individual DXE drivers, which you then load into [Ghidra](../reverse-engineering/ghidra-setup.md) (x86/x64) for analysis.

## chipsec: platform security assessment

**chipsec** is Intel's open-source framework for checking platform security configuration from a running system. It is the standard way to audit a PC's firmware protections:

```bash
sudo chipsec_main                 # run the full suite of checks
sudo chipsec_util spi dump spi.bin  # dump the SPI flash via chipsec
```

chipsec flags whether the flash is write-protected (BIOS_CNTL, protected range registers), whether SMM is locked, whether Secure Boot is configured, and many other platform hardening checks. A failed check is a real finding: for example, a writable BIOS region means persistent firmware implants are possible.

## The security model and its weak points

- **Flash write protection**: the descriptor and chipset registers should make the BIOS region read-only to the OS. Misconfiguration (unlocked BIOS_CNTL, missing protected ranges) lets malware rewrite firmware. chipsec checks this.
- **UEFI Secure Boot**: verifies bootloaders against keys in firmware. Bypasses come from unsigned option ROMs, misconfigured keys, or vulnerable signed shims. This is an instance of the [chain of trust](../attacks/chain-of-trust.md).
- **SMM (System Management Mode)**: the most privileged x86 mode; SMM bugs give ring -2 code execution. A rich research area.
- **The physical attack**: a SPI clip bypasses all software protections for reading, and for writing on boards without hardware flash protection. Implanting or extracting firmware this way is the hardware-hacking endgame on a PC.

## Defender's view

Keep firmware updated (platform vulnerabilities are patched in BIOS updates), enable and verify flash write protection (run chipsec to confirm), use Secure Boot with your own keys where practical, and remember that physical access to the SPI chip defeats software-only protections. For secrets at rest, see [TPM Sniffing](../attacks/tpm-sniffing.md), which is the matching physical attack on the TPM.

## Related pages

- [SPI Flash Dumping](../attacks/spi-flash-dump.md)
- [Intel ME and AMD PSP](intel-me-amd-psp.md)
- [Chain of Trust and Secure Boot](../attacks/chain-of-trust.md)
- [TPM Sniffing](../attacks/tpm-sniffing.md)
- [flashrom](../software/flashrom.md)
