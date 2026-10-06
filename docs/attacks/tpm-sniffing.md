# TPM Sniffing

A Trusted Platform Module is a secure chip that stores keys and platform measurements and releases secrets only when the system is in a trusted state. The irony a hardware hacker exploits: on most systems the TPM talks to the CPU over an **unencrypted bus**, so a secret the TPM releases can be captured in transit. The headline target is BitLocker's default TPM-only mode, where the disk encryption key crosses the bus in the clear at boot.

!!! warning "Authorized use only"
    This is for recovering data from machines you own or are authorized to assess. Capturing someone else's disk key is theft of their data.

## Why it works

A discrete TPM is a separate chip connected to the CPU over a bus, historically [LPC](../protocols/lpc.md) and on modern boards [SPI](../protocols/spi.md) (or I2C on some systems). The TPM protocol supports encrypted sessions, but they are optional and frequently unused. In the default configuration the data between the TPM and the CPU is plaintext.

So when the TPM decides the platform is trustworthy and releases a key, that key travels across accessible PCB traces where a logic analyzer can read it.

## The BitLocker case

BitLocker in **TPM-only** mode (the default, with no PIN or USB key) is the canonical example:

1. At boot, the firmware and bootloader extend measurements into the TPM's PCRs.
2. If the measurements match the sealed policy, the TPM **unseals and releases the Volume Master Key** to the CPU.
3. That key crosses the TPM bus in the clear.

Sniff the bus during that boot window and you capture the VMK, then decrypt the drive offline. The attack needs only brief physical access and cheap hardware, which is why TPM-only BitLocker is considered weak against a physical attacker, and why adding a **pre-boot PIN** (which never crosses the bus) defeats it.

## Doing the capture

1. **Locate the bus.** Find the TPM: a discrete module on a header, or a surface-mount TPM chip. Determine whether it is [LPC](../protocols/lpc.md) or [SPI](../protocols/spi.md) from the chip and routing. A populated TPM header is the easiest tap.
2. **Tap the signals.**
    - **SPI TPM**: tap `CLK`, `MOSI`, `MISO`, `CS` like any [SPI sniff](../protocols/spi.md#sniffing-an-spi-bus). This is the common modern case and the easiest, because SPI is simple to decode.
    - **LPC TPM**: tap `LCLK`, `LFRAME#`, `LAD[3:0]`; the multiplexed 33 MHz bus needs a faster analyzer or an FPGA sniffer. See [LPC](../protocols/lpc.md).
3. **Capture at boot.** Trigger on reset and record the window where the TPM releases the key.
4. **Decode and extract.** Decode the bus, find the TPM command/response for the unseal, and pull the key bytes from the response. Community tooling exists for both SPI and LPC TPM extraction (including ready-made FPGA bitstreams and parsers that locate the key in a capture).
5. **Decrypt.** Use the recovered key with the appropriate tool (for BitLocker, `dislocker` or `bdemu` with the VMK/FVEK) to mount the volume.

## Beyond BitLocker

The same plaintext-bus weakness exposes any secret a TPM releases without an encrypted session: disk keys from other full-disk encryption schemes, attestation secrets, and keys used by measured-boot setups. The defense is identical everywhere.

## Defender's view

- Add a **pre-boot PIN or passphrase** so a secret that never touches the bus is required; this defeats pure bus sniffing for BitLocker.
- Use **TPM encrypted sessions** (parameter encryption / salted sessions) so the bus traffic is not plaintext.
- Prefer **firmware TPM (fTPM)** integrated into the CPU where appropriate, since there is no external bus to tap, though that shifts the problem to the SoC's own security.
- Treat any discrete secure element on an external bus as sniffable, and encrypt the link.

## Related pages

- [LPC](../protocols/lpc.md)
- [SPI](../protocols/spi.md)
- [Chain of Trust and Secure Boot](chain-of-trust.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
- [Side-Channel Attacks](side-channel.md)
