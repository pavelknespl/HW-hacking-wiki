# UBI and NAND Tools

Raw NAND flash is not a clean image. It carries spare bytes, error-correcting codes, and bad-block markers, and its filesystems (UBIFS, JFFS2, YAFFS) are designed around those quirks. This page covers the tools that turn a raw NAND dump into a readable filesystem, with UBI/UBIFS as the main focus because it dominates modern embedded Linux.

## What makes NAND different

A [NAND dump](../attacks/nand-emmc-dump.md) differs from a NOR or eMMC image in three ways:

- **Pages and OOB.** Data is stored in pages (for example 2048 bytes) each followed by a spare "out-of-band" area (for example 64 bytes) holding ECC and metadata. A raw reader may interleave these.
- **ECC.** Error-correcting codes in the OOB fix bit flips. You may need to apply or strip them.
- **Bad blocks.** Some blocks are unusable and marked as such. The flash translation layer or the OS skips them, so a linear read is not a linear filesystem.

Because of this, extracting a NAND filesystem is a two-stage job: normalize the raw dump, then interpret the filesystem.

## UBI and UBIFS, the common modern stack

On current embedded Linux, NAND is usually managed by **UBI** (Unsorted Block Images), a volume layer, with **UBIFS** filesystems inside UBI volumes. A NAND dump therefore contains a UBI image, which contains one or more volumes, one of which is the root UBIFS.

### ubireader

The `ubi_reader` toolkit extracts UBI images without a NAND device, directly from a dump file. It is the workhorse here.

```bash
# identify UBI structures and parameters in the dump
ubireader_display_info firmware_nand.bin

# extract all files from every UBIFS volume in the image
ubireader_extract_files firmware_nand.bin -o extracted/

# extract the raw UBI volume images instead of the files
ubireader_extract_images firmware_nand.bin -o volumes/
```

`ubireader` figures out the block and page sizes from the image, which saves you guessing. If it struggles, pass the parameters explicitly (`-b` block size, `-e` LEB size) based on the chip datasheet.

### ubidump

An alternative that lists and extracts UBIFS contents and is handy when `ubireader` and a particular image disagree:

```bash
python3 ubidump.py -s extracted/ firmware_nand.bin
```

Having two independent UBI extractors is worth it, because vendor variations trip up one or the other.

## nanddump and the mtd tools (on a live device)

If you have a shell on the target (over [UART](../protocols/uart.md) or a [bootloader](../attacks/bootloader-access.md)), the on-device MTD tools read NAND with the hardware ECC already applied, which sidesteps the raw-dump mess entirely:

```bash
# list the flash partitions
cat /proc/mtd

# dump an MTD partition, skipping the OOB so you get clean data
nanddump -o -f mtd3.bin /dev/mtd3
```

- `-o` omits the OOB area in the output.
- `nandwrite` writes an image back.
- `dd if=/dev/mtdblock3 of=part.bin` reads through the block layer, which also hides OOB and applies ECC.

Reading NAND from a running device is almost always cleaner than reading the raw chip off-board, because the controller and driver have already handled ECC and bad blocks. Prefer it when you have any on-device access.

## JFFS2 and YAFFS

Older or smaller devices use these directly on NAND.

=== "JFFS2"

    ```bash
    jefferson rootfs.jffs2 -d extracted/
    # or, with the kernel mtd emulator (nandsim / mtdram) and jffs2 mounted
    ```

    `jefferson` extracts JFFS2 from a dump without needing to mount it.

=== "YAFFS"

    Less tooling exists. `yaffs2utils` and the `unyaffs` family handle extraction; expect to specify page and spare sizes from the datasheet.

## Handling raw dumps with interleaved OOB

If your [NAND dump](../attacks/nand-emmc-dump.md) came off the raw chip with OOB interleaved, you may need to deinterleave before any filesystem tool will cooperate. Tools and scripts exist (for example parts of the `nandtool` and various community scripts) that split page data from spare bytes given the page/OOB geometry. Get the geometry from the chip datasheet: page size, OOB size, pages per block, and total size.

## A practical order of operations

1. **Prefer a clean source.** If you can read NAND from a live device with `nanddump -o` or `/dev/mtdblockN`, do that and skip the raw-dump problems.
2. **Inspect the raw dump** with `ubireader_display_info` to confirm it is UBI and learn its geometry.
3. **Extract files** with `ubireader_extract_files`, falling back to `ubidump` if needed.
4. **For JFFS2/YAFFS**, use `jefferson` or the YAFFS tools with the correct page/spare sizes.
5. **Then analyze** the extracted tree exactly as in [firmware analysis](../attacks/firmware-analysis.md).
