# ARM Trace (SWO, ITM, ETM)

Debug interfaces like [SWD](swd.md) and [JTAG](jtag.md) let you halt and poke a core. **Trace** is different: it streams out what the core is doing *while it runs*, in real time, without stopping it. On ARM this comes in two flavors, the low-cost single-wire **SWO/ITM** and the high-bandwidth parallel **ETM**. For a reverse engineer, trace turns a black-box MCU into something you can watch execute. For authorized work on your own targets.

## SWO and ITM: cheap printf and events

**SWO** (Serial Wire Output) is a single extra pin alongside [SWD](swd.md). The core's **ITM** (Instrumentation Trace Macrocell) and **DWT** (Data Watchpoint and Trace) units send data out over it:

- **ITM** gives software a cheap `printf`-style channel (a few cycles per message) and event stamps, far less intrusive than a UART.
- **DWT** can emit **data trace**: notifications whenever a chosen memory address is read or written, and PC samples for statistical profiling.

Capturing SWO needs only the extra pin and a probe that supports it (ST-Link, J-Link) plus the right [OpenOCD](../software/openocd.md)/tooling config:

```
# OpenOCD: enable the TPIU/SWO and dump ITM output
tpiu config internal swo.log uart off <core_clock> <swo_baud>
itm port 0 on
```

For a reverse engineer, DWT data trace is the useful part: point it at a variable or peripheral register and watch every access live, which reveals how firmware uses a value without single-stepping.

## ETM: full instruction trace

**ETM** (Embedded Trace Macrocell) is the heavyweight: it emits a compressed stream of **every branch the core takes**, enough to reconstruct the exact execution path. This is reverse-engineering gold, you get the real control flow of running firmware, including paths that are hard to reach statically.

The cost is bandwidth and pins:

- ETM streams over a **parallel trace port** (TRACECLK plus several TRACEDATA lines) or, on newer parts, high-speed serial trace.
- It needs a capable probe and buffer: a J-Trace, a Lauterbach, or an FPGA-based capture, not a basic ST-Link.
- You then decode the compressed trace against the binary to recover the instruction-by-instruction path.

Because of the hardware cost, ETM is less common on the hobbyist bench than SWO, but when available it is the most powerful dynamic-analysis channel short of full [emulation](../reverse-engineering/emulation.md).

## Finding the pins

- **SWO** is a single pin, often broken out on the [Cortex Debug 10-pin header](../reference/pinouts.md#arm-cortex-debug-10-pin-header) (pin 6, shared with [TDO](../reference/pin-reference.md#tdo)). See [SWD](swd.md).
- **ETM/parallel trace** uses a dedicated trace connector (the 20-pin Cortex+ETM header) with TRACECLK and TRACEDATA0-3. If the board does not route these, ETM is not reachable without silicon rework.

## Why trace matters for RE

Static analysis in [Ghidra](../reverse-engineering/ghidra-setup.md) shows what code exists; trace shows what code **runs** and in what order, with live data values, all without halting timing-sensitive firmware. It complements the [RE workflow](../reverse-engineering/firmware-workflow.md): use SWO/DWT to watch specific accesses on any SWD target, and reach for ETM when you need the complete execution path and the hardware supports it.

## Related pages

- [SWD](swd.md)
- [JTAG](jtag.md)
- [OpenOCD](../software/openocd.md)
- [Firmware RE Workflow](../reverse-engineering/firmware-workflow.md)
- [Emulation and Dynamic Analysis](../reverse-engineering/emulation.md)
