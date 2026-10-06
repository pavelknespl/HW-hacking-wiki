# Bootloader Access (U-Boot)

Between power-on and the operating system sits the bootloader, and on embedded Linux that bootloader is almost always **U-Boot**. It initializes the hardware, then loads and launches the kernel. If you can interrupt it and reach its command prompt, you gain a powerful foothold: read and write memory and flash, change the kernel command line, and very often hand yourself a root shell.

## Why the bootloader is a target

U-Boot runs with full privilege and no OS security model. From its prompt you can:

- **Read and write memory and flash**, which means dumping firmware or patching it.
- **Change boot arguments** (`bootargs`), for example to boot the kernel straight into a shell.
- **Load code over the network or serial** and run it.
- **Inspect and modify the environment**, which may hold passwords, keys, and boot logic.

It is often the easiest route to full control once you have a [serial console](../protocols/uart.md).

## Interrupting autoboot

During boot, U-Boot prints a countdown and waits briefly for a keypress:

```
U-Boot 2016.05 ...
Hit any key to stop autoboot:  3
```

Press a key (or hold Enter) during that window and you drop to the `=>` prompt. The window is usually one to three seconds, so have the terminal focused and start tapping as soon as power is applied.

### When the prompt is suppressed

Vendors frequently try to hide or disable this. Workarounds:

- **Spam keys from power-on.** Hold a key down or send a stream so you catch the window even if it is tiny.
- **`bootdelay` set to 0.** The countdown is skipped. Some builds still check for a specific interrupt key very early; try sending a flood. Others respond to a particular character.
- **Interrupt character.** Some configurations listen for `Ctrl-C` or a specific byte rather than "any key."
- **Glitch the boot.** A momentary data-line corruption as U-Boot reads its environment can drop it to the prompt or a recovery path. See [Fault Injection](fault-injection.md).
- **Corrupt the kernel or environment.** If the kernel image fails to load (for example you erase or alter a byte via a flash chip), U-Boot often falls back to its prompt. This is drastic; keep a backup image.

## Useful U-Boot commands

Once at the `=>` prompt:

```
help                 # list available commands (varies by build)
printenv             # show all environment variables
printenv bootargs    # the kernel command line
printenv bootcmd     # what autoboot runs
bdinfo               # board info, memory base and size
```

Memory and flash:

```
md 0x80000000 20     # memory display, 20 words from that address
mw 0x80000000 0      # memory write
nand info            # NAND details (nand read/write to move data)
sf probe             # init SPI flash; sf read/write to move data
```

## Dumping firmware from U-Boot

A clean, no-solder extraction when you have the prompt. Read flash into RAM, then move it off the device.

```
# SPI NOR example: probe, read 16 MB of flash into DRAM
sf probe
sf read 0x80000000 0x0 0x1000000
# then transfer: tftp out to a server, or dump over serial
tftpput 0x80000000 0x1000000 firmware.bin
```

For NAND, `nand read 0x80000000 0x0 0x<size>` then transfer. If you have no network, some builds support `loady`/`loadb` over serial, or you can `md` regions and capture the console log, which is slow but works for small areas.

## Getting a shell by changing bootargs

The classic move. The kernel command line's `init=` parameter sets the first process. Point it at a shell and the system boots directly to root:

```
setenv bootargs "${bootargs} init=/bin/sh"
boot
```

If the device uses BusyBox, `init=/bin/sh` or `init=/bin/ash` usually lands you at a root prompt with the filesystem mounted. Variations include `rdinit=/bin/sh` for initramfs-based boots. You can set this for one boot without persisting it, which avoids bricking the normal boot path. Continue at [Getting a Shell](getting-a-shell.md).

## Reading the environment for secrets

`printenv` sometimes reveals more than boot logic:

- Hardcoded passwords or an `bootargs` that disables the login.
- A `bootcmd` that shows exactly how and from where the kernel loads.
- Network boot settings pointing at vendor infrastructure.

## Persisting versus not

- **`setenv` + `boot`** changes the environment only in RAM for this session, so the next normal power cycle is unaffected. Safe for experiments.
- **`saveenv`** writes the environment to flash permanently. Powerful, but a mistake here can brick normal boot. Keep a dump of the original environment first.

## Defender's view

Lock the bootloader on production units: set `bootdelay` to 0 *and* require a password for the prompt (U-Boot supports a prompt password and locked-down environment), disable interactive commands, and do not ship `init=/bin/sh` reachable. An open U-Boot prompt is equivalent to root plus firmware read/write.
