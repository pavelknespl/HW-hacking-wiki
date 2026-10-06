# dd and Imaging

`dd` makes byte-exact copies. In hardware hacking it images block devices (SD cards, [eMMC](../protocols/emmc.md), USB mass storage exposed by a device) and carves known-offset regions out of a firmware dump. It is blunt, fast, and unforgiving, so the discipline is in choosing the right source and destination.

## The mental model

`dd` copies from `if` (input file) to `of` (output file) in blocks of size `bs`, optionally skipping and limiting:

```bash
dd if=<source> of=<dest> bs=<blocksize> [skip=N] [seek=N] [count=N] status=progress
```

- `skip=N` skips N blocks at the **input**.
- `seek=N` skips N blocks at the **output**.
- `count=N` copies only N blocks.
- `status=progress` prints throughput so you know it is alive.

!!! danger "if and of are easy to swap, and the mistake is destructive"
    Writing to the wrong `of=` overwrites it with no confirmation. Double-check the device node with `lsblk` before every write, and prefer imaging *to* a file rather than *onto* a device whenever possible.

## Choosing a block size

`bs` sets how many bytes `dd` reads and writes per I/O operation. It is the setting people most often get wrong, and it affects two different things: **speed** and, when `skip`/`seek`/`count` are involved, **what bytes you actually copy**. Understanding it means understanding a little about how the underlying memory is organized.

### Block size is about memory geometry

Storage is not a flat tape of bytes to the hardware; it is organized into fixed-size units, and matching `dd` to those units is what makes a copy fast and correct.

- **Logical sector (512 B or 4 KiB).** Block devices present data in sectors. Most tools report partition offsets in 512-byte sectors, which is why offset math so often multiplies by 512. See the [eMMC and SD](../protocols/emmc.md) page, where a partition starting at "sector 2048" means byte offset `2048 * 512`.
- **Flash page (for example 2 KiB).** [NAND flash](../fundamentals/memory-types.md) is read and written a page at a time. A page plus its spare [OOB](../software/ubi-nand.md) area is the smallest unit the chip hands out.
- **Flash erase block (for example 128 KiB).** [NAND](../fundamentals/memory-types.md) is erased a whole block at a time, which is why [UBI and NAND tooling](../software/ubi-nand.md) cares about erase-block size, and why writing flash is never truly byte-granular underneath.

When you image a whole [block device](../protocols/emmc.md), the block size is purely a performance knob: too small and `dd` makes millions of tiny calls and crawls; too large and you waste memory with no gain. `bs=4M` is the usual sweet spot for whole-device reads.

### Block size changes what skip and count mean

The subtlety: `skip`, `seek`, and `count` are counted in **blocks**, not bytes. The same `skip=1` copies a completely different region depending on `bs`:

```bash
dd if=fw.bin of=out.bin bs=1   skip=320      # skip 320 bytes
dd if=fw.bin of=out.bin bs=1K  skip=320      # skip 320 KiB  (327680 bytes)
dd if=fw.bin of=out.bin bs=4K  skip=320      # skip 1.25 MiB (1310720 bytes)
```

So when you carve a region at a known byte offset (for example a [SquashFS](binwalk.md) that [binwalk](../software/binwalk.md) located at `0x140000`), you have two correct choices:

1. **`bs=1`, offset in bytes.** Exact and foolproof, but slow because `dd` does one byte per operation:

    ```bash
    dd if=firmware.bin of=rootfs.squashfs bs=1 skip=$((0x140000))
    ```

2. **A larger block size, offset in blocks.** Fast, but only correct when the byte offset is an exact multiple of the block size. `0x140000` is `1310720`, which divides evenly by `4096`, so:

    ```bash
    # 0x140000 / 4096 = 320 blocks
    dd if=firmware.bin of=rootfs.squashfs bs=4096 skip=320
    ```

!!! tip "The alignment trap"
    If the offset is **not** a multiple of your block size, `bs=4K skip=N` lands in the wrong place and your carve is garbage, often silently. When the offset is awkward, either fall back to `bs=1`, or carve the misaligned leading bytes with `bs=1` and the aligned bulk with a big block size. When in doubt, `bs=1` is always correct, just slow.

### A rule of thumb

- **Whole-device image:** `bs=4M` (speed only, offsets not involved).
- **Carving at a byte offset that is block-aligned:** large `bs`, offset expressed in blocks.
- **Carving at an arbitrary byte offset:** `bs=1` with the offset in bytes, or let [binwalk](../software/binwalk.md) carve it for you.

## Imaging a block device

When a device's storage appears as a block device (an [eMMC or SD](../protocols/emmc.md) through a reader, or a device that exposes mass storage), image the whole thing to a file:

```bash
lsblk                                   # identify the device, e.g. /dev/sdb or /dev/mmcblk0
sudo dd if=/dev/mmcblk0 of=emmc.img bs=4M status=progress conv=noerror,sync
```

- `bs=4M` is a good throughput compromise for whole-device reads.
- `conv=noerror,sync` keeps going past read errors and pads the bad spots, so one flaky sector does not abort a multi-gigabyte image. The padding keeps every subsequent offset correct.

## Carving a region from a firmware dump

After [`binwalk`](binwalk.md) tells you where a part starts, carve it precisely. For a SquashFS at offset `0x140000`:

```bash
dd if=firmware.bin of=rootfs.squashfs bs=1 skip=$((0x140000))
```

`bs=1` makes `skip` count bytes exactly, at the cost of speed. For large carves, use a larger block size and express the offset in blocks, or carve the leading bytes with `bs=1` and the bulk with a big block size.

A faster idiom for large, block-aligned carves:

```bash
# skip 0x140000 bytes using a 4096-byte block (0x140000 / 4096 = 320 blocks)
dd if=firmware.bin of=rootfs.squashfs bs=4096 skip=320
```

## Verifying an image

A dump you cannot trust is worthless. Hash it and, for critical reads, take two images and compare:

```bash
sha256sum emmc.img
# read twice, compare
sudo dd if=/dev/mmcblk0 of=a.img bs=4M status=progress
sudo dd if=/dev/mmcblk0 of=b.img bs=4M status=progress
cmp a.img b.img && echo "identical"
```

Matching hashes across reads is your evidence that the copy is faithful, the same principle as reading a [SPI flash](flashrom.md#read-three-times-and-compare) multiple times.

## Mounting an image read-only

Inspect a filesystem image without risking changes to it:

```bash
# whole-disk image: find the partition offset first
fdisk -l emmc.img
# mount a partition that starts at sector 2048 (2048 * 512 = 1048576)
sudo mount -o loop,ro,offset=1048576 emmc.img /mnt
```

Mount **read-only** (`ro`) during analysis so you never accidentally mutate evidence.

## Writing an image back

To restore a device or deploy a modified image to an SD card you own:

```bash
sudo dd if=modified.img of=/dev/sdb bs=4M status=progress conv=fsync
sync
```

`conv=fsync` and a trailing `sync` ensure the data is physically on the card before you pull it.

## When not to use dd

- For **[SPI](../protocols/spi.md) NOR flash**, use [`flashrom`](flashrom.md), which speaks the chip's protocol. `dd` only works on something that already presents as a block device.
- For **raw NAND** with ECC and OOB, `dd` on the raw device gives you an un-processed image that still needs [NAND tooling](ubi-nand.md) to become a filesystem.
- For **structured extraction**, let [`binwalk`](binwalk.md) carve; reserve `dd` for when you want exact, known offsets.


## Related pages

- [eMMC and SD](../protocols/emmc.md)
- [Binwalk and Extraction](binwalk.md)
- [NAND and eMMC Dumping](../attacks/nand-emmc-dump.md)
- [Memory Types](../fundamentals/memory-types.md)
- [UBI and NAND Tools](ubi-nand.md)
