# Soldering and Rework

Sooner or later a clip will not hold, a chip must come off the board, or a tiny pad needs a wire. The physical skills, soldering, hot air rework, and chip-off, are what turn "I can see the memory" into "I have the memory." This page covers the techniques and the adapters that let you read a chip once it is off the board. None of it is hard with practice; buy a scrap board and ruin it a few times first.

## Soldering fine wires

The everyday task is tacking a thin wire to a small pad or pin to reach a [UART](../protocols/uart.md), a [debug port](../protocols/jtag.md), or a test point.

- **Iron:** a temperature-controlled station with a fine conical or small chisel tip. Around 320 to 350 C for leaded solder, hotter for lead-free.
- **Flux is the secret.** A dab of no-clean flux makes solder flow to the pad and not bridge. Most bad joints are a flux problem.
- **Wire:** thin enameled magnet wire (0.1 mm) for dense pads, or fine stranded for headers. Tin the pad, tin the wire, then join.
- **Strain relief:** glue or tape the wire down near the joint so a tug lifts the pad's neighbor, not the pad.

!!! tip "Protect the joint"
    A blob of hot glue or UV resin over a delicate tap keeps it alive through handling. Reworked debug wires break at the pad constantly without it.

## Hot air rework

Hot air is how you remove and place surface-mount chips (SOIC, QFN, and with care BGA). A hot air station blows heated air to melt all of a chip's joints at once.

Removing a [SOIC-8 flash](../attacks/spi-flash-dump.md#desoldering-the-chip):

1. **Protect neighbors** with Kapton tape; shield plastic connectors.
2. **Add flux** around the chip and a little fresh solder to the pins (it improves heat transfer).
3. **Heat evenly**, circling the chip, at an appropriate temperature (lead-free often around 350 to 400 C with medium airflow). Do not blast one spot.
4. When the solder flows, **lift the chip** with tweezers. It should come free with no force; if it resists, keep heating rather than pulling.
5. **Clean the pads** with wick and flux before reading or reinstalling.

A **preheater** under the board spreads the thermal load, which protects the PCB and is nearly essential for larger chips and BGAs.

## Chip-off

"Chip-off" means removing the memory chip entirely to read it in a socket or adapter, the reliable fallback when an [in-circuit read](../attacks/spi-flash-dump.md#the-in-circuit-power-problem) will not stabilize or the chip is a package you cannot clip.

- **NOR/SPI (SOIC-8, WSON):** hot air off, read in a SOIC socket or on an adapter.
- **Parallel NAND (TSOP-48):** hot air off, read in a [programmer](../attacks/nand-emmc-dump.md) with a TSOP-48 adapter.
- **eMMC (BGA):** the hardest; see reballing below.

Chip-off is destructive to the installation but not to the data. It always works when in-circuit does not, at the cost of rework risk and reassembly.

## BGA and reballing

Ball Grid Array packages (eMMC, many SoCs) have their contacts underneath as a grid of solder balls, so there is nothing to clip and removal/replacement is the advanced end of rework.

To read a BGA chip off-board you typically **reball** it:

1. Remove the chip with hot air and a preheater.
2. Clean the old solder off the pads with wick and flux until flat.
3. Place a **reballing stencil** over the pads, apply solder paste or lay solder balls, and reflow to form fresh, even balls.
4. Seat the reballed chip in a **BGA socket/adapter** for its package (for example a BGA-153/169 eMMC adapter) and read it in a programmer or [card reader](../protocols/emmc.md).

Reballing takes practice and the right stencil for the exact ball layout. For [eMMC](../protocols/emmc.md) specifically, an in-system read via `CLK/CMD/DAT0` is well worth attempting first precisely because it avoids all of this.

## Adapters and sockets

The bridge from a desoldered chip to your programmer:

| Package | Adapter |
| --- | --- |
| SOIC-8 / SOIC-16 | SOIC-to-DIP socket or clip |
| WSON-8 | WSON socket adapter |
| TSOP-48 | TSOP-48 socket for the programmer |
| BGA-153 / BGA-169 (eMMC) | eMMC BGA reader/adapter |
| Various | Universal programmer (XGecu T48/T56) with package adapters |

A universal programmer with a handful of adapters covers most chips you will pull.

## Safety

Reiterating the essentials from [Lab Setup](../getting-started/lab-setup.md#personal-and-electrical-safety): ventilate solder fumes, wash hands after handling leaded solder, wear eye protection during rework, and treat any mains section of a board as dangerous. Hot air is genuinely hot and melts plastic and skin; mind where the nozzle points.

## Related pages

- [Lab Setup and Safety](../getting-started/lab-setup.md)
- [The Toolkit](../getting-started/toolkit.md)
- [SPI Flash Dumping](../attacks/spi-flash-dump.md)
- [NAND and eMMC Dumping](../attacks/nand-emmc-dump.md)
- [Chip Identification](chip-identification.md)
