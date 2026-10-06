# SPI Flash Dumping

Pulling a firmware image off an external SPI NOR flash is the most common and most reliable hardware extraction. The chip uses a standard pinout, a cheap clip connects to it, and [`flashrom`](../software/flashrom.md) speaks its protocol. This page is the end-to-end procedure, including what to do when an in-circuit read fights back.

## What you need

- A [SOIC-8 test clip](../getting-started/toolkit.md#clips-sockets-and-probes) (Pomona 5250 or a decent clone).
- A [programmer](../software/flashrom.md#supported-programmers): CH341A or, better, an FT2232H adapter at the correct voltage.
- `flashrom` installed.
- The chip identified during [recon](../fundamentals/chip-identification.md), and its I/O voltage confirmed (usually 3.3 V, sometimes 1.8 V).

## The pinout

SPI NOR flash uses a fixed SOIC-8 pinout. Pin 1 is the dot/bevel corner:

| Pin | Name | To programmer |
| --- | --- | --- |
| 1 | `CS#` | CS |
| 2 | `DO` (MISO) | MISO |
| 3 | `WP#` | 3.3 V (or per below) |
| 4 | `GND` | GND |
| 5 | `DI` (MOSI) | MOSI |
| 6 | `CLK` | CLK |
| 7 | `HOLD#` | 3.3 V (or per below) |
| 8 | `VCC` | 3.3 V |

Most clip-to-programmer cables are pre-wired; just align pin 1.

## Step 1: clip on and probe

With the board **unpowered**, clip onto the chip aligning pin 1, and ask flashrom to identify it:

```bash
flashrom -p ch341a_spi        # or ft2232_spi:type=2232H,port=A
```

Success looks like:

```
Found Winbond flash chip "W25Q128.V" (16384 kB, SPI) ...
```

If it finds nothing, the clip contact is the usual culprit. Reseat it, press gently, and check pin 1 alignment. A multimeter continuity check from each clip leg to the chip pin confirms contact.

## Step 2: read, several times

```bash
flashrom -p ch341a_spi -r dump1.bin
flashrom -p ch341a_spi -r dump2.bin
flashrom -p ch341a_spi -r dump3.bin
sha256sum dump*.bin
```

Three identical hashes mean a clean read. Different hashes mean the rest of the board is interfering, which is common in-circuit. Move to the troubleshooting section.

## Step 3: verify the contents make sense

```bash
binwalk dump1.bin
```

The structure should match the partition layout you saw in the [boot log](recon.md#capture-a-boot-log). A dump that is all `0xFF`, all `0x00`, or pure noise is a failed read, not an encrypted chip.

## The in-circuit power problem

In-circuit, the chip shares its `VCC` and SPI lines with the SoC. When the clip powers the chip, it often back-powers the whole board partially, and the SoC may wake up and drive the bus, corrupting your read. Three fixes, in order:

=== "Hold the SoC in reset"

    Pull the SoC's reset line to its inactive level so it releases the bus while you read the flip. Find the reset pin from the datasheet. This is the cleanest fix when you can locate reset.

=== "Pull WP# / HOLD# appropriately"

    Tying `HOLD#` can pause the chip's response to the SoC; configurations vary by board. Sometimes grounding the SoC's reset plus leaving the clip to power only the flash is enough.

=== "Power the board, isolate the read"

    Some workflows power the board normally (so the SoC is in a known state, ideally held in reset) and read the flash with the programmer not supplying VCC. Never drive two power sources into the same rail. See [Lab Setup](../getting-started/lab-setup.md).

If none of this yields a stable read, desolder.

## Desoldering the chip

When in-circuit reads will not stabilize, remove the chip:

1. Protect neighbors with Kapton tape.
2. With hot air at an appropriate temperature and plenty of flux, heat the chip evenly and lift it with tweezers once the solder flows.
3. Clean the pads. Read the chip in a SOIC-8 socket or on an adapter with the same `flashrom` commands.
4. Resolder it afterward if the device must keep working.

Desoldering always gives a clean read because nothing else is on the bus. It is more work and more risk, so it is the fallback, not the default.

## Step 4: analyze

With a verified image:

```bash
binwalk -Me dump1.bin
```

Carve the filesystem, mount it, and search for secrets. Continue at [Firmware Analysis](firmware-analysis.md).

## Writing back

To run modified firmware on a device that loads SPI contents unverified:

```bash
flashrom -p ch341a_spi -w modified.bin
```

Keep `dump1.bin` as your restore image. If the device verifies the image at boot ([secure boot](bootloader-bypass.md)), a plain rewrite fails and you need a different approach.

## Defender's note

An unencrypted SPI NOR is an open book to anyone with a two dollar clip. Encrypt secrets at rest, authenticate the boot image, and never assume physical storage is private.
