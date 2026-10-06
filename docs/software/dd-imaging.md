# dd and Imaging

`dd` makes byte-exact copies. In hardware hacking it images block devices (SD cards, eMMC, USB mass storage exposed by a device) and carves known-offset regions out of a firmware dump. It is blunt, fast, and unforgiving, so the discipline is in choosing the right source and destination.

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

- For **SPI NOR flash**, use [`flashrom`](flashrom.md), which speaks the chip's protocol. `dd` only works on something that already presents as a block device.
- For **raw NAND** with ECC and OOB, `dd` on the raw device gives you an un-processed image that still needs [NAND tooling](ubi-nand.md) to become a filesystem.
- For **structured extraction**, let [`binwalk`](binwalk.md) carve; reserve `dd` for when you want exact, known offsets.
