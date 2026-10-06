# Pin Reference

Every signal name you are likely to meet on a header, a test pad, or a datasheet, with a short explanation and a link to the bus it belongs to. Other pages link each pin they mention back to the matching entry here, so this page is the canonical glossary of pins. For physical layouts see the [Pinout Cheat Sheet](pinouts.md); for working out which bus an unknown header is, see [Interface Identification](interface-identification.md).

!!! tip "Naming is not standardized"
    The same signal has many names across vendors and datasheets. Aliases are listed with each entry. When a name ends in `#` or is written `nNAME`, the signal is **active low** (asserted when driven low).

## Power and ground

#### VCC { #vcc }
*Power · also VDD, V+.* The positive supply rail for a chip or interface. On a debug or memory header you usually confirm its voltage with a multimeter and match your adapter to it. On a [UART](../protocols/uart.md) header you normally leave it unconnected and power the board normally. See [Level Shifting](../fundamentals/level-shifting.md).

#### VDD { #vdd }
*Power · alias of VCC.* The supply pin name used on most IC datasheets (VDD for the core/logic supply). Functionally the [VCC](#vcc) of that chip.

#### GND { #gnd }
*Power · also VSS, 0V.* Ground, the common zero-volt reference. Every tool you attach must share this with the target, or signals float and captures are meaningless. The first pin to find on any unknown header. See [Finding and Identifying Pins](../fundamentals/identifying-pins.md).

#### VSS { #vss }
*Power · alias of GND.* The ground pin name used on most IC datasheets. Same node as [GND](#gnd).

#### VCCQ { #vccq }
*Power · [eMMC](../protocols/emmc.md).* The I/O supply for an eMMC device, separate from the core supply (VCC). Often 1.8 V or 3.3 V and must match the host's I/O level.

#### VPP { #vpp }
*Power · programming.* A higher programming/erase voltage some older flash and EPROM parts require. Usually left at normal logic level or unused on modern chips.

#### VTref { #vtref }
*Power · debug headers.* Target voltage reference on [JTAG](../protocols/jtag.md)/[SWD](../protocols/swd.md) headers. The debug probe senses this to set its I/O to the target's level rather than supplying power.

## UART

See [UART](../protocols/uart.md).

#### TX { #tx }
*UART · output · also TXD.* The line a device transmits serial data on. It bursts with activity the instant a device powers on, which makes it the easiest signal to spot. Wire it to your adapter's RX.

#### RX { #rx }
*UART · input · also RXD.* The line a device receives serial data on. Usually idle (high) at boot because nothing is typing yet. Wire it to your adapter's TX.

#### RTS { #rts }
*UART · output · flow control.* Request To Send, hardware flow control. On many adapters it is repurposed to drive a target's reset or boot pin during automated flashing.

#### CTS { #cts }
*UART · input · flow control.* Clear To Send, the counterpart to [RTS](#rts). Leaving hardware flow control enabled when the target does not drive CTS is a common reason a console looks dead.

#### DTR { #dtr }
*UART · output · modem/control.* Data Terminal Ready. Like [RTS](#rts), frequently reused to toggle reset or bootloader entry (for example the auto-reset circuit on many dev boards).

## SPI

See [SPI](../protocols/spi.md).

#### SCLK { #sclk }
*SPI · master output · also SCK, CLK.* The clock the SPI master drives; every bit is shifted on its edges. A steadily toggling line during access is a clock, a strong hint you are looking at a synchronous bus.

#### MOSI { #mosi }
*SPI · master to slave · also DI, SDI, SI.* Master Out, Slave In: data from the controller to the chip. On a flash chip's own datasheet this pin is labeled from the chip's view as DI.

#### MISO { #miso }
*SPI · slave to master · also DO, SDO, SO.* Master In, Slave Out: data from the chip back to the controller. On a flash chip it is labeled DO.

#### CS# { #cs }
*SPI · master output · active low · also SS, CE#, nCS.* Chip Select. Pulling it low activates one slave on the bus; there is no addressing on the wire, so CS picks the device. Holding the host's CS inactive is one way to free the bus for an [in-circuit read](../attacks/spi-flash-dump.md).

#### WP# { #wp }
*SPI flash · input · active low · doubles as IO2.* Write Protect. When asserted it blocks writes to the status register and protected regions. In quad mode this same pin becomes data line [IO2](#io2).

#### HOLD# { #hold }
*SPI flash · input · active low · also RESET#, doubles as IO3.* Pauses the chip without deselecting it. In quad mode it becomes data line [IO3](#io3). On some parts it is a reset instead.

#### IO0-IO3 { #io2 }
*Quad SPI · bidirectional.* In Quad SPI (QSPI) four data lines carry four bits per clock. IO0 and IO1 are [MOSI](#mosi) and [MISO](#miso); IO2 and IO3 are the repurposed [WP#](#wp) and [HOLD#](#hold). For plain dumping you use single-bit SPI and ignore IO2/IO3.

## I2C

See [I2C](../protocols/i2c.md).

#### SDA { #sda }
*I2C · bidirectional · open-drain.* Serial Data. Shared by every device on the bus and pulled high by a resistor; devices only pull it low. A pair of lines with pull-up resistors near a small chip is the I2C tell.

#### SCL { #scl }
*I2C · master output · open-drain.* Serial Clock, driven by the master. Together with [SDA](#sda) it forms the whole two-wire bus.

## 1-Wire

See [1-Wire](../protocols/one-wire.md).

#### DQ { #dq }
*1-Wire · bidirectional · open-drain.* The single data line that carries everything, and optionally powers the slave (parasitic power). Pulled high by a resistor; timing of the low pulses encodes the bits.

## JTAG

See [JTAG](../protocols/jtag.md).

#### TCK { #tck }
*JTAG · input (to target).* Test Clock, drives the JTAG state machine. A periodic clock on a quiet debug header.

#### TMS { #tms }
*JTAG · input.* Test Mode Select, steers the TAP state machine. Shares the pin with [SWDIO](#swdio) on ARM parts that offer both.

#### TDI { #tdi }
*JTAG · input.* Test Data In, data shifted into the chip (and down a daisy chain).

#### TDO { #tdo }
*JTAG · output.* Test Data Out, data shifted out of the chip. Shares the pin with [SWO](#swo) on some ARM headers.

#### TRST# { #trst }
*JTAG · input · active low · also nTRST.* Optional Test Reset that resets the TAP logic. Not all targets have it.

#### RTCK { #rtck }
*JTAG · output.* Return Test Clock, an adaptive-clocking echo some targets provide so the probe can sync its clock. Often present on 20-pin ARM headers.

## SWD

See [SWD](../protocols/swd.md).

#### SWDIO { #swdio }
*SWD · bidirectional.* The single data line of ARM Serial Wire Debug. Multiplexes both directions, which is why SWD needs only two active pins. Shares silicon with [TMS](#tms).

#### SWCLK { #swclk }
*SWD · input.* The SWD clock. Shares the pin with [TCK](#tck).

#### SWO { #swo }
*SWD · output · optional.* Serial Wire Output, a single-wire trace channel for printf-style debug messages. Useful but not required to debug. Shares the pin with [TDO](#tdo).

## Reset and boot control

#### nRESET { #nreset }
*Control · input · active low · also RESET, RST, nRST.* The chip reset line. Pulling it to its active level holds the processor in reset, which is how you free a shared bus or stop a CPU from fighting your [flash read](../attacks/spi-flash-dump.md). Present on most debug headers as well.

#### SRST# { #srst }
*Debug · input · active low.* System Reset on JTAG/SWD headers, which resets the whole target (as opposed to [TRST#](#trst), which resets only the TAP). Lets a debugger connect "under reset".

#### BOOT0 { #boot0 }
*Control · input · strapping · also BOOT, BOOT1.* A boot-mode select pin sampled at reset. On STM32, BOOT0 chooses between running user flash and entering the built-in [ROM bootloader](../attacks/bootloader-access.md); the ROM loader is sometimes a weaker interface worth reaching. Other families have equivalents.

#### GPIO0 { #gpio0 }
*Control · input · strapping (Espressif).* On [ESP32](../mcu/esp32-security.md)/ESP8266, the level of GPIO0 at reset selects normal boot versus the serial download (flashing) mode. Holding it low at reset drops the chip into its bootloader.

#### EN { #en }
*Control · input · also CE, CHIP_EN.* Chip enable / reset on Espressif and many other parts; driving it resets or powers down the chip. A convenient place to force a reset.

#### TEST { #test }
*Control · input (Nordic and others).* A mode/test pin. On some parts it must be at a defined level for normal operation; on others it gates factory test or debug features.

## eMMC and SD

See [eMMC and SD](../protocols/emmc.md).

#### CLK (eMMC) { #clk }
*eMMC/SD · input.* The clock the host drives to the card. One of the three lines (with [CMD](#cmd) and [DAT0](#dat0)) needed for a minimal 1-bit read.

#### CMD { #cmd }
*eMMC/SD · bidirectional.* The command and response line between host and card.

#### DAT0-DAT7 { #dat0 }
*eMMC/SD · bidirectional.* Data lines. 1-bit mode uses only DAT0, which is why an in-system eMMC read can get away with tapping just [CLK](#clk), [CMD](#cmd), and DAT0. Full speed uses all eight.

#### DS { #ds }
*eMMC · output.* Data Strobe, used in the fast HS400 mode so the card can clock data back to the host reliably. Not needed for a slow read.

#### CD { #cd }
*SD · output.* Card Detect, a mechanical switch line telling the host a card is inserted. Not a data signal.

## CAN (automotive)

See [CAN Bus](../automotive/can-bus.md).

#### CAN-H { #can-h }
*CAN · differential.* The "high" line of the CAN differential pair. On the OBD-II connector it is pin 6. A healthy bus reads about 60 ohm across CAN-H and CAN-L with power off (two 120 ohm terminators in parallel).

#### CAN-L { #can-l }
*CAN · differential.* The "low" line of the pair, OBD-II pin 14. A CAN bit is the voltage difference between CAN-H and CAN-L.

## USB

See [USB](../protocols/usb.md).

#### VBUS { #vbus }
*USB · power.* The USB 5 V supply from host to device. Useful to know for powering or detecting connection, and the reference for the data pair.

#### D+ { #d-plus }
*USB · differential · also DP.* The positive line of the USB data pair. Its idle state and pull-up identify the device speed.

#### D- { #d-minus }
*USB · differential · also DM.* The negative line of the USB data pair. D+ and D- carry all USB data differentially.

#### ID { #usb-id }
*USB OTG · input.* The On-The-Go identification pin that selects host or device role on a micro-USB OTG port. (Do not confuse with a chip "ID", a flash JEDEC ID, or a CAN arbitration ID, which are unrelated uses of the word.)

#### CC1/CC2 { #cc }
*USB-C · configuration.* The Configuration Channel pins on USB-C that negotiate orientation, role, and power delivery. SBU lines carry sideband/alternate-mode signals.

## MDIO (Ethernet PHY management)

See [MDIO](../protocols/mdio.md).

#### MDC { #mdc }
*MDIO · input.* Management Clock, driven by the Ethernet MAC to clock PHY register accesses.

#### MDIO (signal) { #mdio-signal }
*MDIO · bidirectional.* The management data line the MAC uses to read and write an Ethernet PHY's registers.

## LPC (Low Pin Count)

See [LPC](../protocols/lpc.md).

#### LCLK { #lclk }
*LPC · input.* The 33 MHz LPC clock. Fast enough that sniffing LPC (for example for [TPM key capture](../attacks/tpm-sniffing.md)) wants a quick analyzer or an FPGA.

#### LFRAME# { #lframe }
*LPC · input · active low.* Signals the start of an LPC transaction.

#### LAD0-LAD3 { #lad }
*LPC · bidirectional.* The four multiplexed address/data lines that carry the whole LPC transaction across several clock cycles.

#### LRESET# { #lreset }
*LPC · input · active low.* The LPC reset, usually tied to the platform reset.

## Parallel NAND

See [NAND and eMMC Dumping](../attacks/nand-emmc-dump.md).

#### CE# (NAND) { #ce }
*Parallel NAND · input · active low.* Chip Enable, selects the NAND device.

#### RE# { #re }
*Parallel NAND · input · active low.* Read Enable, clocks data out of the NAND.

#### WE# { #we }
*Parallel NAND · input · active low.* Write Enable, clocks commands, addresses, and data into the NAND.

#### CLE { #cle }
*Parallel NAND · input.* Command Latch Enable; when high, bytes on the I/O lines are interpreted as commands.

#### ALE { #ale }
*Parallel NAND · input.* Address Latch Enable; when high, bytes on the I/O lines are interpreted as addresses.

#### R/B# { #rb }
*Parallel NAND · output · active low busy.* Ready/Busy, tells the host when an internal operation (program, erase, read) has finished.

#### I/O0-I/O7 { #nand-io }
*Parallel NAND · bidirectional.* The 8-bit (or 16-bit) bus that carries commands, addresses, and data to and from the NAND, multiplexed by [CLE](#cle)/[ALE](#ale).

## Miscellaneous

#### NC { #nc }
*No Connect.* A pin that is intentionally not connected inside the package. Leave it alone; on some parts it is secretly a factory/test pin, so do not assume it is safe to drive.

#### XTAL1/XTAL2 { #xtal }
*Clock.* The crystal oscillator pins. Relevant to [clock glitching](../attacks/fault-injection.md) only when the chip runs from an external clock you can reach; parts on an internal oscillator cannot be clock-glitched this way.

#### OE# { #oe }
*Output Enable · active low.* Enables a chip's or buffer's outputs. Common on parallel memories and bus transceivers.

## Related pages

- [Pinout Cheat Sheet](pinouts.md)
- [Finding and Identifying Pins](../fundamentals/identifying-pins.md)
- [Interface Identification](interface-identification.md)
- [Logic Levels and Level Shifting](../fundamentals/level-shifting.md)
- [Glossary](glossary.md)
