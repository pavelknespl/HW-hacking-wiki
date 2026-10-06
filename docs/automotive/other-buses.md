# Other Vehicle Buses (LIN, CAN FD, FlexRay, Automotive Ethernet)

[CAN](can-bus.md) is the backbone, but a modern vehicle is a mix of networks chosen for cost and bandwidth. A hardware hacker mapping a car will meet cheap **LIN** subnets, higher-rate **CAN FD**, time-triggered **FlexRay**, and increasingly **Automotive Ethernet** for cameras and the backbone. Knowing which is which, and how to get on each, rounds out the [automotive](index.md) toolkit. For authorized testing on bench ECUs or vehicles you own and have immobilized safely.

## LIN (Local Interconnect Network)

**LIN** is the cheap, slow subnet for simple body electronics: window switches, mirrors, seat motors, rain sensors. It hangs off a CAN node that acts as gateway.

- **Single wire** plus ground, up to ~20 kbit/s, one master and several slaves, UART-based framing.
- Easy to tap and sniff: a LIN transceiver (or even a [UART](../protocols/uart.md) with a LIN transceiver chip) plus tools like a LIN-capable interface.
- Minimal security; frames are simple and replayable. Good for reaching comfort functions that are not on the main CAN.

## CAN FD (Flexible Data-rate)

**CAN FD** is CAN with a faster data phase and larger payloads (up to 64 bytes vs 8). Electrically the same [CAN-H/CAN-L](can-bus.md) differential pair and the same 60 ohm termination check.

- Treat it exactly like [CAN](can-bus.md): a CAN FD-capable interface (many modern USB-CAN adapters and SocketCAN support `fd on`) brings it up.

```bash
sudo ip link set can0 type can bitrate 500000 dbitrate 2000000 fd on
sudo ip link set up can0
candump can0                 # candump understands FD frames
```

- Newer vehicles use CAN FD on powertrain; the [UDS](uds-obd2.md) diagnostic layer runs over it the same way. **CAN XL** extends this further and is emerging.

## FlexRay

**FlexRay** is a high-speed (10 Mbit/s), **time-triggered**, often dual-channel bus used for safety-critical, deterministic functions (chassis, drivetrain, some ADAS) on higher-end vehicles.

- Two-wire differential per channel, but unlike CAN it is **scheduled**: nodes transmit in fixed time slots defined by a global schedule, so you cannot just inject a frame whenever you like.
- Sniffing and interacting needs a **FlexRay-capable interface** (Vector, and some specialized tools) and knowledge of the cluster's schedule (the FIBEX/communication matrix).
- Less common and harder to work with than CAN; encountered mainly on premium marques.

## Automotive Ethernet

Vehicles increasingly use **Automotive Ethernet** for high-bandwidth links (cameras, ADAS, infotainment) and as a backbone between domain controllers.

- The physical layer is usually **100BASE-T1 / 1000BASE-T1** (BroadR-Reach): full-duplex over a **single twisted pair**, not standard RJ45. You need a **media converter** (T1-to-standard-Ethernet) to tap it with a normal NIC.
- Once converted, it is ordinary Ethernet/IP: capture with Wireshark, and expect automotive protocols on top, **SOME/IP** (service-oriented middleware), **DoIP** (Diagnostics over IP, i.e. [UDS](uds-obd2.md) over Ethernet), and AVB/TSN for media.
- The attack surface is a blend of classic network testing and automotive diagnostics: DoIP exposes the same powerful UDS services over IP, so securing it matters as much as on CAN.

## Mapping a vehicle's networks

A real car has several of these, gatewayed together:

1. **Identify the buses** present (OBD-II pinout, service manual, or probing): powertrain CAN/CAN FD, body CAN, LIN subnets, maybe FlexRay, Ethernet backbone.
2. **Pick the right interface** per bus (CAN adapter, LIN transceiver, FlexRay tool, T1 media converter).
3. **The gateway ECU** bridges them and often filters what passes between, so a function unreachable from OBD-II CAN may be directly on a body CAN or LIN subnet you tap elsewhere.
4. **Diagnostics** ([UDS](uds-obd2.md)) may be reachable over CAN, CAN FD, or Ethernet (DoIP), depending on the vehicle.

## Defender's view

Segment vehicle networks with a gateway that filters and authenticates cross-domain traffic, authenticate diagnostics (DoIP/UDS) rather than relying on the weak [security-access](uds-obd2.md#security-access-0x27-and-its-weaknesses) scheme, add message authentication (SecOC) on safety-relevant CAN frames, and treat LIN and legacy CAN as untrusted. The mix of buses is only as strong as the gateway between them.

## Related pages

- [CAN Bus](can-bus.md)
- [UDS and OBD-II](uds-obd2.md)
- [Automotive overview](index.md)
- [UART](../protocols/uart.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
