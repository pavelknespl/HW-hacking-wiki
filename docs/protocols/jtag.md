# JTAG

JTAG is a debug and test interface that, when enabled, gives you something close to total control of a processor: halt it, single-step it, read and write every register and every byte of memory, and read internal flash. Where [UART](uart.md) gives you whatever the device chooses to print, JTAG lets you take what you want.

## What JTAG actually is

JTAG (IEEE 1149.1) was designed for boundary scan, a way to test board connections at manufacture. Silicon vendors layered CPU debug on top of the same pins, and that debug access is what we use. A chain of devices shares a small state machine driven by a few signals:

| Signal | Meaning |
| --- | --- |
| [`TCK`](../reference/pin-reference.md#tck) | Test clock, drives the state machine |
| [`TMS`](../reference/pin-reference.md#tms) | Test mode select, steers the state machine |
| [`TDI`](../reference/pin-reference.md#tdi) | Test data in, data into the chip |
| [`TDO`](../reference/pin-reference.md#tdo) | Test data out, data from the chip |
| [`TRST`](../reference/pin-reference.md#trst) | Test reset, optional |

Plus `GND` and a reference `VCC`. Multiple chips can be daisy-chained, so a scan may reveal more than one device in the chain.

## Why it is powerful

- **Halt and inspect.** Stop the CPU mid-execution and read all registers and memory, including DRAM contents at runtime.
- **Read internal flash.** On microcontrollers whose firmware lives on-chip, JTAG reads it directly, no external memory chip needed, unless [read-out protection](swd.md) blocks it.
- **Write memory and registers.** Patch running code, change variables, redirect execution.
- **Bypass software locks.** If a login or check lives in RAM, you can often just change it.

## Finding the JTAG pins

JTAG pins are quiet unless a debugger drives them, so they are harder to spot than [UART](uart.md). See [Finding and Identifying Pins](../fundamentals/identifying-pins.md). In short:

- **Look for a standard header.** Many boards use the 20-pin or 10-pin ARM JTAG layout, or a 2x5 0.05 inch header. A recognizable connector saves hours.
- **Continuity to the SoC.** If the datasheet gives the TAP ball locations, trace them to a header or test points.
- **Brute force.** A **JTAGulator** or `jtagenum` (on a microcontroller or [Raspberry Pi](../scripting/gpio-control.md)) drives candidate pins and reports a valid TAP by reading its **IDCODE**. A returned IDCODE confirms the pinout and that JTAG is alive.

## Connecting with OpenOCD

[OpenOCD](../software/openocd.md) is the open-source bridge between a JTAG adapter and your debugger. A typical invocation names an interface config and a target config:

```bash
openocd -f interface/jlink.cfg -f target/stm32f1x.cfg
```

For a generic FT2232H adapter and an unknown target, you start with the interface file and a transport, then scan:

```bash
openocd -f interface/ftdi/ft2232h.cfg -c "transport select jtag" \
        -c "adapter speed 1000" -c "init" -c "scan_chain" -c "shutdown"
```

A successful scan prints the IDCODEs in the chain. From there you connect GDB to [OpenOCD](../software/openocd.md)'s port 3333 and work interactively:

```bash
gdb-multiarch
(gdb) target remote :3333
(gdb) monitor halt
(gdb) monitor reset halt
```

## Dumping memory over JTAG

Once halted, dump a region to a file:

```
# inside an OpenOCD telnet/console session (port 4444)
halt
dump_image firmware.bin 0x08000000 0x20000
```

The base address and length come from the chip's memory map in its datasheet. For an MCU, the internal flash base (for example `0x08000000` on many STM32 parts) and size give you the whole firmware.

## Read-out protection

The main obstacle. Vendors provide a lock that disables debug reads of internal flash:

- STM32 **RDP** levels 1 and 2.
- NXP **CRP**.
- Microchip, Nordic, and others have equivalents.

When set, `dump_image` on the flash region returns zeros, errors, or triggers a mass erase. Defeating read-out protection is a primary use case for [fault injection](../attacks/fault-injection.md): glitch the chip at the moment it checks the protection bit so it reads flash out anyway.

## JTAG versus SWD

On ARM Cortex-M parts, the same debug logic is usually exposed over the two-wire [SWD](swd.md) interface, which is simpler to wire and just as capable for debugging. Many adapters speak both. Use SWD when the target offers it and you only have two free pins; use JTAG for boundary scan, daisy chains, and non-ARM targets.

## Defender's view

Disable or fuse off JTAG on production silicon, or at minimum enable the highest read-out protection level. An exposed, unlocked JTAG port is equivalent to publishing your firmware and handing over runtime control of the device.


## Related pages

- [Pin Reference](../reference/pin-reference.md)
- [SWD](swd.md)
- [OpenOCD](../software/openocd.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [Finding and Identifying Pins](../fundamentals/identifying-pins.md)
