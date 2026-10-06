# esptool

`esptool.py` is Espressif's official tool for talking to the ROM bootloader on ESP8266 and ESP32 chips over serial or USB. For a hardware hacker it is the quickest way to read the firmware off an Espressif target, inspect its security fuses, and flash modified images, without clipping the external flash. Pair it with the [ESP32 Security](../mcu/esp32-security.md) page, which covers what the protections actually stop.

## Entering the bootloader

The ESP ROM has a serial download mode selected by strapping [GPIO0](../reference/pin-reference.md#gpio0) low at reset:

- On most dev boards, esptool toggles DTR/RTS to do this automatically.
- On a bare target, pull GPIO0 to ground and reset (toggle [EN](../reference/pin-reference.md#en)), then release. See [Bootloader Access](../attacks/bootloader-access.md) for the general idea.

## Identify the chip and its fuses

Start by learning what you have and whether anything is protected:

```bash
esptool.py --port /dev/ttyUSB0 chip_id
esptool.py --port /dev/ttyUSB0 flash_id        # external flash size/JEDEC
espefuse.py --port /dev/ttyUSB0 summary        # security fuse state
```

`espefuse.py summary` is the key step: it shows whether **flash encryption** and **secure boot** are enabled, and whether in development or release mode. Most hobby and many commercial devices have neither set, which means the next command just works.

## Read the firmware

```bash
# read the whole 4 MB flash to a file
esptool.py --port /dev/ttyUSB0 read_flash 0x0 0x400000 flash.bin
```

Adjust the size to the chip's flash. If flash encryption is off, `flash.bin` is plaintext and ready for [binwalk](binwalk.md) and [Ghidra](../reverse-engineering/ghidra-setup.md) (remember the original ESP32 core is Xtensa and needs a Ghidra plugin; ESP32-C is RISC-V). If encryption is on, you get ciphertext, and the [ESP32 Security](../mcu/esp32-security.md) page covers the options.

## Write and erase

```bash
esptool.py --port /dev/ttyUSB0 write_flash 0x0 modified.bin
esptool.py --port /dev/ttyUSB0 erase_flash
```

On a device that does not enforce secure boot, writing a modified image is a direct way to run your own code. See [Firmware Modification](../attacks/firmware-modification.md).

## Why reach for esptool before clipping the flash

The ROM bootloader route is faster and less invasive than a [SOIC clip](../attacks/spi-flash-dump.md): no soldering, no risk to the chip, and it reads exactly what the CPU sees. Clip the external flash with [flashrom](flashrom.md) only when the serial bootloader is locked or you cannot reach the strapping pins.

## Related pages

- [ESP32 Security](../mcu/esp32-security.md)
- [SPI Flash Dumping](../attacks/spi-flash-dump.md)
- [Firmware Modification](../attacks/firmware-modification.md)
- [CPU Architectures](../fundamentals/architectures.md)
- [Bootloader Access (U-Boot)](../attacks/bootloader-access.md)
