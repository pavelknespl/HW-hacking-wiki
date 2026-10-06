# Binwalk and Extraction

A raw firmware dump is a blob of concatenated parts: a bootloader, one or more kernels, compressed filesystems, and padding. `binwalk` finds those parts by their signatures and carves them out, so you can mount the filesystem and read the code. It is the first tool you run on any image.

## What binwalk does

`binwalk` scans a file for known **magic signatures**: the byte patterns that mark the start of a gzip stream, a SquashFS filesystem, a JFFS2 image, a uImage header, a certificate, and hundreds more. From those it maps the layout and, with extraction enabled, pulls each piece into separate files.

## A first look

Run a signature scan to see the layout without extracting anything:

```bash
binwalk firmware.bin
```

Typical output:

```
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             uImage header, ... CPU: ARM, ... image name: "Linux Kernel"
64            0x40            LZMA compressed data
1310720       0x140000        Squashfs filesystem, little endian, version 4.0, ...
```

That tells you: a uImage kernel at the start, LZMA-compressed, and a SquashFS root filesystem at offset `0x140000`. You now know the architecture and where the userland lives.

## Extracting

Add extraction to carve the pieces out:

```bash
binwalk -e firmware.bin
```

This creates a `_firmware.bin.extracted/` directory with the carved parts. For deeply nested images (a filesystem inside a kernel inside a wrapper), recurse through matryoshka layers:

```bash
binwalk -Me firmware.bin
```

`-M` (matryoshka) re-scans each extracted file, peeling nested layers automatically.

!!! note "binwalk versions"
    The long-standing Python `binwalk` (v2) and the newer Rust rewrite (v3) differ slightly in flags and in which extractors they bundle. The scan (`binwalk file`) and extract (`-e`, `-Me`) behaviors above hold across both. If an extractor is missing, install the helper tool it needs (for example `sasquatch` for vendor-modified SquashFS).

## Entropy analysis: spotting encryption and compression

High, flat entropy across a region means the data is compressed or encrypted. This distinction matters enormously:

```bash
binwalk -E firmware.bin      # entropy graph
```

- **Rising and falling entropy** with recognizable signatures: normal compressed sections. Carve and decompress.
- **Uniformly high entropy (near 1.0) with no signatures at all:** likely **encrypted**. No amount of carving helps until you find the key or decryption routine. That pushes you toward [firmware analysis](../attacks/firmware-analysis.md) of the bootloader, or [fault injection](../attacks/fault-injection.md) to grab the plaintext after the device decrypts it in memory.

Encrypted firmware is a fork in the road: recognize it early so you do not waste time trying to carve noise.

## Identifying the architecture from a blob

When you have a raw code section and no ELF header, let binwalk guess the instruction set with an opcode scan:

```bash
binwalk -Y firmware.bin      # capstone-based architecture detection
```

This helps pick the right [Ghidra](../reverse-engineering/ghidra-setup.md) language, especially for choosing MIPS endianness.

## Manual carving with dd

When you trust the offsets more than the automatic extractor, carve by hand with [`dd`](dd-imaging.md). From the scan above, the SquashFS starts at `0x140000`:

```bash
dd if=firmware.bin of=rootfs.squashfs bs=1 skip=$((0x140000))
```

Manual carving avoids extractor quirks and gives you exactly the bytes you want.

## Mounting and reading filesystems

The carved filesystem is the goal. How you open it depends on the type:

=== "SquashFS"

    ```bash
    # standard squashfs
    unsquashfs rootfs.squashfs
    # vendor-modified squashfs (common on routers) needs sasquatch
    sasquatch rootfs.squashfs
    ```

    Many vendors tweak the SquashFS header, so stock `unsquashfs` fails where `sasquatch` succeeds.

=== "JFFS2"

    ```bash
    jefferson rootfs.jffs2 -d extracted/
    ```

=== "UBIFS / raw NAND"

    See [UBI and NAND Tools](ubi-nand.md) for `ubireader` and friends. NAND images need their layout handled before the filesystem appears.

=== "CramFS / ext"

    ```bash
    sudo mount -o loop rootfs.img /mnt      # ext2/3/4, cramfs
    ```

## What you are looking for

Once the root filesystem is extracted, the interesting files are predictable:

- `/etc/passwd` and `/etc/shadow` for accounts and password hashes.
- `/etc/` broadly for configuration, keys, and certificates.
- `/www/` or `/usr/www/` and any CGI handlers, a classic injection surface.
- Startup scripts in `/etc/init.d/` or `/etc/rc*` that show how the device boots and what it trusts.
- Custom binaries in `/bin`, `/sbin`, `/usr/bin` to load into [Ghidra](../reverse-engineering/ghidra-setup.md).

Searching the extracted tree for secrets is fast and often productive:

```bash
grep -rnI -E "password|passwd|secret|BEGIN .*PRIVATE KEY|api[_-]?key" extracted/
```

This flow, dump, scan, carve, mount, search, is the backbone of every [firmware analysis](../attacks/firmware-analysis.md).


## Related pages

- [Firmware Analysis](../attacks/firmware-analysis.md)
- [dd and Imaging](dd-imaging.md)
- [UBI and NAND Tools](ubi-nand.md)
- [Firmware Extraction](../attacks/firmware-extraction.md)
