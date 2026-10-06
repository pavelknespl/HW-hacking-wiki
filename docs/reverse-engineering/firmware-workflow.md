# Firmware RE Workflow

With [Ghidra set up correctly](ghidra-setup.md) and a binary loaded, the question becomes how to actually read it productively rather than wandering through thousands of functions. This page is a practical workflow for the two common cases: a Linux userland binary, and a bare-metal firmware blob.

## Before you open the disassembler

A lot of understanding is available more cheaply than reading assembly:

```bash
file bin/httpd                 # architecture, dynamic vs static
strings -n 8 bin/httpd | less  # URLs, formats, error messages, command strings
readelf -d bin/httpd           # shared library dependencies
nm -D bin/httpd                # imported/exported symbols if present
```

Strings alone often reveal the structure of a program: the format strings passed to `system`, the names of config files, debug messages that map code paths, and sometimes secrets. Read the strings before the code.

## Case 1: a Linux userland binary

This covers most router/camera/IoT work: a CGI handler, a web server, a daemon.

### Orient yourself

- Let Ghidra's auto-analysis finish. For a dynamically linked binary it labels library calls, which are your signposts.
- Find `main` (or the entry for a CGI, which may be driven by environment variables rather than `argv`).
- Identify the **input sources**: network reads (`recv`, `read` on a socket), CGI environment (`getenv("QUERY_STRING")`), and file reads.

### Follow the data to dangerous sinks

Vulnerability hunting is largely tracing attacker-controlled input to a dangerous function. Search for calls to:

| Sink | Bug class |
| --- | --- |
| `system`, `popen`, `exec*` | Command injection |
| `strcpy`, `strcat`, `sprintf`, `gets` | Buffer overflow |
| `memcpy` with attacker length | Overflow / out-of-bounds |
| format functions with user format string | Format string |
| `open`/`fopen` with user path | Path traversal |

In Ghidra, use **Search > For Strings** and the symbol tree to find these calls, then work backward (Ghidra's cross-references, `Ctrl-Shift-F` on a function) to see whether user input reaches them unchecked. A `QUERY_STRING` that flows into a `sprintf` that flows into `system` is a textbook finding.

### Locate authentication and secrets

- Search strings for `password`, `login`, `token`, and config paths; follow their cross-references to the check that uses them.
- A comparison against a hardcoded string is a hardcoded credential. A [non-constant-time compare](../attacks/side-channel.md#timing-attacks-the-simplest-case) of a secret is itself a bug.
- Decryption routines are recognizable by their use of standard constants (AES S-box, SHA init values); Ghidra and plugins can flag crypto constants.

## Case 2: a bare-metal firmware blob

An MCU image with no OS. Harder, because there are no library symbols to anchor on.

### Establish the entry and the map

- Confirm the [base address](ghidra-setup.md#finding-the-base-address) so pointers resolve.
- On Cortex-M, define the vector table: the reset handler is the real entry point, and the other vectors (interrupt handlers) are labeled functions worth naming.
- Import an **SVD** file so peripheral register accesses read as `UART->DR` rather than `*(0x40011004)`. This single step makes bare-metal code legible.

### Work from the peripherals inward

Bare-metal code is driven by hardware. Find where it touches peripherals and reason outward:

- Writes to a UART data register mark logging and command handling.
- Reads from a crypto or flash-controller register mark the security-relevant code.
- GPIO accesses map to physical pins (LEDs, resets, chip selects) you can correlate with the board.

### Find the check that matters

For a [glitching](../attacks/fault-injection.md) or [bypass](../attacks/bootloader-bypass.md) campaign, the goal is usually one specific instruction: the signature comparison, the read-protection check, the PIN compare. Locate it statically here so the physical attack has a target and a trigger.

## Make it readable as you go

Reverse engineering is note-taking in disguise. In Ghidra:

- **Rename** functions and variables the moment you understand them (`FUN_0040ab12` becomes `parse_query_string`).
- **Retype** variables and set function signatures so the decompiler improves.
- **Comment** the non-obvious, especially the reasoning behind a conclusion.
- **Bookmark** the sinks and the security checks.

Each name you assign makes the surrounding decompilation clearer, so understanding compounds.

## Close the loop with dynamic analysis

Static reading has blind spots. Confirm hypotheses by running the code:

- **Emulate userland binaries** with QEMU user mode: `qemu-mipsel -L ./rootfs ./bin/httpd`, optionally under `gdb` for breakpoints. Full-system tools like **FirmAE** boot the whole image so you can exercise the web UI. See [Firmware Analysis](../attacks/firmware-analysis.md#emulation-for-dynamic-analysis).
- **Debug on hardware** over [JTAG/SWD](../protocols/jtag.md) with [OpenOCD](../software/openocd.md) and GDB, using `ret-sync` to keep Ghidra aligned with the live state. Set a breakpoint on the function you identified and watch the real arguments.

Static analysis tells you where to look; dynamic analysis confirms what actually happens. Alternating between the two is how firmware gets fully understood.
