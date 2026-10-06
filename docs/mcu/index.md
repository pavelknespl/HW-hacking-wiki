# MCU Targets

On a microcontroller the firmware lives inside the chip, so there is no external flash to clip. Getting the code means reading the internal flash over a [debug port](../protocols/swd.md), and the obstacle is always the vendor's **read-out protection**: a fuse or option byte that disables debug reads. Each silicon family implements this differently, with different bypasses and different fuse semantics. The generic [fault injection](../attacks/fault-injection.md) page explains the technique; these pages cover the specific, real-world targets.

<div class="grid cards" markdown>

-   __STM32 RDP__

    ST's Readout Protection levels 1 and 2, and the glitch that defeats level 1.

    [:octicons-arrow-right-24: Read](stm32-rdp.md)

-   __nRF52 APPROTECT__

    Nordic's access-port protection and its well-known single-glitch bypass.

    [:octicons-arrow-right-24: Read](nrf52-approtect.md)

-   __ESP32 Security__

    Flash encryption and secure boot on Espressif parts, and their boundaries.

    [:octicons-arrow-right-24: Read](esp32-security.md)

-   __AVR and PIC Fuses__

    Lock bits on classic 8-bit parts and what they do and do not protect.

    [:octicons-arrow-right-24: Read](avr-pic-fuses.md)

</div>

## The common pattern

Whatever the vendor, the story rhymes:

1. The chip has a **debug interface** ([SWD](../protocols/swd.md) or [JTAG](../protocols/jtag.md)) that can read internal flash.
2. A **protection fuse** disables that read when set.
3. The protection is enforced by a **check** somewhere in the boot or debug-enable path.
4. If that check can be **skipped with a glitch**, or if the fuse logic has a flaw, the flash reads out anyway.

So the questions for any MCU target are: what is the protection called, what exactly does it block, is it the stronger "permanent" level or the weaker reversible one, and is there a known fault-injection or logic bypass. These pages answer that per family. Start by identifying the exact part during [recon](../attacks/recon.md), then read the matching page.

## Related pages

- [SWD](../protocols/swd.md)
- [JTAG](../protocols/jtag.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [Firmware Extraction](../attacks/firmware-extraction.md)
- [Chain of Trust and Secure Boot](../attacks/chain-of-trust.md)
