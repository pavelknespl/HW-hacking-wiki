# CPU Architectures

The processor at the center of a board decides almost everything downstream: which instruction set your disassembler needs, how the firmware is laid out, what debug interface exists, and whether you are facing a full Linux system or a bare-metal microcontroller. Identify the architecture early, because getting it wrong wastes hours in [Ghidra](../reverse-engineering/ghidra-setup.md).

## The two broad families

Embedded targets split cleanly into two worlds, and the first question to answer is which one you are in.

=== "Application processors (MPUs)"

    Run a full operating system, almost always Linux, sometimes a BSD or a real-time OS.

    - External DRAM and external flash or [eMMC](../protocols/emmc.md).
    - A bootloader (usually [U-Boot](../attacks/bootloader-access.md)) that loads a kernel.
    - A filesystem you can extract and read.
    - Examples: home routers, IP cameras, NAS boxes, set-top boxes, smart hubs.

    These are the most productive targets for [firmware analysis](../attacks/firmware-analysis.md) because the whole userland is sitting in flash.

=== "Microcontrollers (MCUs)"

    Run bare-metal code or a small RTOS, with code and data in on-chip flash and RAM.

    - Often no external memory at all, so dumping means reading internal flash over a debug port.
    - A single firmware blob rather than a filesystem.
    - Examples: door locks, fitness bands, toys, car modules, many IoT sensors.

    Here the prize is the internal flash, and [read-out protection](../protocols/swd.md) is the main obstacle.

## The architectures you will meet

| Architecture | Endianness | Where you see it | [Ghidra](../reverse-engineering/ghidra-setup.md) language ID hint |
| --- | --- | --- | --- |
| **ARM (32-bit)** | Usually little | Vast majority of MCUs (Cortex-M) and many MPUs (Cortex-A) | `ARM:LE:32:*` or `ARM:LE:32:Cortex` |
| **ARM64 / AArch64** | Little | Modern phones, 64-bit SoCs, newer routers | `AARCH64:LE:64:v8A` |
| **MIPS** | Big **or** little | Classic router and camera SoCs (Atheros, Ralink, Broadcom) | `MIPS:BE:32:default` / `MIPS:LE:32:default` |
| **x86 / x86-64** | Little | PC-class embedded, thin clients, some NAS and firewalls | `x86:LE:32:default` / `x86:LE:64:default` |
| **RISC-V** | Little | Newer MCUs, Espressif ESP32-C series, growing fast | `RISCV:LE:32:RV32GC` / `RISCV:LE:64:default` |
| **Xtensa** | Little | Espressif ESP8266 and ESP32 (original) | needs a plugin (see below) |
| **AVR** | Little | Classic Arduino, 8-bit MCUs | `avr8:LE:16:default` (8-bit core, 16-bit addressing) |
| **PowerPC** | Big (usually) | Older networking and automotive gear | `PowerPC:BE:32:default` |
| **SuperH (SH)** | Big or little | Legacy automotive, older appliances | `SuperH:*` |

!!! tip "Endianness is a frequent trap on MIPS"
    MIPS devices exist in both big-endian and little-endian flavors. If your disassembly is garbage but the architecture is right, flip the endianness. The `file` command on an extracted ELF usually tells you, and tools like `binwalk -Y` (capstone-based opcode scan) can guess.

## How to tell what you have

1. **Read the SoC part number.** The main chip's markings lead to a datasheet that states the core. A Qualcomm, MediaTek, Broadcom, Realtek, or Allwinner part number is searchable. See [Chip Identification](chip-identification.md).
2. **Read the boot log.** A [UART console](../protocols/uart.md) prints the CPU model, the architecture, and often the exact SoC during boot. This is the single easiest method when a console is available.
3. **Run `file` on an extracted binary.** Once you have firmware, `file busybox` or `readelf -h` on any ELF states the architecture and endianness outright.
4. **Let [binwalk](../software/binwalk.md) guess.** `binwalk -Y firmware.bin` performs an opcode-based architecture scan on a raw blob.

## Why it matters for your workflow

- **Disassembly.** The architecture and endianness are mandatory inputs to Ghidra, radare2, and objdump. See [Ghidra Setup](../reverse-engineering/ghidra-setup.md).
- **Debug interface.** Cortex-M parts use [SWD](../protocols/swd.md); larger ARM and most MIPS use [JTAG](../protocols/jtag.md). [OpenOCD](../software/openocd.md) config files are chosen per core.
- **Cross-compilation.** Building a tool to run on the target (a static BusyBox, a dropbear, a custom implant for your own device) requires a toolchain that matches the architecture and the ABI, including the endianness and the floating-point convention.
- **Exploitation.** Calling conventions, gadget availability, and cache behavior during glitching all depend on the core.

## Espressif, a common special case

The original ESP8266 and ESP32 use the **Xtensa LX** architecture, which Ghidra does not support out of the box. Install a community Xtensa processor module, or use `esptool.py` to read the flash and the Tensilica tooling to disassemble. The newer **ESP32-C** and **ESP32-H** series switched to **RISC-V**, which Ghidra handles natively. Confirm which chip you have before choosing a disassembler.


## Related pages

- [Ghidra Setup](../reverse-engineering/ghidra-setup.md)
- [Firmware RE Workflow](../reverse-engineering/firmware-workflow.md)
- [Memory Types](memory-types.md)
- [Chip Identification](chip-identification.md)
