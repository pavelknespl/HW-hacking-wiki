# Tigard

Tigard is an open-source multi-protocol interface board built around an FTDI **FT2232H**. It breaks out the common low-speed debug and memory buses, [UART](../protocols/uart.md), [JTAG](../protocols/jtag.md), [SWD](../protocols/swd.md), [SPI](../protocols/spi.md), and [I2C](../protocols/i2c.md), onto clearly labeled headers with a switchable target voltage. For a hardware hacker it is the reliable, boring workhorse: one cable that connects your host to almost any serial bus you meet, with the level translation handled.

## Why it is handy

- **One board, many buses.** The FT2232H has two independent channels, so Tigard can be a serial console on one and a JTAG or SPI adapter on the other.
- **Voltage selection.** A switch and sense line set the I/O voltage (commonly 1.8 / 3.3 V and a sensed target level), so you do not fry a 1.8 V target. See [Level Shifting](../fundamentals/level-shifting.md).
- **Standard silicon.** Because it is FTDI, it works with tools you already use: `flashrom`, `OpenOCD`, `pyftdi`, and `openFPGALoader`, with no special drivers.

## Common uses

```bash
# serial console on the UART channel
screen /dev/ttyUSB0 115200

# dump a SPI flash via the SPI header
flashrom -p ft2232_spi:type=2232H,port=A -r flash.bin

# JTAG/SWD debugging through OpenOCD
openocd -f interface/ftdi/tigard.cfg -f target/<target>.cfg
```

The same board drives [flashrom](flashrom.md) for [SPI flash dumping](../attacks/spi-flash-dump.md), [OpenOCD](openocd.md) for [JTAG](../protocols/jtag.md)/[SWD](../protocols/swd.md) access, and [openFPGALoader](openfpgaloader.md) for [FPGA](../fpga/index.md) work.

## Where it fits

Tigard is the go-to once you know the pinout and the voltage. It is simpler and faster than a [Glasgow](glasgow.md) for a known bus, and unlike a [JTAGulator](jtagulator.md) it is for *using* an interface, not discovering one. When you need to find the pinout first, start with a JTAGulator or Glasgow, then switch to Tigard for the actual work.

## Related pages

- [UART](../protocols/uart.md)
- [JTAG](../protocols/jtag.md)
- [SWD](../protocols/swd.md)
- [SPI](../protocols/spi.md)
- [Logic Levels and Level Shifting](../fundamentals/level-shifting.md)
- [flashrom](flashrom.md)
- [OpenOCD](openocd.md)
- [Glasgow Interface Explorer](glasgow.md)
