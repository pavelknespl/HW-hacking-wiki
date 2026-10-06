# NFC and RFID

RFID and NFC tags show up in access cards, transit passes, car keys, hotel locks, consumable tags, and payment. They are a distinct discipline from the [SDR](sdr-basics.md) work on the rest of this section: the frequencies are low, the protocols are specific, and the standard tool is a **Proxmark3** rather than a wideband radio. This page maps the landscape and the common weaknesses, for authorized testing of cards and systems you own.

## The two frequency worlds

| Band | Frequency | Typical tags | Range |
| --- | --- | --- | --- |
| **LF** | 125 / 134 kHz | EM4100, HID Prox, T5577, animal chips | a few cm |
| **HF** | 13.56 MHz | MIFARE Classic/DESFire, NTAG, iCLASS, EMV payment | up to ~10 cm |

LF is older and mostly dumb identifiers. HF (which includes **NFC**) carries the smart, cryptographic cards and is where the interesting security lives.

## LF: usually just an ID

Most 125 kHz tags transmit a fixed number and nothing else. An **EM4100** or basic **HID Prox** card is read and compared by the reader with no authentication:

- Capture the ID with a [Proxmark3](../software/proxmark3.md) (`lf search`).
- Write it to a rewritable **T5577** card and you have a working clone.

These systems rely entirely on the secrecy of a number that the card broadcasts to any reader, so they are clone-by-design. The finding is simply "the badge is a static ID, cloneable in seconds".

## HF: where cryptography (sometimes) lives

13.56 MHz tags range from dumb (NTAG) to genuinely secure (DESFire EV2, modern EMV). The classic target is **MIFARE Classic**:

- It uses the proprietary **Crypto-1** cipher, which is broken. Attacks (**darkside**, **nested**, **hardnested**) recover the sector keys from a card, often in seconds to minutes with a Proxmark3.
- Once keys are recovered, the card is dumped and cloned to a "magic" card (one with a writable UID block).
- Many access systems still use MIFARE Classic despite this, which makes it a frequent real finding.

Stronger HF cards (**DESFire EV1/EV2**, **iCLASS SE**, EMV) use real cryptography (AES/3DES) and resist cloning; attacks there target implementation flaws, downgrade to legacy modes, or recovered keys, not the cipher itself.

## Tools

- **Proxmark3** is the standard: reads and writes LF and HF, runs the MIFARE key-recovery attacks, emulates tags, and sniffs reader-to-card traffic. See [Proxmark3](../software/proxmark3.md).
- **Flipper Zero** handles common LF/HF read-clone-emulate for the easy cases and is pocketable. See [Flipper Zero](../software/flipper-zero.md).
- **ACR122U** and libnfc for scripted NFC work on a budget.
- **Chameleon** devices for tag emulation.

## Sniffing and relay

Beyond clone-from-a-card, you can:

- **Sniff** the reader-to-card exchange with a Proxmark3 to understand the protocol and catch keys or data in transit.
- **Relay** (extend the range between a real card and a reader) to defeat the "must be physically present" assumption, a known attack class against contactless payment and access.

## Defender's view

Do not use 125 kHz static-ID badges or MIFARE Classic for anything that matters; both are trivially cloned. Use DESFire EV2 or equivalent with diversified AES keys, enable mutual authentication, and consider anti-relay timing (as EMV does). Treat any card that answers the same bytes every time as a password written on the outside of the door.

## Related pages

- [Proxmark3](../software/proxmark3.md)
- [Flipper Zero](../software/flipper-zero.md)
- [SDR Basics](sdr-basics.md)
- [Key Fobs and Rolling Codes](key-fobs.md)
- [Side-Channel Attacks](../attacks/side-channel.md)
