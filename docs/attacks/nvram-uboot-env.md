# NVRAM and U-Boot Environment Analysis

Between the bootloader and the operating system sits a small store of configuration: the **U-Boot environment** and, on many routers and embedded devices, a vendor **NVRAM** area. These hold boot logic, device identity, feature flags, and sometimes credentials and keys. They are small, often in plaintext, and frequently overlooked, which makes them a high-yield, low-effort target once you have a [firmware dump](firmware-extraction.md) or a [shell](getting-a-shell.md).

## The U-Boot environment

U-Boot keeps a set of `name=value` variables that control boot: `bootcmd` (what runs at autoboot), `bootargs` (the kernel command line), network settings, and whatever the vendor added. It lives either embedded in the U-Boot image or in its own flash region, as a block of null-separated `key=value` strings preceded by a CRC32.

Read it three ways:

- **From the U-Boot prompt**: `printenv`. See [Bootloader Access](bootloader-access.md).
- **From a running Linux shell**: `fw_printenv` (and `fw_setenv`), which read the same flash region the bootloader uses.
- **From a flash dump**: locate the environment region (a CRC32 followed by printable `key=value` pairs) and read it directly.

```bash
# on a live device
fw_printenv

# from a raw flash dump: find the env block
strings -n 4 flash.bin | grep -E '^(bootargs|bootcmd|baudrate|ipaddr|serverip)='
```

### What to look for

- **`bootargs`** often reveals how the console and root are set up, and occasionally contains an option that disables login or sets a debug mode.
- **`bootcmd`** shows exactly where and how the kernel loads, which informs [bootloader bypass](bootloader-bypass.md).
- **Vendor variables**: unlock flags, "engineering mode" toggles, factory settings, and sometimes passwords or keys stashed as env vars.
- **Writability**: if you can `setenv`/`saveenv` (or `fw_setenv`), you can change boot behavior, for example adding `init=/bin/sh`. Keep a backup; a bad `saveenv` can brick normal boot.

## Vendor NVRAM (routers especially)

Broadcom-based and many other routers keep a separate **NVRAM** store of `name=value` settings that the firmware reads at runtime: Wi-Fi SSIDs and passphrases, admin credentials, WPS PINs, serial numbers, region and feature flags, and more. It is usually plaintext.

```bash
# on a live device with the vendor tool
nvram show
nvram get http_passwd
nvram get wl0_wpa_psk

# from a flash dump: the NVRAM region is plaintext key=value pairs
strings -n 4 flash.bin | grep -iE 'passwd|psk|key|pin|secret|ssid'
```

NVRAM is a frequent source of real findings: the admin password, the Wi-Fi key, and a device-unique secret often sit here in the clear, recoverable from a flash dump with nothing more than `strings`.

## Why it is worth checking first

Both stores are small, usually plaintext, and sit right next to the firmware you already dumped. Before loading anything into [Ghidra](../reverse-engineering/ghidra-setup.md), grep the environment and NVRAM for credentials and flags, it is minutes of work and regularly yields the password, a debug toggle, or the exact boot logic you need. It pairs naturally with the broader [firmware analysis](firmware-analysis.md) sweep.

## Finding the regions in a dump

- Run [`binwalk`](../software/binwalk.md) and look for the environment/NVRAM region, or scan for a CRC32 followed by a run of printable `key=value` lines.
- Cross-check against the [boot log's](recon.md#capture-a-boot-log) partition table, which often names an `env` or `nvram` partition and its offset.

## Defender's view

Do not store credentials, Wi-Fi keys, or secrets in plaintext NVRAM or the U-Boot environment; anyone with a flash dump reads them with `strings`. Encrypt sensitive settings, lock the [bootloader](bootloader-access.md#defenders-view) so the environment cannot be edited to force a shell, and treat the environment as attacker-readable and attacker-writable on a device with physical access.

## Related pages

- [Bootloader Access (U-Boot)](bootloader-access.md)
- [Firmware Analysis](firmware-analysis.md)
- [Firmware Extraction](firmware-extraction.md)
- [Bootloader Bypass](bootloader-bypass.md)
- [Binwalk and Extraction](../software/binwalk.md)
