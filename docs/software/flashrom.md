# flashrom

`flashrom` reads, writes, verifies, and erases flash chips through a programmer. For hardware hacking its main use is pulling a firmware image off an external [SPI](../protocols/spi.md) NOR flash, and sometimes writing a modified one back. It identifies the chip automatically and handles hundreds of parts.

## Supported programmers

`flashrom` talks to many adapters. The common ones for bench work:

| Programmer | `-p` argument | Notes |
| --- | --- | --- |
| CH341A | `ch341a_spi` | Cheap and everywhere. Watch the voltage (see below). |
| FT2232H / FT232H | `ft2232_spi:type=2232H,port=A` | Reliable, correct 3.3 V, great value. |
| Bus Pirate | `buspirate_spi:dev=/dev/ttyUSB0` | Slow but universal. |
| [Raspberry Pi](../scripting/gpio-control.md) [SPI](../protocols/spi.md) | `linux_spi:dev=/dev/spidev0.0,spispeed=10000` | Uses the Pi's own SPI controller. |
| Dediprog | `dediprog` | Commercial, fast, dependable. |

List what your build supports with `flashrom -L`.

## Identify the chip first

Connect the programmer to the chip (a [SOIC-8 clip](../attacks/spi-flash-dump.md) is the usual way) and ask `flashrom` to probe:

```bash
flashrom -p ch341a_spi
```

A good connection prints the detected chip:

```
Found Winbond flash chip "W25Q128.V" (16384 kB, SPI) ...
```

If it finds nothing, or reports multiple possible chips, the clip contact, wiring, or power is suspect. Reseat and retry before doing anything else.

!!! warning "The CH341A 5 V problem"
    Many CH341A clones drive their SPI lines at 5 V even though most flash chips are 3.3 V parts. This can stress or damage the chip over time. Use a known 3.3 V clone, apply the common 3.3 V modification, or prefer an FT2232H adapter, which drives the correct voltage.

## Read (dump) the flash

The core operation. Read the whole chip to a file:

```bash
flashrom -p ch341a_spi -r dump.bin
```

### Read three times and compare

!!! tip "Verify the read is stable"
    In-circuit reads can be subtly corrupted by the rest of the board. Read the chip two or three times and compare the files:

    ```bash
    flashrom -p ch341a_spi -r dump1.bin
    flashrom -p ch341a_spi -r dump2.bin
    sha256sum dump1.bin dump2.bin
    ```

    Identical hashes mean a clean read. Differences mean interference; power the board externally, hold the SoC in reset, or desolder the chip. See [SPI Flash Dumping](../attacks/spi-flash-dump.md).

## Force a chip model

When `flashrom` cannot decide between similar parts, name the chip with `-c`:

```bash
flashrom -p ch341a_spi -c "W25Q128.V" -r dump.bin
```

Only force a model you have confirmed from the markings; the wrong size truncates or over-reads.

## Write a modified image

To run your own firmware on a device that loads it unverified from SPI:

```bash
# verify current contents match your reference, then write, then verify
flashrom -p ch341a_spi -w newimage.bin
```

`flashrom` erases, writes, and verifies by default. Keep the original `dump.bin` as a restore point. If the device uses [secure boot](../attacks/bootloader-bypass.md), a straight rewrite will fail verification at boot; a direct write only works where the boot chain trusts the flash.

## Erase and verify

```bash
flashrom -p ch341a_spi -E            # erase the whole chip
flashrom -p ch341a_spi -v image.bin  # verify chip matches a file
```

## When reads are unreliable in-circuit

Other chips on the shared SPI bus, or the powered SoC, can corrupt an in-circuit read. Remedies, in order of effort:

1. **Power the board externally** at the correct voltage and read without the clip powering it.
2. **Hold the SoC in reset** (ground its reset line, or pull the `WP#`/`HOLD#` appropriately) so it releases the bus.
3. **Desolder the chip** and read it in a socket, which always works.

The full decision tree is on [SPI Flash Dumping](../attacks/spi-flash-dump.md).

## After the dump

Run [`binwalk`](binwalk.md) over the image to see its structure and carve out the filesystems. That is where [firmware analysis](../attacks/firmware-analysis.md) begins.


## Related pages

- [SPI](../protocols/spi.md)
- [SPI Flash Dumping](../attacks/spi-flash-dump.md)
- [Binwalk and Extraction](binwalk.md)
- [Memory Types](../fundamentals/memory-types.md)
