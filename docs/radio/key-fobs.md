# Key Fobs and Rolling Codes

Remote keyless entry (garage doors, car fobs, gate remotes, some alarms) is a sub-GHz radio problem with a long history of weak designs. It deserves its own treatment because the security model is specific: it is almost entirely about whether the code transmitted is **static**, **rolling**, or **challenge-response**, and each fails differently. For authorized testing of devices you own. See [RF Attacks](rf-attacks.md) for the general sub-GHz workflow.

## The three designs

=== "Fixed code"

    The remote sends the same code every press. Common on old garage doors, cheap gates, and some alarms (often 300 to 433 MHz, ASK/OOK).

    - [Capture once](sdr-basics.md) and re-transmit to repeat the action. Pure [replay](rf-attacks.md#replay-the-most-common-failure).
    - Some use DIP-switch addresses with only a few thousand combinations, brute-forceable directly.
    - A [Flipper Zero](../software/flipper-zero.md) or an SDR clones these trivially.

=== "Rolling code"

    Each press sends a different value derived from a secret key and an incrementing counter (KeeLoq and successors). A captured code is not valid again, defeating naive replay.

    - Still attackable. Published weaknesses include recovering the manufacturer/device key (KeeLoq key-recovery, some via [side-channel](../attacks/side-channel.md)), and counter-window abuse.
    - **RollJam-style capture-and-withhold**: jam the receiver while capturing the fob's code so the car never sees it, let the user press again (you capture a second, the first still unused), then forward the first and keep the second for later. Exploits that the receiver accepts a window of future counters.

=== "Challenge-response (modern cars)"

    The car transmits a challenge, the fob replies with a cryptographic response (and often the fob must be in range). Immobilizers and modern PKE work this way.

    - Resists replay and simple rolling-code attacks.
    - The dominant real attack is the **relay attack**: two devices extend the range between the fob (in the house) and the car, so the car thinks the fob is present. It defeats proximity, not the crypto.
    - Implementation and key-derivation flaws in specific immobilizer transponders (various DST/Hitag/Megamos findings) have also broken real systems.

## Tools

- **Flipper Zero** for capturing, analyzing, and replaying common fixed-code sub-GHz remotes. See [Flipper Zero](../software/flipper-zero.md).
- **RTL-SDR** to receive and **HackRF** (where legal to transmit) for analysis; **Universal Radio Hacker** to decode the bitstream and find the structure.
- For transponder/immobilizer work, a **Proxmark3** or dedicated transponder tools (these are LF 125/134 kHz, overlapping [RFID](nfc-rfid.md)).

## The workflow

1. **Find the frequency and modulation** (FCC ID, then confirm on an SDR). Most are 300 to 868 MHz, ASK/OOK or FSK.
2. **Capture several presses** and diff the decoded bits: fully static means fixed code; changing tail means rolling.
3. **Fixed**: replay. **Rolling**: assess counter handling and whether a capture-and-withhold applies, or move to firmware/key recovery.
4. **Challenge-response**: evaluate relay feasibility and look for known transponder weaknesses rather than attacking the radio directly.

## Defender's view

Use rolling codes at minimum, and prefer challenge-response with sound key management. Against relay attacks (the modern threat), add **distance bounding** or UWB ranging (as newer car keys do), and offer a motion-sensing fob that sleeps when still. Treat any fixed-code remote as equivalent to a repeatable password sent in the clear.

## Related pages

- [RF Attacks](rf-attacks.md)
- [SDR Basics](sdr-basics.md)
- [Flipper Zero](../software/flipper-zero.md)
- [NFC and RFID](nfc-rfid.md)
- [Side-Channel Attacks](../attacks/side-channel.md)
