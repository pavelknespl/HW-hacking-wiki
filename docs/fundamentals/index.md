# Fundamentals

Before you can attack a device you have to read it. This section covers the groundwork: what kind of processor you are facing, how to decode chip markings, how to make sense of a bare circuit board, how to find the pins that matter, and how to connect to them without releasing the magic smoke.

<div class="grid cards" markdown>

-   __CPU Architectures__

    ARM, MIPS, x86, RISC-V, Xtensa, AVR, and what each means for your tooling.

    [:octicons-arrow-right-24: Read](architectures.md)

-   __Chip Identification__

    Turn the cryptic lines of text on a chip into a datasheet and a pinout.

    [:octicons-arrow-right-24: Read](chip-identification.md)

-   __PCB Recon__

    Read a board like a map: power, ground, buses, test points, and headers.

    [:octicons-arrow-right-24: Read](pcb-recon.md)

-   __Finding and Identifying Pins__

    Locate [UART](../protocols/uart.md), [JTAG](../protocols/jtag.md), and memory pins with a multimeter and a [logic analyzer](../software/logic-analyzers.md).

    [:octicons-arrow-right-24: Read](identifying-pins.md)

-   __Logic Levels and Level Shifting__

    Why 3.3 V, 1.8 V, and 5 V matter, and how to bridge them safely.

    [:octicons-arrow-right-24: Read](level-shifting.md)

-   __Memory Types__

    NOR, [NAND](memory-types.md), [eMMC](../protocols/emmc.md), EEPROM, and where firmware actually lives.

    [:octicons-arrow-right-24: Read](memory-types.md)

-   __eFuse and OTP Memory__

    One-time-programmable on-die memory: keys, lock bits, and the anchor of the trust chain.

    [:octicons-arrow-right-24: Read](efuse-otp.md)

-   __Soldering and Rework__

    Fine wires, hot air, chip-off, and reading BGA with adapters.

    [:octicons-arrow-right-24: Read](soldering-rework.md)

</div>
