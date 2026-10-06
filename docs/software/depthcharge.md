# Depthcharge

Depthcharge is a Python toolkit for attacking **U-Boot**. Once you have an interactive [bootloader prompt](../attacks/bootloader-access.md) over [UART](../protocols/uart.md), Depthcharge automates the tedious parts: discovering which commands and features are available, reading and writing memory and registers, dumping flash, and turning a limited console into a memory read/write primitive. It is the power tool for the bootloader stage.

## What it is for

Interacting with U-Boot by hand works but is slow and error-prone. Depthcharge wraps a U-Boot console in a Python API so you can script it, and it knows many tricks for extracting data even from a restricted prompt:

- **Enumerate the environment**: list available commands, environment variables, and compiled-in features.
- **Read and write memory** through whatever primitives the console exposes (`md`/`mw`, and cleverer paths when those are missing).
- **Dump flash and memory** to files reliably, handling the transfer over the slow serial link.
- **Find register and memory details** to inform a deeper attack.

## Using it

Depthcharge attaches to the serial console (or a captured session) and builds a model of the target's capabilities:

```python
from depthcharge import Console, Depthcharge

ctx = Depthcharge(Console("/dev/ttyUSB0", baudrate=115200))
ctx.commands()                 # what U-Boot commands are available
print(ctx.environment())       # dump the environment (bootargs, bootcmd, secrets)

# read a region of memory to a file using the best available primitive
data = ctx.memory.read(0x80000000, 0x100000)
open("dump.bin", "wb").write(data)
```

It picks the most effective available method for each operation, so even a console missing the obvious commands can often be coerced into revealing memory.

## Why it beats doing it by hand

- **Speed and reliability** over a serial link, where a manual `md` loop is painful and lossy.
- **Capability discovery**: it tells you what the specific U-Boot build allows, which shortcuts the trial-and-error on an unfamiliar target.
- **Memory primitives from unlikely commands**: Depthcharge knows how to build read/write out of commands not obviously meant for it, which matters on locked-down or stripped consoles.

## Where it fits

1. [Interrupt U-Boot](../attacks/bootloader-access.md) to reach the `=>` prompt (Depthcharge can help probe how).
2. Point Depthcharge at the console and enumerate.
3. Dump the environment (often holding boot logic and sometimes secrets) and the flash/memory.
4. Use the findings to [extract firmware](../attacks/firmware-extraction.md), force a [shell](../attacks/getting-a-shell.md), or inform a [bootloader bypass](../attacks/bootloader-bypass.md).

## Related pages

- [Bootloader Access (U-Boot)](../attacks/bootloader-access.md)
- [Bootloader Bypass](../attacks/bootloader-bypass.md)
- [Getting a Shell](../attacks/getting-a-shell.md)
- [Firmware Extraction](../attacks/firmware-extraction.md)
- [Serial Consoles](serial-consoles.md)
