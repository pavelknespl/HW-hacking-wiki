# Flipper Zero

The Flipper Zero is a pocket multi-tool that bundles a sub-GHz radio, NFC/RFID, infrared, iButton, a USB BadUSB interface, and GPIO into one device. It will not replace a [Proxmark3](proxmark3.md), a [logic analyzer](logic-analyzers.md), or an [SDR](../radio/sdr-basics.md) for serious work, but it is the fastest way to triage common wireless and access-control targets in the field. For authorized testing of devices you own.

## What it actually does well

| Capability | Good for |
| --- | --- |
| **Sub-GHz** (300 to 928 MHz) | Capturing and replaying [fixed-code remotes](../radio/key-fobs.md), reading protocols |
| **125 kHz RFID** | Reading and emulating LF access cards (EM4100, HID) |
| **13.56 MHz NFC** | Reading, emulating, and some MIFARE Classic work |
| **Infrared** | Universal remote, capturing IR protocols |
| **iButton / 1-Wire** | Dallas key reading and emulation |
| **GPIO / UART** | A [serial console](serial-consoles.md) and basic pin control |
| **BadUSB** | Emulating a USB HID keyboard for scripted input |

## Where it fits, and where it does not

The Flipper shines at the **easy cases**: a fixed-code garage remote, a static-ID badge, an IR-controlled device, a quick UART console in the field. Its sub-GHz and NFC are capable but limited by a small antenna and modest radio compared with dedicated tools.

It is **not** the right tool for:

- Serious [MIFARE key recovery](../radio/nfc-rfid.md) or iCLASS work, which wants a [Proxmark3](proxmark3.md).
- Wideband capture or anything above ~1 GHz, which wants an [SDR](../radio/sdr-basics.md).
- [Glitching](../attacks/fault-injection.md) or power analysis, which wants a [ChipWhisperer](chipwhisperer.md).
- Decoding fast buses, which wants a [logic analyzer](logic-analyzers.md).

Think of it as reconnaissance and quick wins, not the deep bench.

## Sub-GHz workflow

1. **Read / Read RAW** on the suspected frequency (check the device's [FCC ID](../attacks/recon.md) for the band) while triggering the remote.
2. The Flipper recognizes many protocols directly; for unknown ones, capture RAW.
3. **Replay** fixed-code captures to reproduce the action. Rolling codes will not replay, which itself tells you the design. See [Key Fobs](../radio/key-fobs.md).

## GPIO and UART

The GPIO header makes the Flipper a field [UART](../protocols/uart.md) adapter and simple pin tool:

- Use the USB-UART bridge or the GPIO pins to reach a target console.
- 3.3 V logic; mind levels and share ground as always. See [Level Shifting](../fundamentals/level-shifting.md).

## Firmware

Community firmware (unleashed, RogueMaster, Momentum) unlocks region-restricted sub-GHz frequencies and adds features. Use responsibly and legally; expanded TX capability does not change what you are allowed to transmit.

## Defender's view

The Flipper is a convenience, not a new threat: everything it does, dedicated tools already did. If a fixed-code remote, a static RFID badge, or an unauthenticated IR/sub-GHz command is a problem, it was a problem before the Flipper existed. Fix the underlying weak design (rolling codes, authenticated access cards) rather than worrying about the specific gadget.

## Related pages

- [Proxmark3](proxmark3.md)
- [NFC and RFID](../radio/nfc-rfid.md)
- [Key Fobs and Rolling Codes](../radio/key-fobs.md)
- [SDR Basics](../radio/sdr-basics.md)
- [Serial Consoles](serial-consoles.md)
