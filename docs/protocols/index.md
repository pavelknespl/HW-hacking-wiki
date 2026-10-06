# Protocols and Interfaces

Chips talk to each other and to the outside world over a small set of standard buses. Learn these and you can read a device's internal conversation, drop into its debug interface, and pull data straight from its memory. Each page below explains the protocol from the wire up, then shows the wiring and commands to probe it.

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

-   __eMMC and SD__

    Reading large managed [NAND](../fundamentals/memory-types.md) as a block device.

    [:octicons-arrow-right-24: Read](emmc.md)

</div>

## Choosing where to connect

| You want to | Use |
| --- | --- |
| Read boot logs, find a shell | [UART](uart.md) |
| Halt the CPU and read all memory | [JTAG](jtag.md) / [SWD](swd.md) |
| Dump an external flash chip | [SPI](spi.md) |
| Read an EEPROM or sniff config | [I2C](i2c.md) |
| Image a large managed-NAND store | [eMMC](emmc.md) |
