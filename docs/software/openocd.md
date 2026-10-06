# OpenOCD

OpenOCD (Open On-Chip Debugger) is the open-source glue between a [JTAG](../protocols/jtag.md) or [SWD](../protocols/swd.md) adapter and your debugger. It drives the debug transport, exposes a GDB server and a command console, and knows how to flash and dump a wide range of targets. If you are working over [JTAG](../protocols/jtag.md) or [SWD](../protocols/swd.md), OpenOCD is almost always in the loop.

## The two config files

An OpenOCD session needs to know two things: what adapter you have, and what target it is connected to. These come from config files, usually one of each:

```bash
openocd -f interface/<adapter>.cfg -f target/<chip>.cfg
```

- **Interface config** describes your debug probe: `interface/stlink.cfg`, `interface/jlink.cfg`, `interface/cmsis-dap.cfg`, `interface/ftdi/ft2232h.cfg`.
- **Target config** describes the chip: `target/stm32f4x.cfg`, `target/nrf52.cfg`, `target/rp2040.cfg`.

These ship with OpenOCD under its `scripts/` directory. Inline `-c` commands override or extend them.

## Choosing the transport

Many adapters speak both JTAG and SWD. Select explicitly to avoid surprises:

```bash
openocd -f interface/ftdi/ft2232h.cfg -c "transport select swd" -f target/stm32f1x.cfg
```

Use `transport select jtag` for JTAG, `transport select swd` for SWD. See the protocol pages for when to prefer each.

## First contact with an unknown target

When you do not yet know the chip, connect with just the interface and scan:

=== "JTAG scan"

    ```bash
    openocd -f interface/ftdi/ft2232h.cfg \
      -c "transport select jtag" \
      -c "adapter speed 1000" \
      -c "init" -c "scan_chain" -c "shutdown"
    ```

    A valid chain prints IDCODEs you can look up to identify the part.

=== "SWD / DAP info"

    ```bash
    openocd -f interface/cmsis-dap.cfg \
      -c "transport select swd" \
      -c "adapter speed 1000" \
      -c "init" -c "dap info" -c "shutdown"
    ```

    This reports the debug port and core if SWD is alive.

Start the adapter speed low (around 1000 kHz) and raise it once you have a stable connection.

## The command console (telnet / tcl)

Once running, OpenOCD listens on:

- **Port 3333** for GDB.
- **Port 4444** for a telnet command console.
- **Port 6666** for the Tcl interface.

Connect to the console and drive the target:

```bash
telnet localhost 4444
```

Useful commands:

```
reset halt           ; reset and immediately stop the core
halt                 ; stop the running core
resume               ; continue
reg                  ; dump registers
mdw 0x08000000 16    ; memory display, 16 words
dump_image fw.bin 0x08000000 0x100000   ; save a region to a file
```

## Dumping firmware

With the core halted, `dump_image` writes a memory region to disk. The base and length come from the chip's memory map:

```
reset halt
dump_image firmware.bin 0x08000000 0x100000
```

If the region reads back as all zeros or the command errors, suspect [read-out protection](../protocols/swd.md#read-out-protection-again). That is a job for [fault injection](../attacks/fault-injection.md).

## Flashing

OpenOCD writes images too, which matters when you want to run modified code on your own target:

```
program firmware.bin verify reset exit 0x08000000
```

## Driving it from GDB

For interactive debugging, point GDB at OpenOCD:

```bash
gdb-multiarch firmware.elf
(gdb) target extended-remote :3333
(gdb) monitor reset halt
(gdb) load            # if you want to flash the elf
(gdb) break main
(gdb) continue
```

`monitor` passes a command straight to OpenOCD's console, so `monitor dump_image ...` works from inside GDB.

## Common problems

- **"Error: no device found"** points at the interface config or USB permissions. Check the adapter is detected (`lsusb`) and that you have a udev rule or run with the right permissions.
- **Unstable connection, random resets** usually means the adapter speed is too high. Drop `adapter speed`.
- **"target not halted"** means a command needs the core stopped first; run `halt` or `reset halt`.
- **Reads return zeros on internal flash** is the signature of [read-out protection](../protocols/swd.md), not a wiring fault.

## Alternatives

- **pyOCD** is a Python tool focused on CMSIS-DAP and Cortex-M, pleasant for scripting.
- **Vendor tools** (ST-Link Utility / STM32CubeProgrammer, J-Link Commander, nRF Command Line Tools) sometimes handle quirks of their own silicon better.

For most cross-vendor work, OpenOCD plus GDB is the dependable combination.


## Related pages

- [JTAG](../protocols/jtag.md)
- [SWD](../protocols/swd.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [Ghidra Setup](../reverse-engineering/ghidra-setup.md)
- [Firmware Extraction](../attacks/firmware-extraction.md)
