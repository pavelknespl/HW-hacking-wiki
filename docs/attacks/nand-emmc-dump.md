# NAND and eMMC Dumping

When firmware is too large for a NOR flash, it lives in NAND or eMMC. These are higher-capacity but harder to read: raw NAND needs ECC and bad-block handling, and eMMC is a dense BGA. This page covers both, and makes the case that reading from a live device beats reading the raw chip whenever you can manage it.

## Prefer reading from inside the device

Before any soldering, ask whether you can get a shell. A running system reads its own NAND or eMMC with ECC applied and bad blocks skipped, producing clean data. If you have access over [UART](../protocols/uart.md), a [bootloader](bootloader-access.md), or the network:

```bash
cat /proc/mtd                                   # list NAND partitions
nanddump -o -f /tmp/mtd4.bin /dev/mtd4          # clean NAND read, OOB stripped
dd if=/dev/mmcblk0 of=/tmp/emmc.img bs=4M       # clean eMMC block read
```

Then exfiltrate over `tftp`, `nc`, USB, or SD. This sidesteps nearly every difficulty below. Reserve the raw-chip methods for when no interactive access exists.

## eMMC

eMMC presents a clean block device thanks to its on-chip controller, so it is the easier of the two to read off-board or in-system. See [eMMC and SD](../protocols/emmc.md) for the protocol.

### In-system read

Tap the minimal MMC lines and read through an SD reader or eMMC adapter:

1. Locate `CLK`, `CMD`, `DAT0`, `VCC`/`VCCQ`, and `GND` test points.
2. **Isolate the SoC** from the bus: hold it in reset, or power the eMMC while the SoC is off, so your reader is the only master. This isolation is the hard part of in-system eMMC work.
3. Wire to an SD breakout (`CLK`, `CMD`, `DAT0`, VCC, GND map directly) or a dedicated eMMC-to-USB adapter.
4. The eMMC enumerates as a block device; image it with [`dd`](../software/dd-imaging.md):

```bash
sudo dd if=/dev/mmcblk0 of=emmc.img bs=4M status=progress conv=noerror,sync
```

### Off-board read

If isolation is impractical, reball the eMMC onto an adapter:

1. Remove the BGA with hot air (a preheater helps spread the thermal load).
2. Clean and reball with a stencil and fresh solder balls.
3. Seat it in an eMMC socket or adapter and image it in a reader.

BGA rework is a real skill; practice on scrap before a target you care about.

### After the dump

Treat it like a disk: inspect the partition table, mount partitions at their offsets, and carve anything non-standard.

```bash
fdisk -l emmc.img            # or gdisk -l for GPT
sudo mount -o loop,ro,offset=$((2048*512)) emmc.img /mnt
```

## Raw NAND

Raw NAND (SPI NAND in SOIC-8, or parallel NAND in TSOP-48) is the hardest common case because a raw read is not a filesystem. See [Memory Types](../fundamentals/memory-types.md) and [UBI and NAND Tools](../software/ubi-nand.md).

### Reading the chip

- **SPI NAND:** a programmer or adapter with SPI NAND support reads it over the SPI pins, similar to [SPI NOR](spi-flash-dump.md) but with a different command set and OOB handling.
- **Parallel NAND (TSOP-48):** generally desolder and read in a universal programmer (for example a XGecu with a TSOP-48 adapter), or a dedicated NAND reader. In-circuit parallel NAND reads are difficult because of the many shared high-speed lines.

### The raw dump is not a filesystem

A raw read includes:

- **Page data plus OOB** (spare bytes with ECC and metadata), possibly interleaved.
- **Bad-block markers** you must respect.
- **ECC** that may need applying to correct bit flips.

So extraction is two stages: normalize the raw dump (handle OOB, ECC, bad blocks, using the geometry from the chip datasheet), then interpret the filesystem.

### Rebuilding the filesystem

Modern NAND is usually UBI/UBIFS. Extract directly from the dump with `ubireader`:

```bash
ubireader_display_info nand.bin                 # confirm UBI, learn geometry
ubireader_extract_files nand.bin -o extracted/  # pull files from UBIFS volumes
```

For JFFS2, use `jefferson`; for YAFFS, the YAFFS tools with the correct page and spare sizes. Full detail on [UBI and NAND Tools](../software/ubi-nand.md).

!!! tip "Geometry comes from the datasheet"
    Page size, OOB size, pages per block, and total capacity are printed in the NAND datasheet. Getting these right is the difference between a clean extraction and garbage. `ubireader` can often infer them, but have the numbers ready.

## Decision summary

| Situation | Best approach |
| --- | --- |
| Any shell on the device | Read from inside with `nanddump -o` / `dd` |
| eMMC, SoC can be isolated | In-system read via `CLK/CMD/DAT0` |
| eMMC, no isolation | Reball and read off-board |
| SPI NAND, no shell | SPI reader with NAND support, then `ubireader` |
| Parallel NAND, no shell | Desolder, programmer, then `ubireader`/`jefferson` |

In every case, verify the dump by reading twice and comparing, and confirm the structure matches the boot log before trusting it.
