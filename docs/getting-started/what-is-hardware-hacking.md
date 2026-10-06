# What Is Hardware Hacking

Hardware hacking is the practice of attacking or analyzing a physical device through its electronics rather than only through its network or software surface. When you have the board in your hands, assumptions that software security relies on start to break: you can read memory directly, pause the processor, watch the chips talk to each other, and feed the system power and clock signals it was never designed to receive.

The point is almost never the solder joint itself. It is what access to the hardware unlocks: the firmware that runs the device, the keys stored inside it, the way it authenticates to a cloud service, and the bugs that are invisible until you can see the code.

## Why the hardware is the soft underbelly

A networked service is written defensively because its authors assume hostile input from strangers. The same product's circuit board is usually designed under the opposite assumption, that nobody untrusted will ever have physical access. That gap is where most of the interesting findings live:

- **Debug interfaces left enabled.** UART consoles, JTAG, and SWD ports are invaluable during development and frequently ship enabled on production units.
- **Unencrypted external memory.** Firmware and configuration often sit in a SPI flash or NAND chip in the clear, readable with a clip and a cheap programmer.
- **Trusting boot chains.** Many devices load code from external storage without verifying it, so modifying that storage modifies the device.
- **Secrets at rest.** Wi-Fi passwords, API tokens, private keys, and customer data are regularly stored in plaintext on the flash.

## What you can do once you are in

The capabilities you gain tend to compound. A typical progression looks like this:

1. A serial console reveals the boot log, the Linux version, and sometimes a root prompt.
2. A firmware dump gives you every binary on the device to analyze offline.
3. Analysis uncovers hardcoded credentials, a command injection bug, or a cryptographic weakness.
4. That weakness turns into a remote exploit that needs no physical access at all, which is the real prize for a product security assessment.

Hardware access is therefore both an end in itself (recovering data, unlocking a device) and a means to understand software you could not otherwise see.

## The major techniques at a glance

| Technique | What it targets | Typical outcome |
| --- | --- | --- |
| [Serial console (UART)](../protocols/uart.md) | Debug output and shells | Boot logs, sometimes a root shell |
| [JTAG](../protocols/jtag.md) / [SWD](../protocols/swd.md) | CPU debug interface | Halt the CPU, read and write memory, dump firmware |
| [SPI flash dumping](../attacks/spi-flash-dump.md) | External NOR flash | Full firmware image |
| [NAND / eMMC dumping](../attacks/nand-emmc-dump.md) | Larger external storage | Full filesystem image |
| [Bus sniffing (I2C/SPI)](../protocols/i2c.md) | Inter-chip communication | Secrets in transit, EEPROM contents |
| [Bootloader attacks](../attacks/bootloader-access.md) | U-Boot and friends | Boot modified code, dump memory, get a shell |
| [Fault injection](../attacks/fault-injection.md) | Secure boot, read-out protection | Skip a security check, defeat a lock bit |
| [Side-channel analysis](../attacks/side-channel.md) | Cryptographic implementations | Recover keys without reading them directly |

## A note on realism

Not every device yields to the first technique you try, and a surprising number give up everything to the very first one. Experienced practitioners do not reach for an oscilloscope and a glitcher before checking whether there is simply a UART header with an open root shell. Start cheap and non-invasive, escalate only as the target forces you to, and keep careful notes so you can retrace your steps. The [Methodology](methodology.md) page formalizes this.
