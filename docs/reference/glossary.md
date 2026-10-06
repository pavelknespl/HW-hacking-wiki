# Glossary

Plain definitions of the terms used across the wiki. Where a full page exists, the term links to it.

## A

**AArch64 / ARM64**
: The 64-bit [ARM architecture](../fundamentals/architectures.md), found in modern phones and SoCs.

**ASK / OOK**
: Amplitude-Shift Keying and On-Off Keying, simple radio modulations common in cheap sub-GHz devices. See [SDR Basics](../radio/sdr-basics.md).

## B

**Bad block**
: A region of [NAND flash](../fundamentals/memory-types.md) that is unreliable and marked unusable. The OS or controller skips it, which is why a raw NAND read is not a linear filesystem.

**Baud rate**
: The symbol rate of a [UART](../protocols/uart.md) link in bits per second. Both sides must agree. 115200 is the most common.

**Bit-banging**
: Driving a bus by toggling GPIO pins in software rather than using a hardware controller. Universal but slow. See [GPIO Control](../scripting/gpio-control.md).

**Bootloader**
: Early code that initializes hardware and loads the OS. On embedded Linux this is usually [U-Boot](../attacks/bootloader-access.md).

**Boundary scan**
: The original purpose of [JTAG](../protocols/jtag.md): testing board interconnections by shifting values through chip I/O.

## C

**Chain of trust**
: A [secure boot](../attacks/bootloader-bypass.md) design where each stage cryptographically verifies the next, rooted in immutable silicon.

**Chip select (CS)**
: The [SPI](../protocols/spi.md) line that activates one slave on the bus. Active low (`CS#`).

**CPA (Correlation Power Analysis)**
: A [side-channel](../attacks/side-channel.md) technique that correlates many power traces against key hypotheses to recover a key byte by byte.

**Crowbar**
: A [voltage-glitch](../attacks/fault-injection.md) method that briefly shorts the power rail with a fast transistor.

## D

**DPA (Differential Power Analysis)**
: The statistical [side-channel](../attacks/side-channel.md) technique that CPA refines.

**dd**
: The byte-copy tool used for [imaging block devices](../software/dd-imaging.md) and carving regions.

## E

**ECC**
: Error-Correcting Code stored alongside [NAND](../fundamentals/memory-types.md) pages to fix bit flips.

**[eMMC](../protocols/emmc.md)**
: Managed [NAND](../protocols/emmc.md) with an on-chip controller presenting a clean block device.

**Endianness**
: Byte order. Matters when choosing a [Ghidra language](../reverse-engineering/ghidra-setup.md); MIPS exists in both.

**Entropy**
: A measure of randomness used to tell compressed data from [encrypted data](../software/binwalk.md#entropy-analysis-spotting-encryption-and-compression) in a firmware image.

## F

**[Fault injection](../attacks/fault-injection.md)**
: Deliberately disturbing a chip (voltage, clock, EM, laser) to cause a useful malfunction. See [Fault Injection](../attacks/fault-injection.md).

**[flashrom](../software/flashrom.md)**
: The tool that reads and writes [flash chips](../software/flashrom.md) through a programmer.

**FCC ID**
: An identifier on US-sold devices that unlocks internal photos and reports in the [FCC database](../attacks/recon.md).

## G

**[Ghidra](../reverse-engineering/ghidra-setup.md)**
: The free NSA reverse engineering suite. See [Ghidra Setup](../reverse-engineering/ghidra-setup.md).

**Glitching**
: Informal term for [fault injection](../attacks/fault-injection.md), especially voltage and clock faults.

**GPIO**
: General-Purpose Input/Output pins you can [drive and read from code](../scripting/gpio-control.md).

## I

**[I2C](../protocols/i2c.md)**
: A two-wire bus (`SDA`, `SCL`) for low-speed peripherals and [EEPROMs](../protocols/i2c.md).

**IDCODE**
: An identifier a [JTAG](../protocols/jtag.md) TAP returns, confirming a valid debug port and identifying the part.

**IQ samples**
: The complex-number stream an [SDR](../radio/sdr-basics.md) produces, representing a slice of spectrum.

## J

**[JTAG](../protocols/jtag.md)**
: A debug and test interface giving [full CPU control](../protocols/jtag.md): halt, read/write memory, dump flash.

**JTAGulator**
: A tool that [identifies JTAG/UART pinouts](../fundamentals/identifying-pins.md) by probing combinations.

## L

**[Level shifter](../fundamentals/level-shifting.md)**
: A circuit that bridges two [logic voltages](../fundamentals/level-shifting.md) safely.

**[Logic analyzer](../software/logic-analyzers.md)**
: A tool that captures and [decodes digital buses](../software/logic-analyzers.md).

## M

**MISO / MOSI**
: [SPI](../protocols/spi.md) data lines: Master In Slave Out, and Master Out Slave In.

**MTD**
: Memory Technology Device, the Linux subsystem exposing raw flash as `/dev/mtdN`. See [NAND tools](../software/ubi-nand.md).

## N

**[NAND](../fundamentals/memory-types.md) / NOR flash**
: The two [flash types](../fundamentals/memory-types.md). NOR is small, random-access, easy to dump; NAND is large, block-based, needs ECC and bad-block handling.

**nanddump**
: The on-device tool that reads [NAND with ECC applied](../software/ubi-nand.md), producing clean data.

## O

**OOB (Out-Of-Band)**
: Spare bytes after each [NAND](../fundamentals/memory-types.md) page, holding ECC and metadata.

**[OpenOCD](../software/openocd.md)**
: The open-source bridge from a [JTAG/SWD adapter](../software/openocd.md) to GDB and memory dumps.

## P

**PMIC**
: Power Management IC, which generates a board's supply rails. See [PCB Recon](../fundamentals/pcb-recon.md).

## R

**[Read-out protection](../protocols/swd.md) (RDP / CRP / APPROTECT)**
: A lock bit that stops a [debug port reading internal flash](../protocols/swd.md#read-out-protection-again). A target of [fault injection](../attacks/fault-injection.md).

**Replay attack**
: Re-sending a recorded [radio transmission](../radio/rf-attacks.md) to repeat an action, effective against fixed-code devices.

**Rolling code**
: A remote scheme where each transmission differs, defeating naive [replay](../radio/rf-attacks.md).

**Root of trust**
: The immutable base of a [secure boot](../attacks/bootloader-bypass.md) chain, typically a key in on-chip fuses.

## S

**[Secure boot](../attacks/bootloader-bypass.md)**
: A [verified boot chain](../attacks/bootloader-bypass.md) that refuses to run unauthorized code.

**[Side-channel](../attacks/side-channel.md)**
: Information leaked physically (power, timing, EM) that reveals [secrets](../attacks/side-channel.md).

**SoC**
: System on Chip, the main processor integrating CPU, memory controller, and peripherals.

**SPA (Simple Power Analysis)**
: Reading a secret from the shape of one or a few [power traces](../attacks/side-channel.md).

**[SPI](../protocols/spi.md)**
: A fast four-wire bus connecting most [external flash](../protocols/spi.md).

**[SquashFS](../software/binwalk.md)**
: A compressed read-only filesystem common in firmware; vendor variants often need [sasquatch](../software/binwalk.md).

**[SWD](../protocols/swd.md)**
: ARM's [two-wire debug interface](../protocols/swd.md), standard on Cortex-M.

## T

**TAP (Test Access Port)**
: The state machine [JTAG](../protocols/jtag.md) drives, per chip.

**Thumb / Thumb-2**
: Compact ARM instruction encodings; Cortex-M uses Thumb-2, which needs the right [Ghidra variant](../reverse-engineering/ghidra-setup.md#cortex-m-specifics).

## U

**[U-Boot](../attacks/bootloader-access.md)**
: The dominant embedded Linux [bootloader](../attacks/bootloader-access.md).

**[UART](../protocols/uart.md)**
: The asynchronous [serial interface](../protocols/uart.md) behind most device consoles.

**UBI / UBIFS**
: The volume layer and filesystem used on modern [NAND](../software/ubi-nand.md).

**ubireader**
: Tooling that extracts [UBI/UBIFS from a raw dump](../software/ubi-nand.md).

## V

**Verified boot**
: Another name for [secure boot](../attacks/bootloader-bypass.md).

## X

**Xtensa**
: The [architecture](../fundamentals/architectures.md#espressif-a-common-special-case) of the ESP8266 and original ESP32; needs a Ghidra plugin.
