# ESP32 Security

Espressif's ESP32 and ESP8266 are everywhere in cheap Wi-Fi and Bluetooth devices. Unlike [STM32](stm32-rdp.md) or [nRF52](nrf52-approtect.md), the classic ESP32 stores its firmware in an **external** SPI flash, so the default situation is wide open: clip the flash and read it. The security story is therefore about two optional features, **flash encryption** and **secure boot**, and what they do and do not stop.

## The default: external flash, wide open

A stock ESP32 keeps its application in an external [SPI flash](../protocols/spi.md). With no protections enabled you simply:

```bash
esptool.py --port /dev/ttyUSB0 read_flash 0x0 0x400000 flash.bin
```

or clip the flash chip and use [flashrom](../software/flashrom.md). That gives you the whole image, including any secrets. Most hobby and many commercial ESP32 devices ship exactly like this, so always try the plain read first.

## Flash Encryption

Flash encryption encrypts the external flash contents with a key stored in the chip's **eFuse** (one-time-programmable). The flash controller decrypts transparently at runtime, so a raw read of the external chip yields ciphertext.

Boundaries and caveats:

- A direct flash dump is now **encrypted**, so clipping the chip no longer hands you plaintext.
- The key lives in eFuse and, when properly configured, is not software-readable.
- **Development mode** flash encryption is reversible and much weaker than **release mode**; many devices are left in development mode, which undermines the protection.
- Encryption protects confidentiality of the flash, not integrity of boot, and it does not by itself stop you running your own code unless secure boot is also on.

## Secure Boot

Secure Boot makes the ROM verify a signature on the bootloader, and the bootloader verify the app, rooting trust in a key digest burned into eFuse. With it enabled, modified firmware is rejected. Espressif has shipped **Secure Boot v1** and **v2** (RSA/ECDSA based, the current recommendation). Together, flash encryption plus secure boot v2 is the intended strong configuration.

## Attack surface

- **Check what is actually enabled.** `espefuse.py summary` reads the eFuse state: whether flash encryption and secure boot are on, and whether in development or release mode. Many devices enable neither.

    ```bash
    espefuse.py --port /dev/ttyUSB0 summary
    ```

- **Development-mode flash encryption** can often be worked around, since the mode exists precisely to allow re-flashing during development.
- **Published silicon vulnerabilities.** Specific ESP32 revisions have had documented [fault-injection](../attacks/fault-injection.md) attacks against secure boot and flash encryption (for example glitching the bootloader to bypass signature verification, and attacks recovering or misusing the encryption flow on certain chip versions). The applicable attack depends heavily on the exact chip (ESP32, ESP32-S2/S3, ESP32-C3) and its revision, so identify the part precisely.
- **The newer RISC-V parts** (ESP32-C series) changed the [architecture](../fundamentals/architectures.md#espressif-a-common-special-case) and the security implementation; treat them separately from the original Xtensa ESP32.

## Workflow

1. Identify the exact ESP32 variant and revision.
2. Run `espefuse.py summary` to see which protections are set and in which mode.
3. If none, [read the flash](../attacks/spi-flash-dump.md) directly.
4. If encryption/secure boot are on, determine the mode (development vs release) and check current research for a fault-injection bypass against that specific chip revision.
5. Analyze the recovered (and decrypted) image with [binwalk](../software/binwalk.md) and [Ghidra](../reverse-engineering/ghidra-setup.md); remember the original ESP32 core is Xtensa and needs a Ghidra plugin.

## Defender's view

Enable **both** flash encryption (release mode, not development) **and** secure boot v2, on a current chip revision. Encryption alone leaves code-execution open; secure boot alone leaves secrets in plaintext flash. Verify the eFuse state on production units rather than assuming the build configuration took effect.

## Related pages

- [MCU Targets](index.md)
- [SPI Flash Dumping](../attacks/spi-flash-dump.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [CPU Architectures](../fundamentals/architectures.md)
- [Chain of Trust and Secure Boot](../attacks/chain-of-trust.md)
