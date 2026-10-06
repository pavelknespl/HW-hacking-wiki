# Reconnaissance

Recon is the cheap work that makes every later step faster and safer. Before you connect a single wire, you want to know what the device is, what runs its code, where that code lives, and what interfaces exist. This page is the information-gathering phase of the [methodology](../getting-started/methodology.md), covering both the board in front of you and the public record about it.

## Gather the public record first

A surprising amount is knowable without opening the case.

- **Model and FCC ID.** The label carries a model number and, for devices sold in the United States, an FCC ID. The [FCC OET database](https://www.fcc.gov/oet/ea/fccid) returns internal photographs, test reports, and sometimes block diagrams or schematics. Internal photos alone often reveal the SoC, flash, and debug headers before you touch the device.
- **Firmware downloads.** Vendor support pages, update servers, and community mirrors frequently host firmware images. A downloadable image lets you start [analysis](firmware-analysis.md) immediately and tells you what to expect on the chip.
- **Community databases.** WikiDevi, the OpenWrt table of hardware, and teardown sites map many consumer devices to their chipsets and quirks.
- **CVEs and writeups.** Prior research on the same chipset or product line shortcuts your work. Search the SoC name and the model for known issues.

## Open it and document

1. Disassemble carefully, noting screws, clips, and cable routing so you can reassemble.
2. Photograph both sides of the board under even light at high resolution. These become your working reference.
3. Record the model, revision, and any date codes.

See [PCB Recon](../fundamentals/pcb-recon.md) for the physical read of the board.

## Inventory the chips

Identify every significant chip with [Chip Identification](../fundamentals/chip-identification.md) and build a table in your [notebook](../getting-started/lab-setup.md#keep-a-lab-notebook):

| Chip | Marking | Identity | Role | I/O voltage |
| --- | --- | --- | --- | --- |
| U1 | large BGA | SoC | runs firmware | 1.8 V I/O |
| U2 | `W25Q128` | Winbond SPI NOR | firmware store | 3.3 V |
| U3 | `24C64` | I2C EEPROM | config / MAC | 3.3 V |

This table drives everything: it names your extraction target, your console voltage, and your analysis architecture.

## Map the interfaces

Walk the board for ways in, in order of likely payoff:

1. **UART header or pads** near the SoC, a 3 or 4 pin group whose TX bursts at boot. The fastest path to a shell. See [Finding and Identifying Pins](../fundamentals/identifying-pins.md).
2. **External flash** you can clip, for a direct [firmware dump](spi-flash-dump.md).
3. **JTAG/SWD** header or test points for full [memory access](../protocols/jtag.md).
4. **I2C/SPI buses** worth [sniffing](../protocols/i2c.md) for secrets in transit.

## Measure the voltages

With the device powered, find ground and measure the logic rails so you connect everything at the right level. The flash rail and the console rail are the two numbers you most need. See [Level Shifting](../fundamentals/level-shifting.md). This is also a safety step: it prevents the single most common way to destroy a target.

## Capture a boot log

If you found a UART, read the boot log before anything invasive. It is pure recon gold:

- Confirms the [architecture](../fundamentals/architectures.md) and SoC.
- Names the bootloader and whether it can be interrupted.
- Prints the flash **partition layout**, so you know what lives where.
- Shows kernel version, command line, and driver messages.
- Occasionally ends at a root shell.

## Decide the plan

Finish recon with a concrete plan written down:

- "Firmware is in the Winbond SPI NOR; clip and dump with flashrom at 3.3 V."
- "UART at 115200 gives a login prompt; recover the password from the dump."
- "No external flash; internal MCU flash over SWD, expect RDP, plan for glitching if locked."

A clear plan keeps you from thrashing between techniques and from reaching for the oscilloscope when a UART shell was sitting open the whole time.
