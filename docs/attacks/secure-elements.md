# Secure Elements

A secure element is a dedicated chip (or an on-die subsystem) built to store keys and perform cryptography while resisting the very attacks the rest of this wiki describes. You meet them as crypto-authentication chips (Microchip ATECC), SIM/eSIM and smart cards, hardware wallets, TPMs, and the secure enclaves inside SoCs. Their whole job is to make [key extraction](firmware-analysis.md) hard, which changes how you approach a device that uses one. For authorized testing.

## What they are and why they exist

A secure element holds a secret (a private key, a symmetric key) and offers operations that *use* the key without ever exposing it: "sign this", "authenticate this", "derive from this". The key is generated or injected once and is designed never to leave the chip. Common examples:

- **ATECC508A/608** and similar crypto-auth chips on [I2C](../protocols/i2c.md): accessory/consumable authentication, key storage for IoT.
- **SIM / eSIM / smart cards**: carrier and payment credentials in a tamper-resistant package.
- **TPM** and SoC **secure enclaves** (TrustZone-backed): platform keys and attestation.
- **Hardware wallets**: a secure element guarding a cryptocurrency private key.

They typically include the physical countermeasures from the [invasive](decapping.md) and [side-channel](side-channel.md) pages: shields, sensors, constant-time and masked crypto.

## What they change for the attacker

When a device stores its secret in a secure element rather than plain [flash](spi-flash-dump.md), the easy wins disappear:

- A [firmware dump](firmware-extraction.md) no longer contains the key (unlike the usual [plaintext-secret sweep](firmware-analysis.md)); it contains only calls to the secure element.
- You cannot read the key over the bus, because it never crosses the bus.

So the attack shifts from "find the key" to one of:

1. **Attack the protocol around it.** The secure element proves possession of a key, but how does the host *use* that proof? If the host checks only "did the chip answer", you may **replay** a captured valid response, or **bypass the check in the host firmware** entirely (patch out the "is this accessory genuine" branch). This is often the weakest link and needs no attack on the chip at all.
2. **Attack the host-to-element channel.** [Sniff the I2C](../protocols/i2c.md) exchange to understand the challenge-response; if the channel is unencrypted and the host trusts a simple value, you may clone that behavior.
3. **Attack the element itself.** [Side-channel](side-channel.md) (power/EM analysis of the crypto operation) and [fault injection](fault-injection.md) have recovered keys from specific secure elements and wallets, and [invasive](decapping.md) methods against the die. This is the hard path and depends entirely on the chip's countermeasures and known research for that exact part.

## The practical lesson

The secure element is usually not the weak point; the **system around it** is. Repeatedly, products with a strong secure element are defeated because the host firmware:

- Checks the authentication result sloppily (ignores it, or accepts a replay).
- Keeps a derived secret in plain RAM/flash after the element releases it.
- Uses the element for authentication but not for the actual data protection.

So the workflow is: [extract and reverse the host firmware](firmware-analysis.md), find where it talks to the secure element, and see whether the decision it makes from the element's answer can be forged, replayed, or skipped, before ever considering an attack on the element's silicon.

## Defender's view

Use a secure element for what it is good at, and close the system gaps: verify its responses properly (bind them to a fresh challenge, act on the result in a [fault-resistant](fault-injection.md#defenders-view) way), use its encrypted-session features so the bus is not plaintext, keep derived secrets inside it rather than copying them to RAM, and choose a part with countermeasures matching your threat model. The element protects the key; only your firmware can protect the decision.

## Related pages

- [Side-Channel Attacks](side-channel.md)
- [Fault Injection and Glitching](fault-injection.md)
- [Decapping and Invasive Attacks](decapping.md)
- [I2C](../protocols/i2c.md)
- [Firmware Analysis](firmware-analysis.md)
