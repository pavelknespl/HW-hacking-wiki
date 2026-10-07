# UrJTAG

UrJTAG is a classic open-source tool for low-level [JTAG](../protocols/jtag.md) work: boundary scan, TAP interrogation, and reading or writing memory and flash through a chip's JTAG chain. Where [OpenOCD](openocd.md) is oriented toward debugging CPUs, UrJTAG lives closer to the raw IEEE 1149.1 layer, which makes it handy for odd or poorly supported targets, for walking an unknown scan chain, and for driving device pins directly through boundary-scan cells.

## What it does

- **Enumerate the chain.** Detect devices on the TAP by their IDCODE and match them against its BSDL-derived part database.
- **Boundary scan.** Read and set individual device pins through the boundary-scan register, which lets you probe a board's wiring or bit-bang an [SPI flash](../protocols/spi.md) that hangs off an FPGA or CPU's pins.
- **Memory and flash access.** On supported parts, read and program attached parallel flash over JTAG.

## A session

```text
$ jtag
jtag> cable ft2232
jtag> detect
IR length: ...
Chain length: 1
Device Id: ... (Manufacturer ...)
jtag> instruction EXTEST
jtag> ...
```

`detect` is the first useful step: it confirms the cable works and prints the IDCODE of each device on the chain, which tells you what you are connected to before anything else. Find the TAP pins first with a [JTAGulator](jtagulator.md) if the header is unlabeled.

## Where it fits

Reach for UrJTAG when you need boundary-scan access or are working a target that modern debuggers do not support cleanly, especially older CPUs and [FPGAs](../fpga/index.md). For mainstream ARM debugging, [OpenOCD](openocd.md) is usually the better fit, and for FPGA bitstream loading [openFPGALoader](openfpgaloader.md) is more convenient. UrJTAG's strength is the raw boundary-scan layer the others abstract away. It pairs with any FTDI cable such as [Tigard](tigard.md).

## Related pages

- [JTAG](../protocols/jtag.md)
- [OpenOCD](openocd.md)
- [openFPGALoader](openfpgaloader.md)
- [JTAGulator](jtagulator.md)
- [Tigard](tigard.md)
- [SPI](../protocols/spi.md)
