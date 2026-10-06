# Reverse Engineering

Extraction gets you the bytes; reverse engineering turns them into understanding. This section covers setting up [Ghidra](ghidra-setup.md) correctly for each embedded architecture, which is the step people most often get wrong, and a practical workflow for going from a firmware image to located vulnerabilities and keys.

<div class="grid cards" markdown>

-   __Ghidra Setup__

    Per-architecture language settings, loading raw blobs, and the extensions you need.

    [:octicons-arrow-right-24: Read](ghidra-setup.md)

-   __Firmware RE Workflow__

    From image to annotated functions: how to actually read embedded code.

    [:octicons-arrow-right-24: Read](firmware-workflow.md)

-   __Emulation and Dynamic Analysis__

    QEMU, FirmAE, EMBA, and Qiling: run the firmware, do not just read it.

    [:octicons-arrow-right-24: Read](emulation.md)

</div>

## Tools worth knowing

- **Ghidra** is the free, capable default, with a decompiler for many architectures and an extensible processor list. The focus of this section.
- **radare2 / Cutter** is a strong open-source alternative, scriptable and fast.
- **IDA Pro** is the commercial standard with the most mature decompiler, if you have access.
- **objdump, readelf, nm** cover quick questions without a full project.

Pick one disassembler and learn it deeply; the concepts transfer. The architecture-specific setup below applies to all of them.
