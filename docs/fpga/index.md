# FPGA Security

An FPGA is a chip full of configurable logic: lookup tables, flip-flops, routing, and hard blocks that you wire together by loading a **bitstream**. That bitstream is the design. For a hardware hacker it is both a prize (it encodes the vendor's intellectual property and sometimes keys) and a route in (if you can change it, you own the logic). This section covers how configuration works, how to get the bitstream off a device, how vendors try to protect it, and what you can actually do with one.

<div class="grid cards" markdown>

-   __Bitstream Extraction__

    Where the bitstream lives and how to capture it: sniffing the config bus, reading the boot flash, and JTAG readback.

    [:octicons-arrow-right-24: Read](bitstream-extraction.md)

-   __Bitstream Encryption and Attacks__

    AES bitstream encryption, key storage in eFuse/BBRAM, and the attacks that have broken it (Starbleed and friends).

    [:octicons-arrow-right-24: Read](bitstream-encryption.md)

-   __Bitstream Reverse Engineering__

    Turning raw configuration bits back into logic with open databases like Project X-Ray, prjtrellis, and IceStorm.

    [:octicons-arrow-right-24: Read](bitstream-re.md)

</div>

## How FPGA configuration works

Most FPGAs you meet are **SRAM-based**. Their configuration is volatile: it is lost at power-off and must be reloaded every boot. That reload is the single most important fact for an attacker, because the bitstream has to come from somewhere you can reach.

- **SRAM FPGAs** (most Xilinx/AMD, Intel/Altera, Lattice ECP5/iCE40): the bitstream is stored in an external [SPI flash](../protocols/spi.md) or pushed by a host CPU at boot. It crosses a bus you can tap.
- **Flash-based FPGAs** (Lattice MachXO, Microchip/Microsemi IGLOO/PolarFire, older Actel): configuration is held on-die in non-volatile cells. No external config flash to sniff, so extraction is harder and leans on readback or invasive methods.
- **Antifuse FPGAs** (older Actel): one-time programmable, genuinely hard to read, rare today.

A typical SRAM FPGA board boots like this: power comes up, the FPGA reads its bitstream over SPI from a dedicated flash chip, configures its fabric, and asserts a DONE signal. Tap that SPI bus or desolder that flash and you have the design. See [Bitstream Extraction](bitstream-extraction.md).

## Why FPGAs are worth attacking

- **IP recovery.** The bitstream *is* the proprietary design. Competitors and researchers want it back as a netlist.
- **Key and secret recovery.** Designs embed keys, and the FPGA's own key storage (BBRAM/eFuse) is a target in its own right.
- **Logic modification.** Change the bitstream and you change behavior: disable a check, add a backdoor, re-enable a disabled [JTAG](../protocols/jtag.md) or debug path. This is a hardware equivalent of [firmware modification](../attacks/firmware-modification.md).
- **Cloning.** A dumped, unencrypted bitstream can be loaded onto an identical FPGA to clone the product.

## The workflow

1. **Identify the FPGA and its config source.** Read the markings ([chip identification](../fundamentals/chip-identification.md)), find the config [SPI flash](../protocols/spi.md), and locate the [JTAG](../protocols/jtag.md) TAP.
2. **Extract.** Sniff the config load, read the boot flash, or try [JTAG readback](bitstream-extraction.md). See which the board allows.
3. **Check for encryption.** A high-entropy bitstream that will not parse is encrypted. Move to [encryption attacks](bitstream-encryption.md).
4. **Reverse or modify.** Load the bits into the matching [open bitstream database](bitstream-re.md) to recover logic, or patch and reload.

!!! info "Vendor terms you will see"
    Xilinx/AMD call the config memory **BBRAM** (battery-backed RAM) and **eFUSE**; the readback path uses the **ICAP/SelectMAP/JTAG** interfaces. Intel/Altera use **CvP** and **Partial Reconfiguration**. Lattice calls its flash **NVCM**. The concepts map across vendors even when the names do not.

## Related pages

- [SPI](../protocols/spi.md)
- [JTAG](../protocols/jtag.md)
- [SPI Flash Dumping](../attacks/spi-flash-dump.md)
- [eFuse and OTP Memory](../fundamentals/efuse-otp.md)
- [Side-Channel Attacks](../attacks/side-channel.md)
- [openFPGALoader](../software/openfpgaloader.md)
