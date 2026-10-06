# MCU Debug Interfaces (SWIM, UPDI, PDI, debugWIRE)

[JTAG](jtag.md) and [SWD](swd.md) dominate ARM, but the 8-bit and some other microcontroller worlds use their own, often single-wire, programming and debug interfaces. If you meet an STM8, an AVR, or an older Atmel part, these are what you connect to. They do the same job (program flash, read memory, debug) over fewer pins, and the same [read-out protection](../mcu/index.md) questions apply.

## SWIM (STMicro 8-bit)

**SWIM** (Single Wire Interface Module) is the debug/programming interface on **STM8** microcontrollers.

- **One data line** (SWIM) plus reset, ground, and reference voltage. Bidirectional, bit-timing based.
- Driven by ST-Link (the same probe used for [SWD](swd.md) on STM32) through tools like `stm8flash` and OpenOCD's STM8 support.
- Read-out protection on STM8 is a flash option byte; like [STM32 RDP](../mcu/stm32-rdp.md), disabling it to read triggers a mass erase, so a locked part resists a simple read and invites [glitching](../attacks/fault-injection.md).

```bash
stm8flash -c stlink -p stm8s003f3 -r dump.bin    # read flash (if unprotected)
```

## UPDI (modern AVR / ATtiny / ATmega)

**UPDI** (Unified Program and Debug Interface) is the single-wire interface on newer **AVR** parts (the ATtiny 0/1/2-series, modern ATmega, AVR-Dx).

- **One pin** (shared with a GPIO/reset function) plus ground. Serial, UART-like signaling.
- Programmed with `pymcuprog`, `jtag2updi` (an Arduino as a UPDI programmer), or a commercial Atmel-ICE.
- A **lock bit** disables reading; clearing it erases the flash. Fuses configure the chip and can disable UPDI itself (recoverable with high-voltage UPDI on some parts). See the [AVR/PIC fuses](../mcu/avr-pic-fuses.md) page.

```bash
pymcuprog read -t uart -u /dev/ttyUSB0 -d attiny412 -f dump.hex
```

## PDI (XMEGA and some ATmega/ATtiny)

**PDI** (Program and Debug Interface) is the two-wire (data + clock, the clock shared with reset) interface on **XMEGA** and some earlier parts, the predecessor to UPDI.

- Two lines, programmed with Atmel-ICE, `avrdude` (with a PDI-capable programmer), or `jtag2updi`-style adapters.
- Same lock-bit read protection story.

## debugWIRE (classic AVR)

**debugWIRE** is Atmel's single-wire **debug** protocol on classic ATtiny/ATmega parts, carried over the **RESET** pin.

- One wire (RESET), used for on-chip debugging (breakpoints, single-step) in addition to the normal [SPI/ISP](spi.md) programming.
- Enabling debugWIRE takes over the reset pin, which can lock you out of ISP until you disable it again (a common gotcha); an Atmel-ICE or a careful fuse dance recovers it.
- Mostly of interest for live debugging of 8-bit AVR firmware you can already read.

## Identifying which one you have

The MCU part number decides the interface: STM8 means SWIM, modern ATtiny/ATmega means UPDI, XMEGA means PDI, classic ATmega/ATtiny means ISP plus optional debugWIRE. Identify the chip first (see [Chip Identification](../fundamentals/chip-identification.md)), then pick the matching programmer. Physically they are a small number of pins (often a 1 to 4 pin header or test pads) near the MCU; find them as you would any [debug header](../fundamentals/identifying-pins.md).

## Related pages

- [JTAG](jtag.md)
- [SWD](swd.md)
- [AVR and PIC Fuses](../mcu/avr-pic-fuses.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [Finding and Identifying Pins](../fundamentals/identifying-pins.md)
