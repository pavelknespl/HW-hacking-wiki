# MDIO

MDIO (Management Data Input/Output), defined in IEEE 802.3, is the small two-wire bus that a network MAC uses to configure and query an Ethernet PHY. It is niche, but when you are working on routers, switches, and any networking gear, MDIO is how the link layer is controlled, and it occasionally exposes configuration worth changing or reading.

## How it works

Two signals connect the MAC (master) to one or more PHYs (slaves):

| Signal | Role |
| --- | --- |
| [`MDC`](../reference/pin-reference.md#mdc) | Management clock, driven by the MAC |
| [`MDIO`](../reference/pin-reference.md#mdio-signal) | Bidirectional data |

Each PHY has a 5-bit address (so up to 32 PHYs per bus), and each PHY exposes a set of 16-bit **registers**. The MAC reads and writes those registers to set link speed, duplex, autonegotiation, loopback, and vendor-specific features.

- **Clause 22** is the classic frame format (5-bit PHY address, 5-bit register address).
- **Clause 45** extends addressing for 10G and higher PHYs, adding device types and a larger register space.

## What lives in the registers

The lower registers are standardized:

| Register | Contents |
| --- | --- |
| 0 | Control (reset, speed, duplex, autoneg enable, loopback) |
| 1 | Status (link up, capabilities) |
| 2, 3 | PHY identifier (maps to a specific PHY chip) |
| vendor | Vendor-specific features, LEDs, test modes |

The PHY ID registers (2 and 3) let you identify the exact PHY and pull its datasheet, which then documents the vendor registers.

## Why a hardware hacker cares

MDIO is rarely the main prize, but it has uses:

- **Identify the PHY** to understand the board and find its datasheet.
- **Diagnostic and test modes.** Vendor registers can enable loopback or test patterns useful when probing a link.
- **Configuration recovery.** Some designs stash settings reachable over MDIO.
- **Link manipulation** on a bench to force speeds or states a device would not normally accept.

## Reading it

On a Linux-based target with a shell, the kernel exposes PHY registers through `ethtool` and the MDIO ioctl interface:

```bash
ethtool eth0                      # link state, driver, PHY capabilities
# raw register access (needs the right netdev and MDIO support)
mii-tool -v eth0
phytool read eth0/0/2             # read PHY addr 0, register 2 (PHY ID high)
```

Off a running system, you drive MDC/MDIO directly from a [multi-protocol adapter](../getting-started/toolkit.md#multi-protocol-adapters) or by [bit-banging GPIO](../scripting/gpio-control.md), clocking MDC and reading MDIO per the Clause 22 frame format. A [logic analyzer](../software/logic-analyzers.md) with an MDIO decoder shows the MAC configuring the PHY at boot, which both identifies the PHY and reveals the settings the firmware applies.

## Related pages

- [Pin Reference](../reference/pin-reference.md)
- [I2C](i2c.md)
- [GPIO Control](../scripting/gpio-control.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
- [Chip Identification](../fundamentals/chip-identification.md)
- [PCB Recon](../fundamentals/pcb-recon.md)
