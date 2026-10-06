# PCB Recon

A circuit board is a map if you know how to read it. Before you probe anything, spend time understanding the layout: where power enters, where the processor sits, which chips hold code, and where the designers left test points and headers. Good recon turns later steps from guesswork into targeted work.

## Open it up and document

1. **Get the board out cleanly.** Note screw locations, clip positions, and ribbon cables. Photograph each step so you can reassemble.
2. **Photograph both sides** at high resolution with even lighting. These photos are your reference for the rest of the job and let you work away from the bench.
3. **Note shields and heatsinks.** Metal cans hide the SoC and radios. They usually unclip or desolder. Look underneath for the most important chips.

## Map the major components

Walk the board and classify every significant chip using [Chip Identification](chip-identification.md). Build a quick inventory:

- **The SoC / CPU.** The largest chip, often shielded, surrounded by the densest routing and the DRAM. This runs the firmware.
- **DRAM.** Right next to the SoC, high pin count. Not usually a direct target but confirms the class of device.
- **Flash / storage.** The firmware store. A SOIC-8 is a [SPI flash](../attacks/spi-flash-dump.md); a TSOP-48 is parallel [NAND](memory-types.md); a BGA is likely [eMMC](../protocols/emmc.md). This is a primary target. See [Memory Types](memory-types.md).
- **PMIC and regulators.** Near the power input, surrounded by inductors and capacitors. Tells you the supply rails and voltages.
- **Radios.** Wi-Fi, Bluetooth, cellular, or sub-GHz modules, each with an antenna trace or connector.
- **EEPROM.** Small 8-pin chips often hold configuration, MAC addresses, or calibration data.

## Find the power rails

Knowing the voltages is a prerequisite for connecting anything safely.

- Set your multimeter to DC volts, clip the black probe to a known **ground** (a large copper pour, a shield can, or the negative terminal of the input), and measure the rails with the device powered.
- Common rails: the input (often 5 V or 12 V), a 3.3 V logic rail, and lower rails (1.8 V, 1.2 V, 1.1 V) for the SoC core.
- Note which rail the flash chip and the debug header sit on. That voltage is what your adapters must match. See [Level Shifting](level-shifting.md).

## Spot test points and headers

Designers leave access for their own bring-up and factory test, and that access is your way in.

- **Rows of 3 to 6 pads or pins** near the SoC are very often a [UART](../protocols/uart.md) console or a [JTAG](../protocols/jtag.md)/[SWD](../protocols/swd.md) header.
- **Unpopulated footprints** (empty through-holes or pads) frequently mark debug interfaces removed from production but still wired to the SoC.
- **Labeled silkscreen** is a gift: `TX`, `RX`, `GND`, `TCK`, `TMS`, `SWDIO`, `CLK`, and `VCC` are common. Even cryptic labels like `J4` or `TP12` are worth probing.
- **Castellated edges and pogo pads** are factory test points. A pogo jig makes repeatable contact.

## Trace the connections

You often need to know which test point goes where.

- **Continuity testing.** With the device off, use the multimeter's continuity beep to check whether a test pad connects to a known pin on the flash, the SoC, or a header. This is how you confirm that `TP7` is really the flash `CLK`, for example.
- **Follow the traces visually** under a microscope or in your photos. Short traces between the SoC and a header usually carry the debug signals.
- **Use the FCC filing.** For devices sold in the United States, the FCC ID printed on the label unlocks the manufacturer's internal photos, test reports, and sometimes block diagrams on the [FCC OET database](https://www.fcc.gov/oet/ea/fccid). This can hand you the board layout and chip set without any probing.

## Look for obvious wins before anything invasive

- A populated, labeled [UART](../protocols/uart.md) header with a boot log is the fastest path to a shell.
- A socketed or clip-accessible [SPI](../protocols/spi.md) flash means a firmware dump with no cooperation from the CPU.
- A visible [JTAG](../protocols/jtag.md) header on a Linux device may give full memory access.

Only when these easy paths are closed do you escalate to desoldering, bus sniffing, or physical attacks. Finish recon with a clear plan: this is the console, this is the flash, this is the debug port, and this is the voltage everything runs at.


## Related pages

- [Finding and Identifying Pins](identifying-pins.md)
- [Chip Identification](chip-identification.md)
- [Reconnaissance](../attacks/recon.md)
- [Logic Levels and Level Shifting](level-shifting.md)
