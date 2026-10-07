# Tools

Hardware access is only half the job. The other half is the software that drives your adapters, decodes buses, dumps memory, and tears firmware apart, plus the dedicated hardware tools that package a whole attack into one box. This section is a practical reference to both, with the commands you actually run.

## Host software

Tools you run on your workstation to drive an adapter, decode a bus, or process a dump.

<div class="grid cards" markdown>

-   __Serial Consoles__

    `screen`, `minicom`, `tio`, `picocom`: reading and driving a [UART](../protocols/uart.md).

    [:octicons-arrow-right-24: Read](serial-consoles.md)

-   __OpenOCD__

    The bridge from a [JTAG](../protocols/jtag.md)/[SWD](../protocols/swd.md) adapter to GDB and memory dumps.

    [:octicons-arrow-right-24: Read](openocd.md)

-   __flashrom__

    Read and write [SPI](../protocols/spi.md), parallel, and LPC flash chips.

    [:octicons-arrow-right-24: Read](flashrom.md)

-   __Binwalk and Extraction__

    Carve firmware images into their parts.

    [:octicons-arrow-right-24: Read](binwalk.md)

-   __dd and Imaging__

    Byte-exact copies of block devices and partitions.

    [:octicons-arrow-right-24: Read](dd-imaging.md)

-   __UBI and NAND Tools__

    `ubireader`, `nanddump`, and friends for raw NAND.

    [:octicons-arrow-right-24: Read](ubi-nand.md)

-   __Logic Analyzers (sigrok)__

    Capture and decode digital buses with open tooling.

    [:octicons-arrow-right-24: Read](logic-analyzers.md)

-   __esptool__

    Read, inspect fuses, and flash Espressif chips over serial.

    [:octicons-arrow-right-24: Read](esptool.md)

-   __Depthcharge__

    Automate attacks against a U-Boot console.

    [:octicons-arrow-right-24: Read](depthcharge.md)

-   __openFPGALoader__

    Vendor-neutral bitstream and flash loading for FPGAs over JTAG or SPI.

    [:octicons-arrow-right-24: Read](openfpgaloader.md)

-   __UrJTAG__

    Low-level JTAG boundary scan and flash access for odd targets.

    [:octicons-arrow-right-24: Read](urjtag.md)

</div>

## Hardware tools

Self-contained devices that bundle the adapter, the capture, and the attack logic into one piece of kit.

<div class="grid cards" markdown>

-   __ChipWhisperer__

    Integrated glitching and side-channel platform.

    [:octicons-arrow-right-24: Read](chipwhisperer.md)

-   __JTAGulator__

    Brute-force unknown JTAG and UART pinouts.

    [:octicons-arrow-right-24: Read](jtagulator.md)

-   __Proxmark3__

    The reference RFID/NFC read, clone, and attack tool.

    [:octicons-arrow-right-24: Read](proxmark3.md)

-   __Flipper Zero__

    Pocket sub-GHz, NFC, IR, and GPIO multi-tool.

    [:octicons-arrow-right-24: Read](flipper-zero.md)

-   __Tigard__

    FT2232H multi-protocol board: UART, JTAG, SWD, SPI, and I2C with safe voltages.

    [:octicons-arrow-right-24: Read](tigard.md)

-   __Glasgow Interface Explorer__

    Software-defined FPGA multi-tool for exploring unknown boards and buses.

    [:octicons-arrow-right-24: Read](glasgow.md)

</div>

## A rough mapping of task to tool

| Task | Reach for |
| --- | --- |
| Read a UART console | [`tio`](serial-consoles.md), `screen`, `minicom` |
| Halt a CPU, dump internal flash | [OpenOCD](openocd.md) + GDB |
| Dump an external [SPI flash](../attacks/spi-flash-dump.md) | [`flashrom`](flashrom.md) |
| Image a block device ([eMMC](../protocols/emmc.md)/SD) | [`dd`](dd-imaging.md) |
| Carve a firmware image | [`binwalk`](binwalk.md) |
| Rebuild a UBI/NAND filesystem | [`ubireader`, `nanddump`](ubi-nand.md) |
| Decode a captured bus | [sigrok / PulseView](logic-analyzers.md) |
