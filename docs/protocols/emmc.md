# eMMC and SD

eMMC is managed [NAND](../fundamentals/memory-types.md) flash: a NAND die plus a controller in a single BGA package that presents a clean block device, just like an SD card or a disk. On larger embedded devices, such as set-top boxes, tablets, and higher-end routers, the whole operating system lives in eMMC. Reading it gives you a complete, partitioned filesystem image with none of the ECC and bad-block hassle of [raw NAND](../attacks/nand-emmc-dump.md).

## Why eMMC is friendlier than raw NAND

The integrated controller handles wear leveling, bad-block management, and error correction internally. From the outside you see logical blocks, so:

- A read produces a **clean image** with a partition table (often GPT or an Android-style layout), not a soup of pages and spare bytes.
- You can mount partitions directly from the dump.
- No ECC stripping, no OOB interpretation, no bad-block table to rebuild.

The cost is that you cannot see below the controller, so controller-level security (RPMB, write-protect partitions) is opaque.

## The bus

eMMC uses the MMC protocol, electrically similar to SD. The signals you care about for a minimal in-system read:

| Signal | Role |
| --- | --- |
| `CLK` | Clock |
| `CMD` | Command line |
| `DAT0` | One data line (1-bit mode) |
| `VCC` / `VCCQ` | Core and I/O power |
| `GND` | Ground |

Full-speed operation uses eight data lines (`DAT0` to `DAT7`), but a slow 1-bit read needs only `DAT0`, which makes in-system tapping feasible.

## Reading in-system

The appeal of eMMC is that you can often read it without removing the chip, by wiring those few lines to an SD card reader or a dedicated eMMC adapter.

1. **Find the test points.** `CLK`, `CMD`, and `DAT0` are frequently broken out to pads. The datasheet or FCC internal photos help locate them.
2. **Hold the SoC off the bus.** The host SoC also drives these lines. Keep the SoC in reset, or remove its power while powering the eMMC, so your reader is the only master. Finding a way to isolate the SoC is the trickiest part of in-system reads.
3. **Wire to a reader.** An SD card breakout maps cleanly: SD `CLK`, `CMD`, `DAT0`, VCC, and GND to the corresponding eMMC lines. Dedicated eMMC-to-USB adapters make this tidier.
4. **Read as a block device.** The reader enumerates the eMMC as `/dev/mmcblk0` or similar, and you image it with [`dd`](../software/dd-imaging.md):

```bash
sudo dd if=/dev/mmcblk0 of=emmc.img bs=4M status=progress
```

## Reading out-of-circuit

When in-system isolation is impractical, reball the eMMC onto an adapter and read it in a socket. This always works but is invasive and needs BGA rework skills. See [NAND and eMMC Dumping](../attacks/nand-emmc-dump.md) for technique.

## The boot and user partitions

eMMC exposes several hardware partitions:

- **User area:** the large main store with the OS and data.
- **Boot partitions** (`boot0`, `boot1`): small areas the SoC may read first.
- **RPMB:** a replay-protected area for secure counters and keys, not readable without the right key.

When imaging, capture the user area first (it holds the filesystems), and note that boot partitions may contain the bootloader. On Linux these appear as `/dev/mmcblk0`, `/dev/mmcblk0boot0`, and so on.

## After the dump

Treat the image like a disk: inspect the partition table, loop-mount or extract partitions, and carve with [binwalk](../software/binwalk.md) if the layout is non-standard.

```bash
# inspect partitions
fdisk -l emmc.img
# or for GPT
gdisk -l emmc.img

# mount a partition at a known offset (offset = start_sector * 512)
sudo mount -o loop,offset=$((2048*512)) emmc.img /mnt
```

From there it is ordinary [firmware analysis](../attacks/firmware-analysis.md).

## SD cards

Everything above applies to removable SD and microSD cards, which are even easier: pull the card and image it in any reader. On devices that boot from a microSD (many single-board computers and some appliances), the card *is* the firmware store, and reading or modifying it needs no soldering at all.


## Related pages

- [NAND and eMMC Dumping](../attacks/nand-emmc-dump.md)
- [Memory Types](../fundamentals/memory-types.md)
- [dd and Imaging](../software/dd-imaging.md)
- [UBI and NAND Tools](../software/ubi-nand.md)
