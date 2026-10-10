# CAN Bus

CAN (Controller Area Network) is the nervous system of almost every modern vehicle and a lot of industrial equipment. It is a two-wire differential bus that any node can transmit on and every node receives. Understanding its frames and getting a Linux box onto the bus is the foundation for all [automotive](index.md) work.

## How it works

CAN is a multi-master broadcast bus with no node addresses. Instead, every message carries an **arbitration ID**, and that ID serves two purposes: it identifies the kind of message, and it sets priority (lower ID wins when two nodes transmit at once).

The wiring:

| Signal | Role |
| --- | --- |
| [`CAN-H`](../reference/pin-reference.md#can-h) | Differential high |
| [`CAN-L`](../reference/pin-reference.md#can-l) | Differential low |
| [`GND`](../reference/pin-reference.md#gnd) | Reference (shared) |

A bit is encoded as the voltage difference between CAN-H and CAN-L. The bus is terminated with 120 ohm resistors at each end; a healthy bus reads about 60 ohm across H and L with power off, which is a quick way to confirm you have found CAN.

Common bit rates: **500 kbit/s** (powertrain, the most common), 250 kbit/s (body/comfort), and 125 kbit/s. Guess wrong and you see only errors, so bit rate is the first thing to establish.

### Frame shape

- **Arbitration ID**: 11-bit (standard) or 29-bit (extended).
- **DLC**: data length, 0 to 8 bytes (classic CAN) or up to 64 (CAN FD).
- **Data**: the payload bytes.

There is no source address, no authentication, and no encryption. Any node can send any ID. That design is the root of most automotive security findings.

## Finding and tapping the bus

- **OBD-II connector.** Legally mandated on cars since the mid 2000s. Pins 6 (CAN-H) and 14 (CAN-L) carry the primary powertrain CAN. This is the easy, non-invasive tap point.
- **Directly on a harness or ECU.** For buses not bridged to OBD-II (many comfort/body networks are gatewayed off), you tap CAN-H/CAN-L on the wiring or at an ECU connector. Confirm with the 60 ohm termination check.

## Getting on the bus with SocketCAN

On Linux a USB-to-CAN adapter appears as a network interface. Bring it up at the right bit rate:

```bash
sudo ip link set can0 type can bitrate 500000
sudo ip link set up can0
```

Then use [can-utils](index.md#tooling):

```bash
candump can0                    # watch all traffic
candump -l can0                 # log to a file for offline analysis
cansend can0 123#DEADBEEF       # send ID 0x123 with 4 data bytes
cangen can0                     # generate random frames (bench testing only)
```

A `listen-only` mode exists for passive monitoring without acknowledging frames, which avoids disturbing a live bus:

```bash
sudo ip link set can0 type can bitrate 500000 listen-only on
```

## Reverse engineering frames

Nothing tells you what ID `0x3B2` means; you work it out by correlation:

1. **Capture a baseline** with `candump -l` while the vehicle idles.
2. **Perform one action** (turn the wheel, press a button, change RPM) and capture again.
3. **Diff the captures** to find which ID and which byte changed with the action.
4. **Model the signal**: watch how bytes track the physical value (an RPM counter rises monotonically, a switch flips a bit).

Tools accelerate this: **Kayak** and **SavvyCAN** give a GUI for diffing and plotting signals, and **cantools** parses and builds DBC files (the database format that maps IDs and bytes to named signals).

## Replay and injection

Because there is no authentication, a captured frame can be replayed:

```bash
canplayer -I capture.log         # replay a candump log
```

Replaying a captured "unlock" or "window down" frame often reproduces the action. Injection goes further: flooding a high-priority ID can starve lower-priority traffic (a bus-level denial of service), and crafted frames can command actuators directly. This is exactly why the UDS layer matters, and why real ECUs increasingly add message authentication.

### Attack — CAN injection (the real car-theft technique)

**Target:** vehicles where a peripheral ECU sits on a CAN segment that also reaches immobilizer/door/start logic, with no message authentication.
**Why it works:** because any node can send any ID and nothing authenticates frames, a thief reaches an exposed CAN wire behind an easy-access component (famously the **headlight connector**) and injects the exact frames the smart-key ECU would send — "key is valid, unlock, allow start".
**Gear:** a small injector (an MCU + CAN transceiver, sometimes disguised in a speaker/JBL housing) spliced onto the harness.

Steps:

1. [Tap CAN](#finding-and-tapping-the-bus) at an accessible point (headlight, mirror, bumper sensor wiring).
2. Capture or already-know the valid-key / unlock / start frames for that model (reverse-engineered once per model per [frame RE](#reverse-engineering-frames)).
3. Inject them with `cansend`/a standalone injector; the car unlocks and starts with no key present.

**Mitigation:** segment/gateway sensitive buses, authenticate critical messages (SecOC), and detect rogue nodes. See [UDS](uds-obd2.md).

## Where to go next

Reading raw CAN tells you what the car broadcasts. To reach flashing, memory, and actuator routines, you move up to the diagnostic layer. See [UDS and OBD-II](uds-obd2.md). Once you reach an ECU's firmware through UDS, it is a [microcontroller target](../mcu/index.md) like any other.

## Related pages

- [Pin Reference](../reference/pin-reference.md)
- [UDS and OBD-II](uds-obd2.md)
- [Automotive overview](index.md)
- [MCU Targets](../mcu/index.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
- [Firmware Analysis](../attacks/firmware-analysis.md)
