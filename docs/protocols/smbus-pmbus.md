# SMBus and PMBus

SMBus and PMBus are [I2C](i2c.md) with extra rules. If you understand I2C, you are most of the way there: these are standardized command sets and conventions layered on the same two wires, used for system management (SMBus) and power management (PMBus). You meet them on PC motherboards, server management, smart batteries, and power supplies, and they occasionally expose configuration or identity worth reading.

## SMBus (System Management Bus)

**SMBus** is a tightly specified subset of [I2C](i2c.md) (fixed-ish clock range, defined timeouts, a standard command/response structure, optional PEC error-checking byte). It is the management bus inside PCs and servers.

Where you find it and what it carries:

- **SPD EEPROMs on memory modules**: the DIMM's SPD data (timings, size, manufacturer) is an SMBus-readable EEPROM. `decode-dimms` on Linux reads it.
- **Smart batteries**: the Smart Battery Data spec rides on SMBus; a laptop battery reports charge, cycle count, and design capacity, and the battery's fuel-gauge chip is addressable.
- **Hardware monitoring**: temperature/voltage/fan sensors (the `lm-sensors` world) hang off SMBus/I2C.
- **Server management (IPMI/BMC)** uses SMBus-derived buses to reach sensors and FRU EEPROMs.

Because it is electrically I2C, you probe it exactly as on the [I2C page](i2c.md): `i2cdetect` to enumerate, `i2cget`/`i2cdump` to read. On a PC the buses appear via the `i2c-dev` kernel interface.

```bash
i2cdetect -l                 # list SMBus/I2C adapters
decode-dimms                 # read SPD EEPROMs over SMBus
sensors                      # hardware monitor readings
```

## PMBus (Power Management Bus)

**PMBus** is SMBus specialized for talking to power-conversion hardware: digital VRMs, power supplies, point-of-load regulators, battery chargers. It standardizes commands to set and read output voltages, current limits, fault thresholds, and telemetry.

For a hardware hacker, PMBus is interesting because it lets you **read and sometimes change power parameters** over the bus:

- **Telemetry**: read live voltage, current, temperature, and fault status from a regulator or PSU.
- **Configuration**: many PMBus devices allow setting the output voltage and limits over the bus. On research hardware this connects to [fault injection](../attacks/fault-injection.md) ideas, commanding a regulator to under- or over-volt a target can itself be a glitch vector, and misconfiguring limits can stress a board.

```bash
# PMBus devices expose attributes via the kernel pmbus/hwmon drivers,
# or you can issue raw SMBus commands with i2c tools once you know the map
i2cdump -y <bus> <addr>       # survey a device's registers (read-only, carefully)
```

!!! warning "Writing to power hardware is risky"
    Changing a PMBus regulator's output or limits can destroy the board it powers (over-voltage) or is a deliberate [glitch](../attacks/fault-injection.md). Only write to power devices on hardware you are prepared to damage and understand.

### Attack — undervolt a CPU over the voltage bus (VoltPillager)

**Target:** a CPU whose core voltage is set by a regulator over **SVID** or PMBus (most x86 boards).
**Gear:** a cheap MCU (Teensy-class) wired to the SVID/PMBus clock+data between CPU and VRM.
**Why it works:** the regulator obeys whatever voltage-ID commands it receives; injecting them from outside the CPU undervolts the core on demand — a hardware [fault-injection](../attacks/fault-injection.md#software-triggered-fault-injection-no-bench-rig) primitive that bypasses software undervolt locks (the fix for Plundervolt).

Steps: find the SVID clock/data lines near the VRM, sniff the normal VID traffic to learn the encoding, then inject a brief deep-undervolt command synchronized to the target crypto operation; detect a faulted result and iterate (same loop as [voltage glitching](../attacks/fault-injection.md#the-workflow)).

### Attack — overvolt a fast charger (BadPower)

**Target:** USB-PD fast chargers whose firmware is updatable over the data lines.
**Idea:** reflash the charger's MCU so it negotiates a higher voltage than the sink can tolerate, over-volting and physically damaging the attached device. Reads/writes ride the same I2C/PD comms the charger already uses. Destructive by design — bench only.

## The practical point

Treat SMBus/PMBus as "I2C with a rulebook and a known command set". Enumerate and read with the [I2C](i2c.md) tooling, consult the SMBus/PMBus spec or the specific chip datasheet for the command meanings, and remember that the readable identity and telemetry (SPD, battery data, power rails) is often the useful part, while writing, especially to PMBus power devices, is where you can do damage or, deliberately, induce faults.

## Related pages

- [I2C](i2c.md)
- [RS-485 and Modbus](rs485-modbus.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
- [PCB Recon](../fundamentals/pcb-recon.md)
