# Chip Identification

The text printed on a chip is the thread you pull to unravel the whole device. From a part number you get a datasheet, from the datasheet you get the pinout, the voltage, the interface, and the internal block diagram. Learning to read these markings quickly is one of the highest-leverage skills on the bench.

## Anatomy of a chip marking

Surface mount chips typically carry several lines of laser-etched text:

- **Manufacturer logo or prefix.** A small symbol or a few letters identifying the maker (for example, a Winbond `W`, a Macronix `MX`, a Microchip `AT`).
- **Part number.** The line you care about most. It encodes the family, capacity or model, package, and often speed grade.
- **Date and lot codes.** A four digit `YYWW` date and a batch code. Useful for provenance, not for identification.

Example: a flash chip marked `W25Q128JVSIQ` decodes as Winbond (`W`), serial flash (`25Q`), 128 Mbit (`128`), a generation code (`JV`), and package and temperature suffixes (`SIQ`, an SOIC-8).

## The common chip categories and their tells

| Category | Typical package | How to recognize |
| --- | --- | --- |
| **Main SoC / CPU** | Large BGA or QFP, often the biggest chip | Many pins, near the DRAM, often under a shield or heatsink |
| **DRAM** | BGA or TSOP, usually next to the SoC | Marked with Samsung/Hynix/Micron codes, high pin count |
| **SPI NOR flash** | SOIC-8 (sometimes SOIC-16 / WSON) | 8 legs, marked `25` series (`W25`, `MX25`, `GD25`, `EN25`) |
| **SPI NAND flash** | SOIC-8 / WSON | Marked with a `25` style code but larger capacity, different command set |
| **Parallel NAND** | TSOP-48 | 48 legs, marked `29F`, `TC58`, Samsung `K9` series |
| **eMMC** | BGA-153 / BGA-169 | Ball grid, marked Samsung `KLM`, SK Hynix, Micron `JY`/`FE` |
| **EEPROM** | SOIC-8 / TSSOP | Small, marked `24Cxx` (I2C) or `93Cxx` (Microwire) |
| **PMIC** | QFN | Near the power input, many inductors and capacitors around it |

## Decoding by prefix

Serial flash is the chip you will identify most often, because it is the usual firmware store on routers and cameras. The leading letters map to a maker:

| Prefix | Manufacturer |
| --- | --- |
| `W25` | Winbond |
| `MX25` | Macronix |
| `GD25` | GigaDevice |
| `EN25` | Eon |
| `S25` / `S25FL` | Spansion / Infineon |
| `IS25` | ISSI |
| `AT25` / `AT45` | Adesto / Microchip |
| `N25Q` / `MT25Q` | Micron |
| `BY25` / `XM25` | Boya / XMC |

The `25` in the middle marks an SPI serial flash using the common command set, which is exactly what [`flashrom`](../software/flashrom.md) and a [SOIC clip](../attacks/spi-flash-dump.md) expect.

## When the markings are gone

Manufacturers sometimes sand off or black-top chips, or the laser etch is faint. Options:

- **Angle a light across the surface.** Faint etching becomes legible with raking light and a loupe or microscope.
- **Count pins and measure the package.** The footprint plus pin count narrows the category. An 8-pin SOIC near the SoC is almost certainly a SPI flash.
- **Probe the pins.** A quick continuity and signal check against the standard SPI flash pinout confirms a guess even with no markings. See [Finding and Identifying Pins](identifying-pins.md).
- **Ask `flashrom`.** On an unmarked SPI chip, `flashrom` reads the JEDEC ID directly from the chip and tells you the manufacturer and capacity, no markings needed.

```console
$ flashrom -p ch341a_spi
Found Winbond flash chip "W25Q128.V" (16384 kB) ...
```

## Turning a part number into a datasheet

- Search the full part number plus the word "datasheet". Manufacturer PDFs are the ground truth.
- For SoCs, vendor datasheets are often under NDA, but WikiDevi, the OpenWrt hardware database, and the device's FCC filing frequently fill the gap. See [PCB Recon](pcb-recon.md) for the FCC trick.
- Sites like [datasheetspdf](https://www.datasheetspdf.com/) and manufacturer portals cover the common memory and logic parts.

## What to record

For each chip, note in your [lab notebook](../getting-started/lab-setup.md#keep-a-lab-notebook): the full marking, the decoded identity, the package, the I/O voltage from the datasheet, and its role on the board. That table becomes your map for the rest of the assessment.
