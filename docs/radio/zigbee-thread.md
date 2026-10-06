# Zigbee, Thread, and 802.15.4

Zigbee and Thread are low-power mesh networks built on the same **IEEE 802.15.4** radio (2.4 GHz, and sub-GHz variants). They run smart-home devices: bulbs, sensors, locks, hubs. Because they form meshes and onboard many cheap devices, their security hinges on key management, which is where the practical weaknesses are. For authorized testing of your own network.

## The shared foundation

802.15.4 defines the PHY and MAC; Zigbee and Thread are the network and application layers on top:

- **Zigbee** uses a network key shared by the mesh plus optional link keys, historically with a notorious weakness in how the network key is distributed.
- **Thread** is newer, IPv6-based (6LoWPAN), and uses stronger commissioning (a network credential established out of band), so it resists the classic Zigbee key-sniffing.

Both hop and mesh, so capturing traffic means following the network, not just a single link.

## Sniffing 802.15.4

- An **802.15.4 radio** with sniffer firmware: a TI CC2531/CC2652 dongle, an Atmel RZUSBstick, a **nRF52840**, or an **Ubertooth**-class device, feeding **Wireshark** (which decodes Zigbee and Thread).
- **KillerBee** is the classic Zigbee assessment toolkit (`zbdump`, `zbreplay`, `zbstumbler`) for capture, replay, and discovery.
- Scan channels to find the network, then capture.

## The classic Zigbee weakness

Zigbee's headline issue has been **key transport during joining**. In some configurations a device receives the network key encrypted with a well-known default **Trust Center link key** (historically a published value). If you sniff the moment a device joins, you capture the network key and can then decrypt all mesh traffic and inject commands.

So the high-value capture window is **device onboarding**:

- Sniff while a new bulb or sensor pairs to the hub.
- Extract the network key from the join exchange.
- Decrypt the mesh and craft commands (turn a lock, read a sensor).

Modern Zigbee 3.0 improved this with install codes and per-device link keys, but many deployed devices and hubs still fall back to weak defaults.

## Thread

Thread's commissioning establishes the network credential out of band (a QR/passphrase through a commissioner), so there is no network key handed over the air in the clear to sniff. Attacks shift to implementation flaws, the border router, and the application layer, rather than the join exchange. Treat it as the hardened cousin.

## The workflow

1. **Find the network and channel** with `zbstumbler` or a channel scan.
2. **Capture onboarding** of a device (trigger a pair) to grab the network key if the deployment is weak.
3. **Decrypt** the mesh in Wireshark with the recovered key.
4. **Replay or inject** application commands, or move to the [firmware](../attacks/firmware-analysis.md) of a device/hub for app-layer secrets.

## Defender's view

Use Zigbee 3.0 with install codes and unique link keys, never ship or accept the default Trust Center key, and prefer Thread or a well-configured mesh. Rotate the network key and treat onboarding as the sensitive moment it is. Assume anyone nearby can capture the join if it is weak.

## Related pages

- [RF Attacks](rf-attacks.md)
- [SDR Basics](sdr-basics.md)
- [BLE](ble.md)
- [LoRa and LoRaWAN](lora.md)
- [Firmware Analysis](../attacks/firmware-analysis.md)
