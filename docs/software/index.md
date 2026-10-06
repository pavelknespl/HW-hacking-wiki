# Software Tools

Hardware access is only half the job. The other half is the software that drives your adapters, decodes buses, dumps memory, and tears firmware apart. This section is a practical reference to the toolchain, with the commands you actually run.

<div class="grid cards" markdown>

-   __Serial Consoles__

    `screen`, `minicom`, `tio`, `picocom`: reading and driving a UART.

    [:octicons-arrow-right-24: Read](serial-consoles.md)

-   __OpenOCD__

    The bridge from a JTAG/SWD adapter to GDB and memory dumps.

    [:octicons-arrow-right-24: Read](openocd.md)

-   __flashrom__

    Read and write SPI, parallel, and LPC flash chips.

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

</div>

## A rough mapping of task to tool

| Task | Reach for |
| --- | --- |
| Read a UART console | [`tio`](serial-consoles.md), `screen`, `minicom` |
| Halt a CPU, dump internal flash | [OpenOCD](openocd.md) + GDB |
| Dump an external SPI flash | [`flashrom`](flashrom.md) |
| Image a block device (eMMC/SD) | [`dd`](dd-imaging.md) |
| Carve a firmware image | [`binwalk`](binwalk.md) |
| Rebuild a UBI/NAND filesystem | [`ubireader`, `nanddump`](ubi-nand.md) |
| Decode a captured bus | [sigrok / PulseView](logic-analyzers.md) |
