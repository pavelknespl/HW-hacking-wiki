# RS-485 and Modbus

Step away from consumer gadgets and into industrial, building-automation, and energy equipment, and you meet **RS-485** carrying **Modbus**. RS-485 is a rugged serial bus for long cable runs and many devices; Modbus is the simple, ubiquitous, and famously unauthenticated protocol that usually rides on it. For a hardware hacker assessing industrial/OT equipment, this pairing is the [UART](uart.md) of that world. For authorized testing only; OT systems control physical processes.

## RS-485: the physical layer

RS-485 is a **differential**, multi-drop serial bus:

| Signal | Role |
| --- | --- |
| `A` / `D-` | Inverting line |
| `B` / `D+` | Non-inverting line |
| `GND` | Common reference |

- **Differential** like [CAN](../automotive/can-bus.md), so it tolerates long runs (hundreds of meters) and noise, with 120 ohm termination at the ends.
- **Multi-drop**: many nodes (typically up to 32) on one pair, usually **half-duplex** (one talker at a time), so it needs an addressing/turn-taking protocol on top, which is where Modbus comes in.
- Electrically it is just an asynchronous serial frame (often 9600 or 19200 baud, 8N1), so a USB **RS-485 adapter** presents it as a normal serial port.

## Modbus: the protocol

Modbus is a master/slave request-response protocol with a tiny, readable structure:

- A **slave address**, a **function code**, and data, protected only by a CRC.
- **Function codes** read and write the device's data model: coils (bits), discrete inputs, holding registers (16-bit), input registers. For example read holding registers (0x03), write single register (0x06), write multiple registers (0x10).
- Variants: **Modbus RTU** (binary, over RS-485/serial), **Modbus ASCII**, and **Modbus TCP** (the same model over Ethernet, port 502).

Crucially, standard Modbus has **no authentication and no encryption**. Any node that can speak on the bus can read and write any register, which in OT terms means read sensor values and command actuators.

## Assessing it

1. **Get on the bus**: a USB RS-485 adapter on the A/B pair plus ground; match the baud and framing (sniff with a [logic analyzer](../software/logic-analyzers.md) if unknown, RS-485 decoders exist).
2. **Enumerate**: scan slave addresses and probe function codes. Tools: `mbpoll`, `modpoll`, the `pymodbus` library, or Metasploit's Modbus modules.

    ```bash
    # read 10 holding registers from slave 1
    mbpoll -m rtu -a 1 -b 19200 -t 4 -r 1 -c 10 /dev/ttyUSB0
    ```

3. **Map the registers**: like [reverse engineering CAN](../automotive/can-bus.md#reverse-engineering-frames), change one input or output and watch which register moves to learn the device's data model.
4. **Read and (carefully) write**: reading is passive; writing a register commands the device, which in OT can move a physical actuator. Do this only on equipment you are authorized to manipulate and that is safe to actuate.

!!! danger "OT is physical and safety-critical"
    Writing Modbus registers can open valves, start motors, or trip breakers. Only test on isolated bench equipment or with explicit authorization and safety measures. A careless write on a live industrial bus can cause real-world harm.

## Defender's view

Modbus has no built-in security, so it must be protected by the network around it: segment and firewall OT networks, never expose Modbus TCP (port 502) to untrusted networks, use gateways that enforce access control, and monitor for unexpected writes. Where available, use authenticated variants (Modbus/TCP Security) or wrap the link in a VPN. Treat bus access as equivalent to control of the process.

## Related pages

- [UART](uart.md)
- [CAN Bus](../automotive/can-bus.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
- [SMBus and PMBus](smbus-pmbus.md)
- [Serial Consoles](../software/serial-consoles.md)
