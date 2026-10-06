# RF Attacks

Once you can [capture and decode](sdr-basics.md) a device's radio link, the security questions are the same ones you would ask of any protocol: is it authenticated, is it encrypted, does it resist replay. Cheap wireless devices frequently fail all three. This page surveys the common weaknesses and how analysis proceeds, for authorized testing of hardware you own.

!!! warning "Legality"
    Transmitting is regulated and often requires a license. Analyze and transmit only where you are legally allowed, on bands and devices you are authorized to use. Jamming is illegal in most jurisdictions. This page is about understanding weaknesses, not operating against systems you do not own.

## Replay: the most common failure

A great many simple remotes (garage doors, some alarm sensors, cheap smart plugs, basic key fobs) transmit a **fixed code** with no freshness. If the same bits open the device every time, a recording of one legitimate transmission can be re-sent to repeat the action.

The analysis:

1. Capture a transmission with an SDR or a sub-GHz tool.
2. Decode it to a bitstream and confirm, across several captures, that the payload does not change.
3. A static payload means the design relies on obscurity, not authentication. Re-transmitting the capture reproduces the command.

The defense is a **rolling code**, discussed below. If you find a static code, that is the finding.

## Rolling codes and their limits

Better remotes use a **rolling code** (hopping code): each press sends a different value derived from a secret and an incrementing counter, so a captured transmission is not valid again. This defeats naive replay.

Rolling codes are not automatically safe, though. Historical weaknesses in real systems have included:

- **Weak key derivation** that let the per-device key be recovered from a few captures.
- **Counter desynchronization** handling that could be abused.
- **Capture-and-withhold** schemes that exploit the receiver accepting a window of future counter values.

Analyzing a rolling-code system is largely reverse engineering the [firmware](../attacks/firmware-analysis.md) of the transmitter or receiver to understand the algorithm, since the weakness is in the cryptographic design rather than the radio.

## Unencrypted data links

Many devices send their actual data in the clear over the air:

- Wireless sensors broadcasting readings and identifiers (`rtl_433` decodes hundreds of these).
- Remote controls whose button-to-bit mapping is trivially recoverable.
- Telemetry that leaks device identifiers, status, or even credentials.

Decoding these is passive and often reveals more than expected: device addresses, state, and the command structure, which is the groundwork for any deeper work.

## Protocol reverse engineering

For a custom digital link, the work is structural, done in [URH](sdr-basics.md#from-samples-to-bits) or GNU Radio:

1. Capture many transmissions of known actions (button A, button B, idle beacon).
2. Align and diff the decoded frames to find the fields: preamble, sync word, device address, command, counter, checksum.
3. Confirm the checksum or CRC algorithm by brute-forcing common polynomials against captured frames.
4. With the frame format understood, you can interpret any transmission and, where authorized, craft valid ones.

This is the same reverse-engineering mindset as [firmware analysis](../attacks/firmware-analysis.md), applied to bits on the air.

## The common 2.4 GHz protocols

Beyond simple sub-GHz links, the crowded 2.4 GHz band carries protocols with their own tooling and weaknesses:

- **Bluetooth / BLE:** sniffable with dedicated hardware (a cheap nRF-based sniffer or Ubertooth). Pairing weaknesses, unauthenticated characteristics, and plaintext data are common findings in cheap devices.
- **Zigbee / 802.15.4:** sniffable with an appropriate radio; key management and default keys are the usual issues.
- **Proprietary nRF24-style links:** many wireless keyboards, mice, and toys use Nordic nRF24 radios with weak or no encryption.

Each has its own capture tools, but the analysis goal is unchanged: identify authentication, encryption, and replay resistance, and test each.

## Where RF meets the rest of the wiki

RF analysis rarely stands alone. The radio protocol is implemented in [firmware](../attacks/firmware-analysis.md) you can extract and reverse, the keys may sit in an [EEPROM](../protocols/i2c.md) or internal flash you can dump, and the cryptographic implementation may leak through a [side channel](../attacks/side-channel.md). A finding over the air is often fastest to confirm by reading the code that produced it.

## Defender's view

Authenticate and encrypt wireless commands, use rolling codes or proper challenge-response with sound key management, include anti-replay freshness, and do not rely on the obscurity of a custom protocol. Assume an attacker can receive, decode, and record everything the device transmits.
