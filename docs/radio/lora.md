# LoRa and LoRaWAN

LoRa is a long-range, low-power sub-GHz radio; **LoRaWAN** is the network protocol built on it for wide-area IoT (utility meters, agriculture, asset tracking, smart city sensors). The range (kilometers) and the long device lifetimes make key management the central security question. For authorized testing of your own devices and networks.

## LoRa versus LoRaWAN

- **LoRa** is just the physical layer: a chirp-spread-spectrum modulation in the 433/868/915 MHz bands. Capturing it needs a LoRa-capable radio, not a plain [RTL-SDR](sdr-basics.md) (though an SDR with a LoRa decoder, or gr-lora, can work).
- **LoRaWAN** adds the MAC and security: devices (end nodes) talk to gateways, which forward to a network server. Security rests on two AES-128 root keys and session keys derived during join.

## The keys that matter

LoRaWAN 1.0.x uses:

- **AppKey** (root), from which session keys are derived at join.
- **NwkSKey** and **AppSKey** (session), protecting the MAC layer and the application payload respectively.

Two join methods:

- **OTAA** (Over-The-Air Activation): the device joins with the AppKey and derives fresh session keys. Preferred.
- **ABP** (Activation By Personalization): session keys are hardcoded into the device. Weaker, because the keys are static and often recoverable.

## Where it breaks

- **ABP devices with hardcoded keys.** Extract the [firmware](../attacks/firmware-analysis.md) of an ABP node and the NwkSKey/AppSKey are usually right there, letting you decrypt and forge that device's traffic.
- **Keys in firmware generally.** Even OTAA devices store the AppKey; a [flash dump](../attacks/spi-flash-dump.md) often yields it, after which you can impersonate the device.
- **Weak or default keys.** Devices shipped with predictable AppKeys, or vendors reusing one key across a fleet.
- **Replay and ABP counter handling.** Poor frame-counter enforcement allows replay of captured uplinks.
- **1.0.x protocol weaknesses** that 1.1 addressed (key separation, join nonce handling); many deployed devices are still 1.0.x.

## Capturing and testing

- A **LoRa radio** (a gateway concentrator, or a LoRa dev board as a sniffer) plus tools like **gr-lora**, **LoRaPWN**-style scripts, or **ChirpStack** as a test network server.
- Capture uplinks and joins, then, with keys recovered from firmware, decrypt and optionally forge messages against your test server.

## The workflow

1. **Identify the band and device**; determine OTAA vs ABP.
2. **Dump the device firmware** and recover the AppKey (OTAA) or session keys (ABP). See [Firmware Extraction](../attacks/firmware-extraction.md).
3. **Capture** join and uplink traffic.
4. **Decrypt and forge** against a controlled network server to test what the backend accepts.

## Defender's view

Use OTAA, not ABP; use LoRaWAN 1.1 where possible; store the AppKey in a [secure element](../attacks/secure-elements.md) rather than plain flash; use unique per-device keys; and enforce frame counters strictly to block replay. The long device lifetime means a leaked key is a long-lived problem, so protect it at rest.

## Related pages

- [SDR Basics](sdr-basics.md)
- [RF Attacks](rf-attacks.md)
- [Firmware Extraction](../attacks/firmware-extraction.md)
- [Secure Elements](../attacks/secure-elements.md)
- [Zigbee, Thread, and 802.15.4](zigbee-thread.md)
