---
hide:
  - navigation
  - toc
---

<div class="hero" markdown>

# Hardware Hacking, written down

A practical, vendor-neutral reference for getting into embedded devices: reading the board, talking to the chips, pulling the firmware, and understanding what you find. Every page is a plain Markdown file, so the whole wiki is yours to fork, correct, and extend.

</div>

This site collects the workflow that repeats across almost every embedded target, from a cheap IP camera to an industrial gateway: identify the hardware, locate a debug or memory interface, get code or data off the device, and analyze it. The goal is depth without ceremony. Expect working commands, real pin names, and the reasoning behind each step rather than a list of magic incantations.

!!! warning "Scope and ethics"
    Everything here is written for work you are authorized to do: your own devices, hardware you have written permission to test, CTF targets, and research. Hardware attacks are often destructive and almost always void warranties. Do not point these techniques at property you do not own or have permission to test.

## Start here

<div class="grid cards" markdown>

-   :material-flag-checkered:{ .lg .middle } __New to this?__

    ---

    Understand what hardware hacking actually involves, build a lab that fits your budget, and learn the repeatable methodology.

    [:octicons-arrow-right-24: Getting Started](getting-started/index.md)

-   :material-chip:{ .lg .middle } __Know your target__

    ---

    CPU architectures, chip markings, reading a PCB, and finding the pins that matter before you solder anything.

    [:octicons-arrow-right-24: Fundamentals](fundamentals/index.md)

-   :material-serial-port:{ .lg .middle } __Talk to the chips__

    ---

    UART, JTAG, SWD, I2C, and SPI explained from the wire up, with wiring and the commands to probe each one.

    [:octicons-arrow-right-24: Protocols](protocols/index.md)

-   :material-console:{ .lg .middle } __The software side__

    ---

    `screen`, `minicom`, `tio`, `flashrom`, `binwalk`, `dd`, OpenOCD, sigrok, and the rest of the toolchain.

    [:octicons-arrow-right-24: Software Tools](software/index.md)

-   :material-target:{ .lg .middle } __Attacks and techniques__

    ---

    Firmware extraction, flash dumping, bootloader access and bypass, fault injection, and side-channel analysis.

    [:octicons-arrow-right-24: Attacks](attacks/index.md)

-   :material-radar:{ .lg .middle } __Reverse and radio__

    ---

    Ghidra set up per architecture, a firmware analysis workflow, and an introduction to RF and SDR.

    [:octicons-arrow-right-24: Reverse Engineering](reverse-engineering/index.md)

</div>

## The core loop

Most assessments follow the same arc regardless of the device in front of you:

1. **Recon.** Open the case, photograph the board, read the chip markings, and pull datasheets. Decide what you are looking at before you touch it. See [Reconnaissance](attacks/recon.md).
2. **Find an interface.** Hunt for a serial console, a debug port, or an external memory chip. See [Finding and Identifying Pins](fundamentals/identifying-pins.md).
3. **Get in or get the data.** Drop to a shell over [UART](protocols/uart.md), halt the CPU over [JTAG](protocols/jtag.md), interrupt [U-Boot](attacks/bootloader-access.md), or dump the [SPI flash](attacks/spi-flash-dump.md) directly.
4. **Analyze.** Carve the image with [binwalk](software/binwalk.md), rebuild filesystems, and load binaries into [Ghidra](reverse-engineering/ghidra-setup.md).
5. **Go deeper when needed.** When software paths are locked down, fall back to [fault injection](attacks/fault-injection.md) or [side-channel analysis](attacks/side-channel.md).

## About this wiki

The structure and much of the practical framing are inspired by community resources such as [hardbreak.wiki](https://www.hardbreak.wiki/). Where those sites are excellent, treat this as a parallel notebook with its own emphasis on explaining *why* each step works. Corrections and additions are welcome through the [repository](https://github.com/pavelknespl/hwhack.info).
