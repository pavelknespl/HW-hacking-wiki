# AVR and PIC Fuses

The classic 8-bit microcontrollers, Microchip's AVR (the ATmega/ATtiny behind Arduino) and PIC families, protect their firmware with **lock bits** and configure their behavior with **fuses**. These are simpler than the Cortex-M protections, but the same questions apply: what does the lock actually block, and can it be bypassed.

## Fuses versus lock bits

Two related concepts, often confused:

- **Fuses** configure the chip: clock source, brown-out detection, whether the reset pin is a reset or a GPIO, bootloader size, and so on. They are not primarily a security feature, but one wrong fuse write can make a chip hard to reprogram (see below).
- **Lock bits** are the security feature: they control whether the flash and EEPROM can be read back or reprogrammed through the programming interface.

## AVR lock bits

On AVR, two lock bits (LB1, LB2) set the protection mode:

| Mode | Effect |
| --- | --- |
| No lock | Flash and EEPROM read/write freely via ISP/SPI programming. |
| LB1 programmed | Further programming disabled. |
| LB1+LB2 programmed | Programming **and verification (readback) disabled**. Firmware is protected from a simple read. |

Key points:

- With lock bits fully set, an ISP read returns protected/garbage data rather than the firmware.
- **A chip erase clears the lock bits, but it also erases the flash**, so you cannot unlock and read the original code the easy way. Same dilemma as [STM32 RDP level 1](stm32-rdp.md).
- Lock bits gate the standard programming interfaces, but they have historically been **weaker against physical attacks**. [Fault injection](../attacks/fault-injection.md) during the lock-bit check, and in some documented cases careful timing or voltage manipulation during readback, has recovered firmware from locked AVRs. Older parts were not designed with glitch resistance in mind.

## PIC code protection

PIC devices have an analogous **CP (Code Protection)** configuration bit in their config words. When set, the programmer cannot read the program memory back. As with AVR:

- A bulk erase clears protection but wipes the code.
- Various PIC families and specific devices have had published readback and glitch-based extraction techniques; applicability is highly part-specific, so identify the exact device.

## The self-bricking fuse trap

A practical hazard rather than a security control: misconfiguring certain fuses makes a chip appear dead to your programmer.

- Setting the AVR clock fuse to an **external clock/crystal** that is not present means the chip will not run, and ISP needs a running clock. Recovery: feed an external clock into the XTAL pin, or use **high-voltage programming** (HVPP/HVSP), which does not depend on the configured clock.
- Disabling the **reset pin** (RSTDISBL) or the ISP interface (SPIEN) locks out ISP entirely; recovery again means high-voltage programming.

So when a cheap AVR target seems bricked, it is often a fuse mistake, and a high-voltage programmer revives it.

## Workflow

1. Identify the exact part (ATmega/ATtiny/PIC model).
2. Read fuses and lock bits with the programmer (`avrdude` for AVR, MPLAB/pickit tooling for PIC):

    ```bash
    avrdude -c usbasp -p m328p -U lfuse:r:-:h -U lock:r:-:h
    ```

3. If unlocked, read flash and EEPROM directly.
4. If locked, you cannot read through the normal interface without erasing; evaluate a [fault-injection](../attacks/fault-injection.md) readback attack appropriate to that specific part, or accept the limit.

## Defender's view

Set the readback lock bits (LB1+LB2 on AVR, CP on PIC), but know that on older 8-bit parts this is not glitch-proof. For firmware that must stay secret on a hostile-access device, prefer a modern MCU with hardened protection over relying on legacy lock bits alone.

## Related pages

- [MCU Targets](index.md)
- [STM32 RDP](stm32-rdp.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [CPU Architectures](../fundamentals/architectures.md)
- [Memory Types](../fundamentals/memory-types.md)
