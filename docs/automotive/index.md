# Automotive

Vehicles are rolling computer networks. Dozens of electronic control units (ECUs) talk over shared buses, and the dominant one is CAN. For a hardware hacker the car is an unusually rich target: physical bus access is easy through the OBD-II port, the protocols are well documented, and the security model was designed in an era that assumed nobody hostile would ever be on the bus.

<div class="grid cards" markdown>

-   __CAN Bus__

    The backbone network: frames, wiring, and how to read and write it.

    [:octicons-arrow-right-24: Read](can-bus.md)

-   __UDS and OBD-II__

    The diagnostic protocols that ride on CAN: services, sessions, and security access.

    [:octicons-arrow-right-24: Read](uds-obd2.md)

-   __Other Vehicle Buses__

    LIN, CAN FD, FlexRay, and Automotive Ethernet (DoIP/SOME-IP).

    [:octicons-arrow-right-24: Read](other-buses.md)

</div>

## Why automotive is its own topic

CAN is not like the on-board buses in the rest of this wiki. It is a multi-master broadcast network with no addressing and no authentication by default, so every node hears every message and any node can transmit. On top of CAN sit diagnostic protocols (OBD-II for emissions-related standard data, UDS for the full manufacturer diagnostic surface) that expose firmware flashing, actuator control, and memory access to anyone who can speak them.

The attack surface therefore spans several layers:

- **The physical bus**: tapping CAN at the OBD-II connector or directly on a harness.
- **The protocol**: reading and injecting frames, reverse engineering what each arbitration ID means.
- **The diagnostics**: UDS sessions that unlock programming, read memory, and drive outputs, often gated only by a weak "security access" challenge.
- **The ECUs themselves**: once reached over UDS, an ECU is a [microcontroller](../mcu/index.md) with firmware you can dump and analyze like any other.

## Tooling

- **CAN interface**: a USB-to-CAN adapter (CANable/candleLight, Kvaser, PCAN, or a cheap MCP2515 board) brings CAN onto your workstation as a SocketCAN interface on Linux.
- **can-utils**: the standard Linux toolkit (`candump`, `cansend`, `cangen`, `isotpdump`). See [CAN Bus](can-bus.md).
- **Higher-level tools**: `python-can`, `caringcaribou` (UDS and diagnostics discovery), and Wireshark with its CAN/ISO-TP dissectors.

!!! danger "A car is a safety-critical system"
    Injecting frames can disable brakes assistance, trigger airbags, or make a vehicle behave unpredictably. Only ever do this on a bench ECU, a vehicle you own and have immobilized safely, or a dedicated test rig. Never experiment on a moving vehicle or public road.
