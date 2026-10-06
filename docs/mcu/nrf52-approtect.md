# nRF52 APPROTECT

Nordic's nRF52 series powers a huge number of Bluetooth Low Energy devices: wearables, locks, beacons, medical gadgets. Its firmware protection is **APPROTECT** (access port protection), and it is the textbook example of a read-out protection defeated by a single, cheap, well-timed glitch.

## What APPROTECT does

When APPROTECT is enabled, the nRF52 disables the debug access port, so [SWD](../protocols/swd.md) cannot read the internal flash. On the face of it this is the same deal as [STM32 RDP](stm32-rdp.md): debug blocked, firmware safe.

The weakness is in how it is enforced. On the original nRF52 silicon, APPROTECT is handled by the boot ROM copying a value from a flash register (UICR) into a peripheral register early in boot, and only then is the debug port gated. That copy-and-enforce step happens at a predictable moment after reset, and it is a single point of failure.

## The single-glitch bypass

The published attack (widely reproduced and documented) is remarkably simple:

1. Reset the chip and watch for the moment the boot ROM enforces APPROTECT.
2. Deliver one [voltage glitch](../attacks/fault-injection.md) at that instant to corrupt the enforcement so the debug port is left enabled.
3. Connect over SWD and read the flash normally.

Because the vulnerable operation is a single register write during a narrow, repeatable window, the parameter search is small and the success rate is high compared to many glitch targets. A ChipWhisperer or even a modest custom glitcher recovers firmware from affected parts in short order. This bug is why APPROTECT became a canonical teaching example for fault injection.

## Hardware revisions matter

Nordic responded with newer silicon revisions and a hardened **APPROTECT** (and a related `SECUREAPPROTECT` on the nRF5340/nRF91) that enforce protection differently and resist the original single-glitch technique. So the exact outcome depends on the chip revision:

- **Older/affected nRF52 revisions**: the classic single-glitch bypass applies.
- **Newer revisions**: the simple attack is mitigated; expect more effort, and check current research for that specific part and date code.

Identify the precise variant and revision during [recon](../attacks/recon.md) before assuming either result.

## Workflow

1. Confirm the part is an nRF52 and note the revision.
2. Check APPROTECT state with [OpenOCD](../software/openocd.md) or `nrfjprog`: a blocked flash read on a running device indicates it is enabled.
3. For affected revisions, set up a [glitch rig](../attacks/fault-injection.md#the-equipment), trigger on reset, and sweep the narrow post-reset window.
4. On success, dump flash over SWD and proceed to [firmware analysis](../attacks/firmware-analysis.md).

## Defender's view

Use a current silicon revision with the hardened protection, and do not rely on the original APPROTECT alone. The lesson of this bug is general: any protection enforced by a single instruction or register write is a glitch target, so make enforcement redundant. See the [fault injection defender notes](../attacks/fault-injection.md#defenders-view).

## Related pages

- [MCU Targets](index.md)
- [STM32 RDP](stm32-rdp.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [SWD](../protocols/swd.md)
- [RF Attacks](../radio/rf-attacks.md)
