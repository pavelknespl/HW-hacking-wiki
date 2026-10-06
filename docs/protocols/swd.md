# SWD

SWD, the Serial Wire Debug interface, is ARM's two-wire replacement for [JTAG](jtag.md) on Cortex-M and many Cortex-A parts. It exposes the same debug capability, halting the core, reading and writing memory, and dumping internal flash, over just two signals instead of four or five. On modern ARM microcontrollers it is the debug interface you will use most.

## The signals

| Signal | Meaning |
| --- | --- |
| `SWDIO` | Bidirectional data |
| `SWCLK` | Clock |
| `GND` | Ground reference |
| `nRESET` | Optional reset line, helpful for connecting under reset |

Some headers also break out `SWO`, a single-wire trace output for debug printf-style messages. It is useful but not required to debug.

Because SWD multiplexes data and direction onto one `SWDIO` line, it needs only two active pins, which is why tiny packages favor it.

## When you will meet it

- Almost every **ARM Cortex-M** microcontroller (STM32, nRF5x, SAM, Kinetis, RP2040, and so on).
- Many Cortex-A application processors expose SWD alongside or instead of [JTAG](jtag.md).
- Development boards expose it through an onboard debugger (ST-Link, CMSIS-DAP, J-Link OB).

Many debug adapters auto-negotiate between JTAG and SWD, and ARM's **SWJ-DP** lets a target offer both on shared pins.

## Finding the pins

- **Standard connectors.** The ARM 10-pin Cortex Debug (0.05 inch) and the older 20-pin header both carry SWD. A Tag-Connect footprint is another common sight.
- **Silkscreen labels.** `SWDIO`, `SWCLK`, `SWO`, `RST` are often printed.
- **Two quiet pins near the MCU** that trace back to the documented SWD balls. See [Finding and Identifying Pins](../fundamentals/identifying-pins.md).

## Connecting with OpenOCD

Select the SWD transport explicitly:

```bash
openocd -f interface/stlink.cfg -c "transport select swd" -f target/stm32f4x.cfg
```

For a generic CMSIS-DAP or FT2232H adapter against an unknown Cortex-M:

```bash
openocd -f interface/cmsis-dap.cfg -c "transport select swd" \
        -c "adapter speed 1000" -c "init" -c "dap info" -c "shutdown"
```

A good connection reports the DAP and the core. Then attach GDB or use the [OpenOCD](../software/openocd.md) console as with [JTAG](jtag.md).

## Dumping internal flash

```
halt
dump_image firmware.bin 0x08000000 0x100000
```

The flash base and size come from the part's datasheet. Vendor tools also work: `st-flash read firmware.bin 0x08000000 0x100000` for ST-Link, or `pyocd` for CMSIS-DAP targets.

```bash
# pyocd, convenient for many Cortex-M parts
pyocd cmd -c "savemem 0x08000000 0x100000 firmware.bin"
```

## Read-out protection, again

The same locks that block JTAG block SWD, because it is the same debug engine:

- STM32 **RDP level 1** disconnects flash from the debugger (and erases it if you then try to lower protection); **level 2** disables the debug port entirely.
- Nordic **APPROTECT**, NXP **CRP**, and others are equivalents.

A fully locked part returns no useful flash contents over SWD. This is exactly where [fault injection](../attacks/fault-injection.md) enters: a well-timed glitch during the protection check can let the read proceed. On some Nordic parts, historic APPROTECT bypasses needed only a single carefully timed fault.

## SWD versus JTAG, quickly

| | SWD | JTAG |
| --- | --- | --- |
| Pins | 2 (plus optional reset, SWO) | 4 to 5 |
| Scope | ARM debug (and most Cortex) | Debug plus boundary scan, any architecture |
| Daisy chain | No (point to point) | Yes |
| Best for | Cortex-M, tight pin budgets | Non-ARM, board test, multi-TAP chains |

If the target is a Cortex-M and offers SWD, use it; it is simpler and fully capable. Reach for JTAG when you need boundary scan, a daisy chain, or a non-ARM core.


## Related pages

- [JTAG](jtag.md)
- [OpenOCD](../software/openocd.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [Ghidra Setup](../reverse-engineering/ghidra-setup.md)
