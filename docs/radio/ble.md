# Bluetooth Low Energy

BLE is the radio behind most modern wearables, locks, sensors, medical devices, and toys. It is attractive to assess because the devices are cheap, the protocol is well documented, and cost-reduced implementations frequently skip the security the standard offers. This page covers sniffing BLE and the recurring weaknesses, for authorized testing of your own devices.

## How BLE is structured

A BLE device advertises on three channels, then (optionally) connects and exchanges data over **GATT**, a tree of **services** and **characteristics**, each with a handle you can read, write, or subscribe to. Security is layered on top through pairing and bonding, and that layer is where implementations go wrong.

## Sniffing

Unlike classic Bluetooth, BLE is sniffable with cheap hardware:

- **nRF52840 dongle** with the Nordic/Wireshark sniffer firmware, or an **Ubertooth One**, follows connections and dumps packets into Wireshark.
- The hard part is catching the **connection** (the three advertising channels plus channel hopping); sniffers lock onto a connection as it forms, so you capture from the moment of connect.

Once captured, Wireshark decodes the ATT/GATT layer, revealing the services, characteristics, and the actual data exchanged.

## Enumerating a device

Without sniffing, you talk to the device directly from a host:

```bash
# scan, connect, and browse the GATT database
bluetoothctl            # scan on; connect <MAC>
# or the scriptable route
gatttool -b <MAC> --primary            # list services
gatttool -b <MAC> --characteristics    # list characteristics
nrfconnect / nRF Connect (mobile)      # GUI GATT explorer
```

`bleah`, `gattacker`, and the **nRF Connect** app are the common exploration tools.

## The recurring weaknesses

- **"Just Works" pairing.** The most common BLE pairing mode provides no man-in-the-middle protection. Many devices use it, so a sniffer that catches the pairing can derive the keys, or an attacker can MITM the connection.
- **No link-layer encryption at all.** Plenty of cheap devices skip pairing entirely and send commands in the clear. Sniff once, replay or craft commands at will.
- **Unauthenticated characteristics.** A GATT characteristic that unlocks a door or changes settings but requires no bonding or app-layer auth is directly writable by anyone. This is a frequent real finding: write the "unlock" handle and the lock opens.
- **Static or app-layer secrets** recoverable from the companion app (decompile the APK) or the device [firmware](../attacks/firmware-analysis.md).
- **Replay** of a captured command when there is no freshness.

## Attack 1 — write an unauthenticated characteristic

**Target:** a device with a sensitive GATT characteristic (unlock, config) that needs no bonding/app-layer auth.
**Gear:** any BLE host (`gatttool`/`bleak`), or nRF Connect.

```bash
gatttool -b <MAC> --primary                 # map services
gatttool -b <MAC> --characteristics         # find the writable handle
# write the "unlock" value to its handle:
gatttool -b <MAC> --char-write-req -a 0x002b -n 01
```

**Success condition:** the device acts on the write (e.g. the lock opens) with no pairing.

## Attack 2 — sniff + replay a command

**Gear:** nRF52840 dongle (Nordic sniffer fw) or Ubertooth → Wireshark.

Steps: lock the sniffer onto the connection as it forms, capture the app controlling the device, decode the ATT write in Wireshark, then replay that write with Attack 1's `--char-write-req`. Works whenever there is no link encryption or no per-command freshness.

## Attack 3 — hijack / MITM an active connection

**Gear:** **btlejack** (single BLE board) or **GATTacker** (clones the GATT profile to sit between app and device).

```bash
btlejack -s                 # scan for existing connections
btlejack -f <access_addr>   # follow, then jam/hijack that connection
```

`btlejack` can jam and take over an existing connection; GATTacker proxies it so you observe and modify traffic live. This turns a passive capture into an active test of what the device accepts.

## The workflow

1. **Recon** with nRF Connect: enumerate services/characteristics, note which are writable and whether they need bonding → Attack 1.
2. **Sniff** a legitimate interaction to see which handle does what and whether data is encrypted → Attack 2.
3. **Hijack** live where you need to modify in-flight → Attack 3.
4. **Pull the companion app and firmware** when the on-air protocol has an app-layer secret; reverse them to recover it.

## Defender's view

Use **LE Secure Connections** pairing (not Just Works) with MITM protection, require bonding for any sensitive characteristic, add app-layer authentication and anti-replay freshness on commands, and never rely on the obscurity of a custom GATT layout. Assume every characteristic is enumerable and every unencrypted packet is readable.

## Related pages

- [RF Attacks](rf-attacks.md)
- [SDR Basics](sdr-basics.md)
- [nRF52 APPROTECT](../mcu/nrf52-approtect.md)
- [Firmware Analysis](../attacks/firmware-analysis.md)
- [Flipper Zero](../software/flipper-zero.md)
