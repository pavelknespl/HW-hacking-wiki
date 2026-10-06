# Memory Types

Firmware has to live somewhere, and where it lives determines how you extract it. This page covers the storage technologies you will meet, how to recognize each, and the right tool for reading it. Pair it with [Chip Identification](chip-identification.md) and the dumping guides under [Attacks](../attacks/index.md).

## The landscape

| Type | Interface | Typical size | Package | How you read it |
| --- | --- | --- | --- | --- |
| **SPI NOR flash** | SPI | 1 to 32 MB | SOIC-8, WSON-8 | Clip and [`flashrom`](../software/flashrom.md) |
| **SPI NAND flash** | SPI | 128 MB to 2 GB | SOIC-8, WSON-8 | SPI reader with NAND support |
| **Parallel NAND** | Parallel (x8/x16) | 128 MB to several GB | TSOP-48 | Desolder and NAND programmer |
| **eMMC** | MMC bus | 4 to 128 GB | BGA-153/169 | [In-system or socketed reader](../attacks/nand-emmc-dump.md) |
| **EEPROM** | I2C or SPI/Microwire | bytes to 1 Mbit | SOIC-8, TSSOP | [I2C](../protocols/i2c.md) dump or programmer |
| **Internal MCU flash** | On-chip | tens of KB to a few MB | inside the SoC | [JTAG](../protocols/jtag.md)/[SWD](../protocols/swd.md) if not locked |
| **DRAM** | Parallel memory bus | MB to GB | BGA, TSOP | Volatile, not a firmware store |

## NOR versus NAND

The distinction matters because it changes both what the data looks like and how you read it.

### NOR flash

- **Random access, execute in place.** The CPU can run code directly from NOR, so bootloaders and small firmware often live here.
- **No bad blocks, simple layout.** What you read is what is stored, byte for byte. No translation layer.
- **Smaller.** Usually up to 32 MB, which is why it holds bootloaders and compact firmware rather than whole Linux filesystems on larger devices.
- **Easy to dump.** A SOIC-8 SPI NOR is the friendliest target in hardware hacking: clip on, run `flashrom`, done. See [SPI Flash Dumping](../attacks/spi-flash-dump.md).

### NAND flash

- **Block access, higher density, cheaper per bit.** Holds large filesystems.
- **Bad blocks and ECC.** NAND ships with and develops unusable blocks. A controller or the OS tracks them and applies error-correcting codes. A raw dump therefore includes spare/OOB bytes and bad-block markers that you must interpret.
- **Needs reassembly.** A raw NAND read is not directly a filesystem. You often strip ECC/OOB, handle the bad-block table, and only then carve it. See [NAND and eMMC Dumping](../attacks/nand-emmc-dump.md).

## eMMC

eMMC is NAND plus an integrated controller in one BGA package. The controller hides the bad blocks, ECC, and wear leveling behind a clean block device, which is a gift:

- A raw eMMC read is a clean image with partitions, like reading a disk. No OOB wrangling.
- It can often be read **in-system** by tapping the clock, command, and one data line (`CLK`, `CMD`, `DAT0`) plus power and ground, using an SD card reader or a dedicated eMMC adapter. See [eMMC and SD](../protocols/emmc.md).
- When in-system reads fail, the chip is reballed onto an adapter and read in a socket.

## EEPROM

Small non-volatile memories, usually on I2C (the `24Cxx` family) or Microwire (`93Cxx`). They rarely hold firmware but frequently hold the interesting small secrets:

- MAC addresses and serial numbers.
- Wi-Fi calibration data.
- Configuration flags, sometimes including security settings or unlock states.

Dump an I2C EEPROM by [sniffing or mastering the I2C bus](../protocols/i2c.md), or clip it and read with a programmer.

## Internal MCU flash

On a microcontroller, the firmware lives inside the chip. There is no external chip to clip. Your options:

- **Debug port.** If [SWD](../protocols/swd.md) or [JTAG](../protocols/jtag.md) is enabled and read-out protection is off, OpenOCD reads the internal flash directly.
- **Read-out protection.** Most MCUs offer a lock bit (STM32 RDP, the NXP CRP, the ESP32 flash encryption and secure boot). When set, the debug port refuses to read flash. Defeating it is where [fault injection](../attacks/fault-injection.md) comes in.
- **Bootloader ROM.** Some MCUs expose a built-in serial or USB bootloader that may or may not respect the read protection, occasionally a weak point.

## DRAM

Volatile working memory. It is not where firmware is stored, but it is where secrets live at runtime: decrypted keys, plaintext configuration, and the running code of an otherwise encrypted system. Reading DRAM generally means halting the CPU over [JTAG](../protocols/jtag.md) and dumping memory, or in specialized cases a cold-boot attack. It is an advanced target, not a routine dump.

## Choosing your approach

1. Identify the storage chips during [recon](pcb-recon.md).
2. If firmware sits in an external SPI NOR, clip and dump it first. It is the easiest win.
3. For SPI NAND or eMMC, plan for reassembly (NAND) or an in-system tap (eMMC).
4. For an MCU with internal flash, try the debug port, and expect read-out protection that may require a physical attack.
5. Always check small EEPROMs; they hold secrets out of proportion to their size.
