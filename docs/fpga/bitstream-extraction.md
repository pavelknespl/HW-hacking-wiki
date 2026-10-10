# Bitstream Extraction

The bitstream is the FPGA's configuration. On an SRAM-based FPGA it is reloaded every boot from somewhere off-chip, which gives you three ways in: catch it on the wire as it loads, read it from the flash it lives in, or ask the FPGA to hand it back over [JTAG](../protocols/jtag.md). This page covers all three. For authorized testing of your own hardware.

## Where the bitstream lives

Before extracting, work out the configuration mode. It is set by MODE pins on the FPGA and tells you where the bits come from:

- **Master SPI**: the FPGA reads its own bitstream from a dedicated [SPI flash](../protocols/spi.md). The most common case on standalone boards.
- **Slave SPI / SelectMAP**: a host CPU or microcontroller pushes the bitstream into the FPGA at boot. The bits cross whatever bus links them.
- **JTAG**: configuration is loaded over the [JTAG](../protocols/jtag.md) chain, often during development or by an on-board programmer.
- **BPI / parallel**: older designs read from parallel NOR flash over an address/data bus.

Find the config flash by following the SPI lines from the FPGA's config pins, or just look for a small [SPI flash](../protocols/spi.md) chip near the FPGA. See [chip identification](../fundamentals/chip-identification.md) and [PCB recon](../fundamentals/pcb-recon.md).

## Method 1: read the configuration flash

This is usually the easiest win. The config flash is an ordinary SPI NOR chip, so everything on the [SPI Flash Dumping](../attacks/spi-flash-dump.md) page applies: clip onto it with a [SOIC clip](../fundamentals/soldering-rework.md) and read it with [flashrom](../software/flashrom.md), or desolder and read in a socket.

```bash
flashrom -p ch341a_spi -r config_flash.bin
```

The contents are the bitstream, often with a small header and sometimes a multiboot layout (a golden image plus an update image). If the result parses in the vendor tools or an [open database](bitstream-re.md), it is unencrypted and you are done. If it is uniformly high-entropy, it is probably [encrypted](bitstream-encryption.md).

## Method 2: sniff the configuration load

If you cannot pull the flash, capture the bitstream as it is loaded at power-on. Put a [logic analyzer](../software/logic-analyzers.md) on the SPI lines between the config flash and the FPGA, power-cycle the board, and decode the SPI read. The bitstream is the payload of the read commands.

- Use a fast enough analyzer: config SPI can run tens of MHz, so sample at several times that. See [logic analyzers](../software/logic-analyzers.md).
- A [Glasgow](../software/glasgow.md) or [Tigard](../software/tigard.md) works for slower loads; a dedicated fast analyzer is better for high-speed config.
- This also works for Slave SPI / SelectMAP, where a host pushes the bits: tap the same way.

Sniffing captures the bitstream in whatever form crosses the bus. If the design uses [encryption](bitstream-encryption.md), you capture ciphertext, not plaintext.

## Method 3: JTAG readback

Many FPGAs can read their current configuration back out through [JTAG](../protocols/jtag.md) (or SelectMAP/ICAP). If readback is not disabled, you can dump the live configuration without touching the flash:

- Xilinx/AMD: the `READBACK` path via JTAG, driven by vendor tools or `openocd`/custom TAP sequences. 7-series parts expose readback registers; this is the same path the [Starbleed attack](bitstream-encryption.md) abuses.
- Use [openFPGALoader](../software/openfpgaloader.md) or [UrJTAG](../software/urjtag.md) to talk to the TAP once you have found it with a [JTAGulator](../software/jtagulator.md) or from the datasheet.

```bash
openFPGALoader --detect                       # enumerate the JTAG chain / IDCODE
openFPGALoader --dump-flash -o config.bin     # read the attached config flash via JTAG
# vendor readback of the live fabric (7-series) uses the READBACK CAPTURE path
# driven by openocd/custom TAP sequences; the result is the running configuration.
```

Readback is frequently **disabled** by a security bit or by setting the device to encrypted-only mode, so treat it as a path to try rather than rely on. When it is open, it is the cleanest dump because it reflects exactly what the fabric is running.

## After extraction

- **Parses cleanly** → move to [Bitstream Reverse Engineering](bitstream-re.md) to recover logic, or patch and reload with [openFPGALoader](../software/openfpgaloader.md).
- **High-entropy, will not parse** → [encrypted](bitstream-encryption.md). Identify the part and look for a known key-recovery attack.
- **Partial / multiboot** → separate the golden and update images; headers mark the boundaries.

## Defender's view

Store the design encrypted and lock the device to encrypted bitstreams only, disable [JTAG](../protocols/jtag.md) readback and configuration readback, blow the relevant [security eFuses](../fundamentals/efuse-otp.md), and enable per-boot authentication (HMAC/RSA) so a sniffed or modified bitstream cannot be replayed or altered. Assume the config flash is readable and design as if the attacker already has the ciphertext.

## Related pages

- [FPGA Security overview](index.md)
- [Bitstream Encryption and Attacks](bitstream-encryption.md)
- [SPI Flash Dumping](../attacks/spi-flash-dump.md)
- [SPI](../protocols/spi.md)
- [JTAG](../protocols/jtag.md)
- [openFPGALoader](../software/openfpgaloader.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
