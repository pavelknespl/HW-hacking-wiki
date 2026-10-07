# Protocols and Interfaces

Chips talk to each other and to the outside world over a small set of standard buses. Learn these and you can read a device's internal conversation, drop into its debug interface, and pull data straight from its memory. Each page below explains the protocol from the wire up, then shows the wiring and commands to probe it.

## Core buses

The handful you meet on almost every board. Learn these first.

<div class="grid cards" markdown>

-   __UART__

    The [serial console](uart.md). The single most productive interface on most devices.

    [:octicons-arrow-right-24: Read](uart.md)

-   __JTAG__

    Full CPU debug: halt, read and write memory, dump internal flash.

    [:octicons-arrow-right-24: Read](jtag.md)

-   __SWD__

    ARM's two-wire debug interface, standard on Cortex-M.

    [:octicons-arrow-right-24: Read](swd.md)

-   __I2C__

    The two-wire bus for EEPROMs, sensors, and configuration chips.

    [:octicons-arrow-right-24: Read](i2c.md)

-   __SPI__

    The fast four-wire bus that connects most external flash.

    [:octicons-arrow-right-24: Read](spi.md)

</div>

## Memory and peripheral

Buses that reach mass storage and attached devices.

<div class="grid cards" markdown>

-   __eMMC and SD__

    Reading large managed [NAND](../fundamentals/memory-types.md) as a block device.

    [:octicons-arrow-right-24: Read](emmc.md)

-   __USB__

    Descriptors, enumeration, DFU, and device emulation with Facedancer.

    [:octicons-arrow-right-24: Read](usb.md)

-   __1-Wire__

    Single-line sensors, ID chips, and consumable authentication.

    [:octicons-arrow-right-24: Read](one-wire.md)

</div>

## System and networking

Platform housekeeping buses and industrial serial links.

<div class="grid cards" markdown>

-   __MDIO__

    Configuring and reading Ethernet PHY registers.

    [:octicons-arrow-right-24: Read](mdio.md)

-   __LPC__

    The legacy bus behind TPM sniffing and BIOS flash.

    [:octicons-arrow-right-24: Read](lpc.md)

-   __RS-485 and Modbus__

    The industrial serial bus and its unauthenticated protocol.

    [:octicons-arrow-right-24: Read](rs485-modbus.md)

-   __SMBus and PMBus__

    I2C with a rulebook: system and power management.

    [:octicons-arrow-right-24: Read](smbus-pmbus.md)

</div>

## Debug extensions

Program and trace interfaces beyond the core JTAG/SWD pair.

<div class="grid cards" markdown>

-   __MCU Debug (SWIM/UPDI/PDI/debugWIRE)__

    Single-wire program and debug interfaces on STM8 and AVR.

    [:octicons-arrow-right-24: Read](mcu-debug.md)

-   __ARM Trace (SWO/ITM/ETM)__

    Watch a core execute in real time, without halting it.

    [:octicons-arrow-right-24: Read](trace.md)

</div>

For vehicle networks (CAN, UDS, OBD-II, LIN, FlexRay, automotive Ethernet), see the dedicated [Automotive](../automotive/index.md) section.

## Choosing where to connect

| You want to | Use |
| --- | --- |
| Read boot logs, find a shell | [UART](uart.md) |
| Halt the CPU and read all memory | [JTAG](jtag.md) / [SWD](swd.md) |
| Dump an external flash chip | [SPI](spi.md) |
| Read an EEPROM or sniff config | [I2C](i2c.md) |
| Image a large managed-NAND store | [eMMC](emmc.md) |
