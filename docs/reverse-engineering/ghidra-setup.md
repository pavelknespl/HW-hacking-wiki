# Ghidra Setup

Ghidra is the free, open-source reverse engineering suite from the NSA, and it is the default disassembler and decompiler for firmware work. Most of the pain people hit with it comes from one thing: loading a binary with the wrong architecture, endianness, or base address. Get those right and the decompiler does the rest. This page is a per-architecture setup reference.

## Install and first run

Download Ghidra from the [official releases](https://github.com/NationalSecurityAgency/ghidra/releases), install a matching JDK (Ghidra states the required version per release), and run `ghidraRun`. Create a project, then import your file with **File > Import File**.

On import, Ghidra guesses a **Language** (the processor spec). For a proper ELF it usually guesses correctly. For a raw firmware blob it cannot, and you must set the language, and often the base address, by hand.

## The Language setting is everything

A Ghidra language ID looks like `processor:endianness:size:variant`, for example `ARM:LE:32:Cortex`. Choosing it correctly is the single most important setup step. The [architectures page](../fundamentals/architectures.md) covers how to identify the processor; here is how to express it in Ghidra.

| Target | Ghidra language ID |
| --- | --- |
| ARM Cortex-M (STM32, nRF, RP2040) | `ARM:LE:32:Cortex` |
| ARM Cortex-A, generic ARMv7 | `ARM:LE:32:v7` |
| ARM, big-endian variant | `ARM:BE:32:v7` |
| ARM64 / AArch64 | `AARCH64:LE:64:v8A` |
| MIPS32, little-endian | `MIPS:LE:32:default` |
| MIPS32, big-endian | `MIPS:BE:32:default` |
| x86 32-bit | `x86:LE:32:default` |
| x86-64 | `x86:LE:64:default` |
| RISC-V 32-bit | `RISCV:LE:32:RV32GC` |
| RISC-V 64-bit | `RISCV:LE:64:default` |
| AVR (8-bit) | `avr8:LE:16:default` |
| PowerPC 32-bit, big-endian | `PowerPC:BE:32:default` |

!!! warning "MIPS endianness is the classic mistake"
    MIPS devices come in both endiannesses. If your disassembly is nonsense but you are sure it is MIPS, switch between `MIPS:BE:32` and `MIPS:LE:32`. Run `binwalk -Y blob.bin` or `file` on an extracted ELF to settle it. The same caution applies to ARM and PowerPC, which also have big-endian variants in the wild.

## Loading a proper ELF

If you extracted an ELF binary (a BusyBox, a daemon, a CGI handler), Ghidra reads its header and sets language, entry point, and section layout automatically. Just import, accept the detected language (verify it against `readelf -h`), and let auto-analysis run. This is the easy case and covers most [firmware analysis](../attacks/firmware-analysis.md) of userland binaries.

## Loading a raw binary blob

Bootloaders, bare-metal MCU firmware, and carved sections usually have **no ELF header**, so Ghidra needs three things from you:

1. **Language.** Choose from the table above. For an MCU, pick the exact core (`ARM:LE:32:Cortex`).
2. **Base address.** The address the code expects to run at. Choose "raw binary" as the format, then in **Options** set the base. Getting this right makes absolute pointers and string references resolve.
3. **Memory map.** For MCUs, define flash and RAM regions so cross-references work.

### Finding the base address

- **MCU vectors.** On Cortex-M, the image starts with a vector table: the first word is the initial stack pointer, the second is the reset handler address. The reset handler's value tells you where the image is mapped (for example values around `0x08000xxx` mean a `0x08000000` flash base on many STM32 parts). Set the base so the reset vector points into your loaded image.
- **String and pointer alignment.** Try a candidate base, run analysis, and check whether pointer-like values resolve to strings and code. If references are consistently off by a constant, adjust the base by that constant.
- **Datasheet memory map.** The chip's documented flash and RAM bases are the ground truth. See [Memory Types](../fundamentals/memory-types.md).

### Cortex-M specifics

- Use `ARM:LE:32:Cortex`, which enables Thumb-2 decoding that plain ARM variants get wrong.
- After loading at the correct base, let Ghidra find the vector table, or create the reset vector function manually at the address in the second word, then disassemble from there as Thumb.
- An SVD file for the exact chip can be imported (via the SVD-Loader extension) to label all the peripheral registers, which makes bare-metal code dramatically more readable.

## Architectures Ghidra does not ship

- **Xtensa** (ESP8266, original ESP32) needs a community processor module. Install it through **File > Install Extensions** or by dropping the module in Ghidra's extensions directory. The newer **ESP32-C/H** parts are RISC-V and work natively. See [Architectures](../fundamentals/architectures.md#espressif-a-common-special-case).
- Other niche cores (some DSPs, older microcontrollers) have community modules of varying quality. Search before assuming a core is unsupported.

## Useful extensions

- **SVD-Loader:** labels MCU peripheral registers from a CMSIS SVD file.
- **ret-sync:** syncs Ghidra with a live debugger (GDB/OpenOCD), so static and dynamic views line up.
- **Ghidra scripts** (Python/Java) automate repetitive labeling; the script manager ships with many.

## After loading: let analysis run, then verify

Run auto-analysis and then sanity-check:

- Do strings appear and get referenced from code? If not, the base address is likely wrong.
- Does the decompiler produce plausible C, with recognizable library calls (`memcpy`, `printf`, `strcmp`)? If it is garbage, recheck language and endianness.
- Can you find `main` or the reset handler and follow control flow? If yes, you are set up correctly.

Only once the setup checks out do you move on to the [firmware workflow](firmware-workflow.md) of actually hunting through the code.
