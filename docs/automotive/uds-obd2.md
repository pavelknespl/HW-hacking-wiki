# UDS and OBD-II

On top of raw [CAN](can-bus.md) ride the diagnostic protocols. **OBD-II** is the standardized, emissions-focused subset every car must expose. **UDS** (Unified Diagnostic Services, ISO 14229) is the full manufacturer diagnostic surface, and it is where the real capability lives: reading and writing memory, flashing firmware, driving actuators, and clearing immobilizers. If you are building an OBD-II or UDS tool, this is the protocol map.

## The transport: ISO-TP

CAN frames carry at most 8 data bytes, but diagnostic messages are longer, so they use **ISO-TP** (ISO 15765-2) to segment a message across multiple CAN frames. A request goes to one arbitration ID (commonly `0x7E0` for the engine ECU), responses come back on another (`0x7E8`). On Linux the kernel implements ISO-TP:

```bash
# send a UDS request and read the response over ISO-TP
echo "22 F1 90" | isotpsend -s 7E0 -d 7E8 can0    # read VIN (DID 0xF190)
isotprecv -s 7E8 -d 7E0 can0
```

`python-can` plus `udsoncan`, or the discovery tool **caringcaribou**, wrap this cleanly.

## OBD-II in one table

OBD-II uses service IDs (modes) `0x01` to `0x0A`. The useful ones:

| Mode | Purpose |
| --- | --- |
| `0x01` | Current data (live sensor PIDs: RPM, speed, coolant temp) |
| `0x03` | Read stored diagnostic trouble codes (DTCs) |
| `0x04` | Clear DTCs |
| `0x09` | Vehicle info (VIN, calibration IDs) |

OBD-II is read-mostly and standardized, so it is safe exploration territory and the basis of every generic scan tool.

## UDS: the services that matter

UDS services are where diagnostics becomes powerful. Key service IDs (SIDs):

| SID | Service | Why it matters |
| --- | --- | --- |
| `0x10` | Diagnostic Session Control | Switch to extended/programming sessions that unlock other services |
| `0x27` | Security Access | The seed/key challenge gating protected services |
| `0x22` / `0x2E` | Read / Write Data By Identifier | Read and write configuration (DIDs) |
| `0x23` / `0x3D` | Read / Write Memory By Address | Direct memory access |
| `0x31` | Routine Control | Run built-in routines (erase, self-test, actuator drive) |
| `0x34` / `0x36` / `0x37` | Request/Transfer/Exit Download | The firmware flashing sequence |
| `0x11` | ECU Reset | Reboot the ECU |

A typical privileged flow: enter a programming session (`0x10 0x02`), pass security access (`0x27`), then use memory or download services.

## Security Access (0x27) and its weaknesses

Protected services are gated by a seed/key exchange:

1. The tester requests a **seed** (`0x27 0x01`); the ECU returns a random value.
2. The tester computes a **key** from the seed with a secret algorithm and sends it (`0x27 0x02`).
3. If the key matches, the session unlocks.

The whole scheme rests on the secrecy of the key algorithm, and that is its weakness. Common findings on real ECUs:

- **Weak or fixed algorithms.** The seed-to-key function is sometimes a trivial transform, or the same constant key is accepted.
- **Algorithm recovered from firmware.** Once you [dump the ECU firmware](../attacks/firmware-extraction.md) and [reverse](../reverse-engineering/firmware-workflow.md) the `0x27` handler, you have the algorithm and can compute keys at will. Many published ECU unlocks come from exactly this.
- **No rate limiting or weak delay.** Some ECUs allow rapid guessing; a short key space becomes brute-forceable.
- **Replayable seeds.** If the seed is not truly random, a captured valid key can be reused.

Concrete recipe:

```bash
# 1. enter programming session, then probe 0x27 and brute a short/weak key space
caringcaribou uds security_seed 0x7E0 0x7E8 1 2 -d 1   # request seeds, observe
# 2. if seed is constant or key algo is trivial, confirm the seed->key mapping
#    by collecting pairs; if the ECU accepts a fixed key, send it directly:
echo "27 02 <key-bytes>" | isotpsend -s 7E0 -d 7E8 can0
# 3. otherwise dump the ECU firmware and reverse the 0x27 handler to get the algo
```

The durable path is step 3: recover the seed-to-key function from [firmware](../attacks/firmware-extraction.md) once, then compute keys for every ECU of that type.

## Discovery with caringcaribou

To map an unknown ECU's diagnostic surface:

```bash
# find which arbitration IDs answer diagnostic requests
caringcaribou discovery

# enumerate supported UDS services on a found ID
caringcaribou uds discovery

# dump DIDs, probe services, test security access
caringcaribou uds dump_dids 0x7E0 0x7E8
```

This reveals which sessions, DIDs, and routines an ECU exposes, which is the recon step before any targeted work.

## From UDS to firmware

The flashing services (`0x34`/`0x36`/`0x37`) and memory services (`0x23`) are an extraction path: if you can unlock security access, you may read the ECU's memory straight out over UDS, no desoldering. Where security access blocks you, fall back to physically dumping the ECU's [microcontroller](../mcu/index.md) flash, often defeating its [read-out protection](../mcu/stm32-rdp.md) with [glitching](../attacks/fault-injection.md).

## Related pages

- [Pin Reference](../reference/pin-reference.md)
- [CAN Bus](can-bus.md)
- [Automotive overview](index.md)
- [MCU Targets](../mcu/index.md)
- [Firmware Extraction](../attacks/firmware-extraction.md)
- [Firmware RE Workflow](../reverse-engineering/firmware-workflow.md)
