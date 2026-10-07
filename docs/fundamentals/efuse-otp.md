# eFuse and OTP Memory

Many secrets on a modern chip do not live in the flash you can clip onto. They live in **one-time-programmable (OTP)** memory inside the die: eFuses, antifuses, or a dedicated OTP block. This is where keys, lock bits, and permanent configuration are burned so they survive power-off and cannot be rewritten. Understanding OTP is central to hardware hacking because it is both the thing that locks a device down and, occasionally, the thing you can read out. Pair this with [Memory Types](memory-types.md) and [Secure Elements](../attacks/secure-elements.md).

## What OTP stores

The exact layout is vendor-specific, but the same categories appear everywhere:

- **Keys.** AES keys for flash or [bitstream encryption](../fpga/bitstream-encryption.md), HMAC keys, and public-key hashes that root [secure boot](../attacks/chain-of-trust.md).
- **Lock and security bits.** Flags that disable [JTAG](../protocols/jtag.md), close a [boot ROM download mode](../platform/soc-recovery-modes.md), raise a read-protection level ([STM32 RDP](../mcu/stm32-rdp.md), [nRF52 APPROTECT](../mcu/nrf52-approtect.md)), or force encrypted-only boot.
- **Configuration.** Boot source selection, clock trims, feature enables, and die/lot identifiers.
- **Monotonic counters.** Anti-rollback values that can only increase, used to refuse downgraded firmware.

## How it works physically

- **eFuse.** A thin polysilicon or metal link that is blown by a programming current, like a microscopic fuse. Blown is permanent.
- **Antifuse.** The inverse: a normally-open structure that is made conductive by gate-oxide breakdown. Dense and hard to read, used in high-security parts.
- **OTP block.** A one-time-writable region of an otherwise normal memory, with the same effect from software's view.

The common property is irreversibility. A burned bit cannot be reset, which is why OTP enforces security state: once secure boot or read protection is latched, there is no software path back.

## Why it matters to an attacker

- **It holds the key you want.** If a flash or bitstream key sits in OTP, recovering it breaks everything encrypted under it.
- **It holds the bit stopping you.** A single lock bit may be all that stands between you and open [JTAG](../protocols/jtag.md) or a readable flash.
- **It is readable to the die, so possibly to you.** The chip reads these bits at boot. Any path that exposes that read, logically or physically, is an opening.

## Attacks on OTP

- **Optical read after decap.** Blown and intact fuses often look physically different. After [decapping](../attacks/decapping.md), a fuse bank can sometimes be read directly under a microscope or SEM, recovering bits a byte at a time. Antifuses are much harder to image than classic eFuses.
- **Passive voltage contrast and probing.** Semi-invasive imaging in an SEM can distinguish fuse states by charge, and microprobing can tap the sense lines that read OTP into the chip.
- **Fault injection on the read.** The security bit is read once at boot into a register or comparator. A [voltage, clock, or EM glitch](../attacks/fault-injection.md) at that instant can make the chip sample a locked fuse as unlocked, bypassing the protection without changing the fuse. This is the cheap, popular attack and underlies many [RDP](../mcu/stm32-rdp.md) and [APPROTECT](../mcu/nrf52-approtect.md) bypasses.
- **Incomplete burns.** Development or recovery states where fuses are only partially blown, or where a debug fuse was left unset, are a frequent real-world weakness.

## Defender's view

Treat OTP state as the anchor of the whole trust chain, and assume the attacker will try to glitch the one read that latches it. Read security bits redundantly and at multiple points, not once, so a single faulted sample does not unlock the device. Prefer antifuse or hardened OTP over simple eFuse where optical readout is a concern, store only hashes of public keys rather than secrets where possible, and burn every production lock bit (JTAG disable, encrypted-only boot, read protection) as part of manufacturing rather than trusting software configuration to take effect. Add [glitch detection](../attacks/fault-injection.md#defenders-view) around the fuse-sampling logic.

## Related pages

- [Memory Types](memory-types.md)
- [Chip Identification](chip-identification.md)
- [Secure Elements](../attacks/secure-elements.md)
- [Decapping and Invasive Attacks](../attacks/decapping.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [STM32 RDP](../mcu/stm32-rdp.md)
- [nRF52 APPROTECT](../mcu/nrf52-approtect.md)
- [Chain of Trust and Secure Boot](../attacks/chain-of-trust.md)
