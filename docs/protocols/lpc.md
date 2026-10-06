# LPC

LPC (Low Pin Count) is a legacy bus on PC-class motherboards that connects the chipset to low-bandwidth peripherals: the BIOS/UEFI flash (on older boards), the super I/O chip, embedded controllers, and, most interestingly for a hardware hacker, the **TPM**. The headline use of LPC in security work is sniffing the bus to capture secrets a TPM hands to the CPU in the clear.

## How it works

LPC is a synchronous bus derived from the old ISA bus, running at 33 MHz. The signals you care about:

| Signal | Role |
| --- | --- |
| `LCLK` | 33 MHz clock |
| `LFRAME#` | Start of a transaction |
| `LAD[3:0]` | 4-bit multiplexed address/data |
| `LRESET#` | Reset |

Address, data, and control are multiplexed over the four `LAD` lines across clock cycles. That multiplexing makes decoding more involved than SPI or I2C, but it is well documented and within reach of a decent [logic analyzer](../software/logic-analyzers.md) or an FPGA.

## The TPM sniffing use case

A Trusted Platform Module stores keys and measurements, and on many systems it communicates with the CPU over LPC (newer boards use SPI, which is sniffable the same way). The crucial weakness: by default the TPM-to-CPU link is **not encrypted**. Anything the TPM sends back crosses the bus in plaintext.

The classic target is **BitLocker** in its default TPM-only mode. At boot, the TPM releases the Volume Master Key to the CPU over the bus once the platform measurements match. If you sniff LPC (or SPI) during that boot, you capture the key in transit and can decrypt the drive. The full procedure lives on the [TPM Sniffing](../attacks/tpm-sniffing.md) page; LPC is simply the physical layer it rides on.

## Sniffing LPC

- **Tap points.** The TPM header (many motherboards have a populated or unpopulated TPM connector) exposes the LPC signals conveniently. Otherwise probe the LPC pins on the chipset, super I/O, or BIOS flash.
- **Capture hardware.** Because LPC multiplexes four lines at 33 MHz, you want either a fast logic analyzer with an LPC decoder or, better, an FPGA-based sniffer. Community tools and bitstreams exist specifically for TPM/LPC sniffing (for example FPGA designs that decode LPC and extract the TPM response frames).
- **Trigger on the boot** so you capture the window where the key is released.

## Finding LPC on a board

- A **TPM header** (often labeled, 14 or 20 pin) is the giveaway and the easiest tap.
- The **BIOS flash** on older boards is LPC rather than SPI; its package and the chipset routing indicate the bus.
- The **super I/O** chip (keyboard, fan, legacy ports) hangs off LPC.

On modern boards LPC is increasingly replaced by **eSPI**, and discrete TPMs often sit on [SPI](spi.md) instead, which you sniff with the same logic-analyzer approach you already use for flash. Either way the principle is unchanged: an unencrypted bus between a secure element and the CPU leaks whatever crosses it.

## Related pages

- [TPM Sniffing](../attacks/tpm-sniffing.md)
- [SPI](spi.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
- [Chain of Trust and Secure Boot](../attacks/chain-of-trust.md)
- [Side-Channel Attacks](../attacks/side-channel.md)
