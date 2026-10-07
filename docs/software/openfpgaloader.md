# openFPGALoader

`openFPGALoader` is a universal, open-source tool for loading bitstreams into FPGAs and the flash chips beside them. It speaks to a wide range of cables and boards (FTDI-based JTAG, dedicated programmers, even a Raspberry Pi's GPIO) and supports Xilinx/AMD, Intel/Altera, Lattice, Gowin, and more. For a hardware hacker it is the vendor-neutral way to talk to an [FPGA](../fpga/index.md)'s [JTAG](../protocols/jtag.md) TAP or its config [SPI flash](../protocols/spi.md) without installing a gigabyte of proprietary tooling.

## Detect the device

Start by confirming the cable sees a TAP and identifying what is on the chain:

```bash
openFPGALoader --detect                 # list devices on the default cable
openFPGALoader -c ft2232 --detect       # specify the cable explicitly
```

This reads the JTAG IDCODE and prints the matched part. If you found the TAP pins with a [JTAGulator](jtagulator.md), wire them to any supported [FTDI interface](tigard.md) and point `openFPGALoader` at it.

## Load a bitstream

```bash
# load into SRAM (volatile, lost at power-off)
openFPGALoader -c ft2232 design.bit

# write to the configuration SPI flash (persistent)
openFPGALoader -c ft2232 -f design.bit
```

Loading to SRAM is the fast path for testing; writing with `-f` targets the external [config flash](../fpga/bitstream-extraction.md) so the design survives a reboot. On a device with no [bitstream authentication](../fpga/bitstream-encryption.md), writing a modified bitstream is a direct way to change the logic.

## Read back and dump flash

```bash
# dump the configuration flash to a file
openFPGALoader -c ft2232 --dump-flash -o config_flash.bin --file-size 0x400000
```

Dumping the config flash gives you the stored bitstream for [extraction](../fpga/bitstream-extraction.md) and [reverse engineering](../fpga/bitstream-re.md). Whether JTAG *readback* of the live fabric is allowed depends on the part and its security bits.

## Where it fits

Reach for `openFPGALoader` when you want one tool across vendors, when the board exposes JTAG but you do not want the vendor IDE, or when you are programming from an SBC in the field. For raw flash work the chip is still an ordinary SPI NOR, so [flashrom](flashrom.md) and a [SOIC clip](../attacks/spi-flash-dump.md) remain an option when JTAG is locked. For low-level boundary-scan beyond loading, see [UrJTAG](urjtag.md).

## Related pages

- [FPGA Security overview](../fpga/index.md)
- [Bitstream Extraction](../fpga/bitstream-extraction.md)
- [Bitstream Reverse Engineering](../fpga/bitstream-re.md)
- [JTAG](../protocols/jtag.md)
- [SPI](../protocols/spi.md)
- [flashrom](flashrom.md)
- [JTAGulator](jtagulator.md)
- [UrJTAG](urjtag.md)
