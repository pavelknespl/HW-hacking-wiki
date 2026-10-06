# 1-Wire

1-Wire is a minimalist bus that carries both data and, optionally, power on a single line plus ground. It shows up in temperature sensors, identification chips, small EEPROMs, battery authentication, and consumables (cartridges, accessories) that a device checks to decide whether to work. That last use, authentication of consumables, is what makes 1-Wire interesting to a hardware hacker.

## How it works

A single data line, pulled high by a resistor, with one master and one or more slaves:

- The master drives timing by pulling the line low for defined durations; bits are encoded by how long the line stays low.
- Slaves can be powered by the data line itself (**parasitic power**), so a device may need only one wire plus ground.
- Every 1-Wire slave has a unique factory **64-bit ROM ID** (an 8-bit family code, a 48-bit serial, and a CRC). The master enumerates devices by their IDs.

Timing is strict but slow, which makes 1-Wire easy to bit-bang and easy to [sniff with a logic analyzer](../software/logic-analyzers.md).

## Where you meet it

| Device family | Example use |
| --- | --- |
| Temperature sensors (DS18B20) | Environmental monitoring |
| ID / serial chips (DS2401) | Board or cable identification |
| EEPROM (DS24xx) | Small config, calibration |
| Authentication (DS28E/SHA family) | Consumable and accessory authentication |

## Reading the bus

With a [Raspberry Pi](../scripting/gpio-control.md) the kernel has a 1-Wire driver (`w1-gpio`): enable it and devices appear under `/sys/bus/w1/devices/`, each directory named by its ROM ID. Reading a sensor is then a file read.

With a [multi-protocol adapter](../getting-started/toolkit.md#multi-protocol-adapters) or a dedicated 1-Wire master (DS2480B based), you issue the ROM and memory commands directly. A [logic analyzer](../software/logic-analyzers.md) with the 1-Wire decoder reveals the ROM IDs and the command/data exchange when you sniff a device talking to its host.

## The authentication angle

The common security use of 1-Wire is consumable authentication: a printer, medical device, or accessory contains a 1-Wire chip, and the host refuses to operate unless the chip answers a challenge correctly. Analysis proceeds like any protocol study:

- **Simple ID check.** If the host only reads the ROM ID or an EEPROM value and compares it, that value is trivially cloneable onto a blank 1-Wire part. These are weak by design.
- **SHA-based challenge-response (DS28E/DS1961S style).** The chip holds a secret and answers a challenge with a MAC. This is stronger. Weaknesses come from recovered or weak secrets, implementation flaws in the host's verification, or [side-channel](../attacks/side-channel.md) leakage during the MAC computation.

Either way, the workflow is to sniff the host-to-chip exchange, understand whether it is a plain read or a cryptographic challenge, and then either clone the value or attack the secret.

## Related pages

- [Pin Reference](../reference/pin-reference.md)
- [I2C](i2c.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
- [GPIO Control](../scripting/gpio-control.md)
- [Side-Channel Attacks](../attacks/side-channel.md)
- [The Toolkit](../getting-started/toolkit.md)
