# Bitstream Reverse Engineering

A dumped, unencrypted bitstream is a sea of configuration bits with no documentation: vendors keep the format secret. Turning those bits back into logic (lookup-table contents, routing, block RAM, hard blocks) needs a database that maps bit positions to fabric features. For several families the open-source FPGA community has built exactly that, as a side effect of writing free toolchains. This page is how to use that work to recover a design. For authorized research.

## The problem

A bitstream is the serialized state of every configurable element in the fabric: each LUT's truth table, every routing switch, every flip-flop mode, the contents of block RAM at power-on. The vendor's tools know the mapping from bit to feature; you do not. Reverse engineering a bitstream means applying a bit-to-feature **database** to recover:

- **LUT contents**, which give you the Boolean logic.
- **Routing**, which gives you the netlist connectivity.
- **Block RAM / ROM initialization**, which can hold constants, microcode, or keys.
- **Hard-block configuration** (DSPs, PLLs, transceivers, I/O standards).

Block RAM init is often the quickest prize: an initialized ROM may contain a lookup table, a state machine encoding, or embedded secrets, and reading it out can be far easier than recovering the full logic.

## The open databases and tools

These projects documented bitstream formats to build free synthesis/place-and-route flows. You can run them in reverse.

- **Project X-Ray (prjxray)**: Xilinx/AMD 7-series (Artix-7, Kintex-7, Spartan-7, Zynq-7000 PL). The database behind the open [SymbiFlow / F4PGA](bitstream-re.md) flow. Maps 7-series config frames to features.
- **Project IceStorm**: Lattice **iCE40**. The original open FPGA flow (Clifford Wolf and others). `icebox` tools convert between bitstream (`.bin`/`.asc`) and a documented ASCII format; iCE40 is often unencrypted, so this is a very practical target.
- **Project Trellis (prjtrellis)**: Lattice **ECP5**. Database and tools (`ecppack`/`ecpunpack`) for the nextpnr flow.
- **Project Mistral**: Intel **Cyclone V**.
- **Project Peppercorn**: Lattice **iCE40 UltraPlus** and related.

For Xilinx 7-series the practical path is: unpack the bitstream to a frame/feature list with prjxray tooling, recover LUTs and routing, then rebuild a netlist you can read or re-synthesize. For iCE40, `iceunpack` plus the IceStorm database gets you a human-readable description directly.

## Workflow

1. **Confirm it is plaintext.** Entropy should not be uniform; vendor headers and sync words should be visible. If not, go back to [encryption attacks](bitstream-encryption.md).
2. **Identify the exact device.** The database is part-specific. Get the family and die from [chip markings](../fundamentals/chip-identification.md) and the bitstream header.
3. **Unpack with the matching database** (prjxray / IceStorm / prjtrellis / Mistral).
4. **Recover features**: dump LUT truth tables and block RAM init first (fast, high value), then rebuild routing into a netlist if you need the full design.
5. **Lift to logic.** Feed the recovered netlist into [Ghidra](../reverse-engineering/ghidra-setup.md)-style analysis is not the fit here; instead use netlist tools (yosys can read/represent netlists) and work up to functional blocks.

## Limits

- **No database, no easy win.** Large modern UltraScale/Stratix parts have no complete open database, so RE there is a research effort, not a tool run.
- **Netlist is not RTL.** You recover gates and routing, not the original readable Verilog/VHDL. Reconstructing intent is manual, similar in spirit to [firmware RE](../reverse-engineering/firmware-workflow.md).
- **Block RAM is the shortcut.** When the goal is a secret or a table rather than the whole design, extracting initialized memory is often enough.

## Defender's view

If the design must stay secret, [encrypt and authenticate the bitstream](bitstream-encryption.md) and lock the device so plaintext cannot be dumped, keep secrets out of block-RAM initialization (derive them at runtime from a [secure element](../attacks/secure-elements.md) or [eFuse-backed](../fundamentals/efuse-otp.md) key instead), and prefer a family whose format is not openly documented if RE resistance matters more than tool freedom.

## Related pages

- [FPGA Security overview](index.md)
- [Bitstream Extraction](bitstream-extraction.md)
- [Bitstream Encryption and Attacks](bitstream-encryption.md)
- [Firmware Analysis](../attacks/firmware-analysis.md)
- [openFPGALoader](../software/openfpgaloader.md)
