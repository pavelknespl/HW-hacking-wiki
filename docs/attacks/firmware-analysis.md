# Firmware Analysis

You have a verified firmware image. Now you read it. Analysis is where hardware access pays off: the whole userland, the configuration, the keys, and the bugs are all sitting in the file. This page is the workflow from raw image to understanding, bridging [extraction](firmware-extraction.md) and deep [reverse engineering](../reverse-engineering/firmware-workflow.md).

## Step 1: understand the layout

Start with structure before content.

```bash
binwalk firmware.bin        # signature map
binwalk -E firmware.bin     # entropy: compressed vs encrypted
binwalk -Y firmware.bin     # architecture guess for raw code
```

From this you learn the parts (bootloader, kernel, filesystem), whether anything is encrypted, and the [architecture](../fundamentals/architectures.md). Cross-check against the boot log's partition table. See [Binwalk](../software/binwalk.md).

## Step 2: extract the filesystem

```bash
binwalk -Me firmware.bin
```

Carve recursively, then open the root filesystem (`unsquashfs`, `sasquatch`, `jefferson`, or [UBI tools](../software/ubi-nand.md) depending on type). You should end with a familiar Linux tree: `/etc`, `/bin`, `/www`, and so on.

## Step 3: sweep for secrets

Low-effort, high-yield. Search the extracted tree:

```bash
# accounts and hashes
cat etc/passwd etc/shadow 2>/dev/null

# broad secret sweep
grep -rnI -E "password|passwd|secret|token|BEGIN .*PRIVATE KEY|api[_-]?key|ssh-rsa" .

# strings across all binaries for URLs, hosts, and keys
grep -rnI -E "https?://|[0-9]{1,3}(\.[0-9]{1,3}){3}" .
```

What commonly turns up:

- **Hardcoded credentials** and default accounts in `passwd`/`shadow` or config files.
- **Private keys and certificates** (TLS, SSH host keys, code-signing keys).
- **API tokens and cloud endpoints** the device talks to.
- **Backdoor or debug accounts** gated behind an undocumented flag.

### Cracking password hashes

If `shadow` has hashes, crack them offline:

```bash
# john
john --wordlist=rockyou.txt shadow
# hashcat (mode depends on the hash type, e.g. 1800 for sha512crypt)
hashcat -m 1800 hashes.txt rockyou.txt
```

A cracked root password often opens the [UART login](../protocols/uart.md) you could not get past.

## Step 4: understand how it boots and what it trusts

Read the startup path to learn the device's logic and trust assumptions:

- `/etc/inittab`, `/etc/init.d/`, `/etc/rc*`: what starts, in what order.
- The web server config and its CGI directory: the network attack surface.
- Update scripts: how new firmware is fetched and **whether it is verified**. An unverified update path is a remote code path.
- Anything that reads an [EEPROM](../protocols/i2c.md) or environment variable to decide behavior (debug modes, unlock flags).

## Step 5: find the interesting binaries

The custom binaries are where product-specific bugs live. Prioritize:

- The **web/CGI handlers** (classic command injection and authentication bugs).
- Daemons listening on the network (`netstat`-named services, from init scripts).
- Anything **setuid** or running as root.
- Update and provisioning tools.

Identify architecture and endianness per binary:

```bash
file bin/busybox
readelf -h bin/httpd
```

## Step 6: reverse the code

Load the chosen binaries into [Ghidra](../reverse-engineering/ghidra-setup.md) with the correct language setting (the [architecture page](../fundamentals/architectures.md) lists the IDs, and remember MIPS endianness). For a raw bootloader blob with no ELF header, you set the base address and architecture manually. The full process is on the [Firmware RE Workflow](../reverse-engineering/firmware-workflow.md) page.

Typical goals:

- Trace user input from the network into a dangerous call (`system`, `popen`, `strcpy`).
- Locate the authentication check and see how it can be bypassed.
- Find the decryption routine and key for an encrypted update or partition.

## Handling encrypted firmware

If Step 1 showed uniform high entropy and no signatures, the image is encrypted and carving will not work. Pivot:

- Reverse the **bootloader**, which must contain or fetch the key to run the system. The key and algorithm are in there somewhere.
- **Capture plaintext at runtime**: halt over [JTAG](../protocols/jtag.md) after decryption, or [glitch](fault-injection.md) into a state that exposes memory.
- Look for an **unencrypted recovery or factory image**, or a debug mode that accepts plaintext.

## Emulation for dynamic analysis

Static reading has limits. To run firmware userland on your workstation:

- **QEMU user-mode** runs a single cross-architecture binary with its libraries: `qemu-mipsel -L ./rootfs ./bin/httpd`.
- **Full-system emulation** boots the whole image; frameworks like **FirmAE** and **Firmadyne** automate this for Linux router/camera firmware, letting you poke the web interface locally.

Emulation lets you fuzz and debug the services without the physical device, though hardware-specific peripherals often need stubbing.

## Keep notes and diff

Record findings as you go, and when you have two firmware versions, **diff them**. A security update's changed files point straight at what was vulnerable, which is one of the fastest ways to find a bug worth reporting.
