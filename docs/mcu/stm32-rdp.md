# STM32 RDP

STM32 is the most common Cortex-M family you will meet, and its firmware protection is **RDP** (Readout Protection), stored in an option byte. Understanding its three levels, and the well-documented glitch that defeats level 1, is a standard part of assessing an STM32-based device.

## The three RDP levels

| Level | Value | Effect |
| --- | --- | --- |
| **0** | `0xAA` | No protection. Debug reads flash freely. |
| **1** | anything except `0xAA`/`0xCC` | Debug blocked from flash/SRAM, but **dropping to level 0 triggers a mass erase**. User code still runs and can access flash. |
| **2** | `0xCC` | Debug port permanently disabled. No going back. The strongest and, by design, irreversible. |

The crucial distinction:

- **RDP level 1** is the common shipping state and the interesting one. The debugger is blocked from reading flash, but the protection's own escape hatch (lower it to 0) wipes the chip, so you cannot simply reset it to read the code. The firmware itself, however, still reads its own flash normally.
- **RDP level 2** truly closes the debug interface. There is no supported way back, and glitching it is far harder because the port is dead rather than merely gated.

## The level 1 glitch

The classic attack targets RDP level 1. The insight: although the debugger cannot read flash, running firmware can, and there is a brief window during boot where the protection check happens. A well-timed [voltage glitch](../attacks/fault-injection.md) at that moment can cause the chip to treat the flash as unprotected for a read.

Several concrete techniques have been published against various STM32 series:

- **Glitch the RDP check at reset** so the debug access port is enabled while flash is still readable, then dump over [SWD](../protocols/swd.md) before the chip settles.
- **Cold-boot stepping / debug-while-booting** variants that read small windows of flash each attempt and stitch the dumps together.
- **Exploiting peripheral access**: in some series, a debug-accessible peripheral (or SRAM where a copied routine runs) can be coerced into revealing flash contents word by word.

The campaign is the standard [glitch loop](../attacks/fault-injection.md#the-workflow): trigger on reset, sweep glitch offset and width, detect the success condition (a flash word reads as real data instead of `0x00`/error), and repeat. Because you often recover only a few bytes per successful glitch, automation and patience matter.

## Practical workflow

1. **Identify the exact part** (series and revision) during [recon](../attacks/recon.md); specific glitch parameters and known bypasses are series-dependent.
2. **Check the current RDP level** with [OpenOCD](../software/openocd.md): a flash read returning all zeros or an error, while the chip clearly runs, indicates level 1.
3. If level 1, set up a [glitch rig](../attacks/fault-injection.md#the-equipment) (a ChipWhisperer is the easy path) targeting the core voltage at reset.
4. Sweep parameters, capturing flash over SWD on each attempt, until you reconstruct the image.
5. If level 2, the debug port is dead; pivot to other routes (a [bootloader interface](../attacks/bootloader-access.md), an external interface, or accept that physical extraction may not be feasible without advanced methods).

## Defender's view

For anything sensitive, set **RDP level 2** and accept its irreversibility, because level 1 is defeatable by glitching. Combine it with the fault-resistant practices on the [fault injection](../attacks/fault-injection.md#defenders-view) page: redundant checks, so a single skipped instruction does not unlock the device.

## Related pages

- [MCU Targets](index.md)
- [SWD](../protocols/swd.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [OpenOCD](../software/openocd.md)
- [nRF52 APPROTECT](nrf52-approtect.md)
