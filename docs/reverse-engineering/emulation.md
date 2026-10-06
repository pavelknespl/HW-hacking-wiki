# Firmware Emulation and Dynamic Analysis

Static reading in [Ghidra](ghidra-setup.md) tells you what code *could* do. Running it tells you what it *does*. Emulation is the bridge between pulling an image apart with [binwalk](../software/binwalk.md) and understanding it fully: you execute the firmware (or pieces of it) on your workstation, set breakpoints, feed it input, and watch it behave, without needing the physical device powered on your bench. This page fills the gap in the dump then analyze workflow with the dynamic step.

## Why emulate

- **Confirm hypotheses** from static analysis by actually running the code path.
- **Exercise network services** (the web UI, a daemon) to find and trigger bugs without the hardware.
- **Fuzz** the firmware's parsers and services at speed.
- **Debug** with breakpoints and memory inspection where static reading stalls.

## The levels of emulation

=== "User-mode (single binary)"

    Run one cross-architecture Linux binary under QEMU user mode, borrowing your host kernel. Fast and simple, ideal for a specific daemon or CGI.

    ```bash
    # run a MIPS little-endian binary with its own libraries
    qemu-mipsel -L ./squashfs-root ./usr/sbin/httpd
    # under a debugger on port 1234
    qemu-mipsel -g 1234 -L ./squashfs-root ./usr/sbin/httpd
    ```

    Attach `gdb-multiarch` to `:1234` and sync with Ghidra via `ret-sync`. The catch: hardware-specific files (`/dev`, `/proc` entries, NVRAM) the binary expects may be missing, so you stub them (a common trick is an `LD_PRELOAD` shim that fakes `nvram_get`/`nvram_set` on routers).

=== "Full-system (whole image)"

    Boot the entire firmware (kernel plus root filesystem) in QEMU system mode, so the real init, services, and networking come up. Heavier, but it reproduces the device closely and lets you browse to the web interface locally.

    This is hard to set up by hand (right kernel, device tree, machine type, peripherals), which is why the frameworks below exist.

## The frameworks

- **FirmAE** and the older **Firmadyne**: automate full-system emulation of Linux router/camera/IoT firmware. You point them at an image and they extract it, pick a kernel, fake the needed NVRAM and network interfaces, and boot it so the web UI is reachable on a local IP. The go-to for "make this router's firmware run so I can poke its web server."

    ```bash
    ./run.sh -r brand image.bin        # FirmAE: extract, infer, and boot
    # then browse to the emulated device's IP and test the web interface
    ```

- **EMBA**: an automated firmware **analysis** pipeline. It does not just emulate; it extracts, inventories components and versions, flags known CVEs, finds weak binaries and secrets, and can drive emulation for dynamic checks. Excellent for a fast, thorough first pass over an image and for reporting.

    ```bash
    ./emba -l ./logs -f ./firmware.bin -p ./scan-profiles/default-scan.emba
    ```

- **Qiling** (built on **Unicorn**): a programmable emulation framework in Python. Where QEMU runs a process, Qiling lets you script the environment: hook syscalls, fake files and peripherals, start and stop execution at arbitrary addresses, and emulate bare-metal or partial firmware. Unicorn underneath is a bare CPU emulator (no OS), perfect for running an isolated routine (a decryption function, a checksum, a [UDS security-access](../automotive/uds-obd2.md) key algorithm) to recover its behavior.

    ```python
    # Qiling: run a firmware's userland binary in a faked rootfs and hook a function
    from qiling import Qiling
    ql = Qiling(["./squashfs-root/bin/httpd"], "./squashfs-root")
    ql.run()
    ```

## Choosing the approach

| You want to | Use |
| --- | --- |
| Run one Linux binary quickly | QEMU user mode |
| Boot a whole router/IoT image and poke its web UI | FirmAE |
| Automated triage, CVEs, secrets, components | EMBA |
| Script the environment, emulate a routine or bare-metal | Qiling / Unicorn |
| Recover what a single isolated function computes | Unicorn |

## Where it fits in the workflow

1. [Extract](../attacks/firmware-extraction.md) and [carve](../software/binwalk.md) the image.
2. Triage with **EMBA** to inventory components, versions, and obvious issues.
3. Read key binaries statically in [Ghidra](ghidra-setup.md).
4. **Emulate** to confirm and exercise: FirmAE for the whole system, QEMU/Qiling for a specific binary or routine.
5. Breakpoint the function you care about and watch real arguments, closing the loop between static and dynamic. See the [RE workflow](firmware-workflow.md).

Emulation does not replace the hardware for peripheral-heavy behavior (radios, custom chips), but for the software surface, which is where most findings live, it is faster, safer, and more repeatable than working on the device.

## Related pages

- [Firmware RE Workflow](firmware-workflow.md)
- [Ghidra Setup](ghidra-setup.md)
- [Firmware Analysis](../attacks/firmware-analysis.md)
- [Binwalk and Extraction](../software/binwalk.md)
- [UDS and OBD-II](../automotive/uds-obd2.md)
