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

## Step 0 — locate and identify the bus

Find the TPM: a discrete module on a header, or a surface-mount chip. Determine whether it is [LPC](../protocols/lpc.md) or [SPI](../protocols/spi.md) from the part number and routing. A populated TPM header is the easiest tap.

---

## Attack 1 — SPI TPM sniff (common modern case)

**Target:** a board with a discrete SPI TPM in a plaintext (no encrypted-session) config.
**Gear:** a logic analyzer ≥ ~4× the SPI clock (a cheap FX2 does ~24 MSa/s; a faster Saleae/FPGA is safer), fine clips.

Steps:

1. Tap `CLK`, `MOSI`, `MISO`, `CS` like any [SPI sniff](../protocols/spi.md#sniffing-an-spi-bus). Share grounds.
2. Arm on power-on and capture the boot window:
   ```bash
   sigrok-cli -d fx2lafw --config samplerate=24m \
     -C clk,cs,miso,mosi -o tpm.sr --continuous   # stop after the key is released
   ```
3. Decode SPI, then the TPM framing, and locate the `TPM2_Unseal`/response that carries the key. The published `TPM-genie` / `bitlocker-spi-toolkit` parsers find the VMK in a capture automatically.
4. Mount offline (below).

**Success condition:** a 32-byte VMK/FVEK appears in the unseal response.

## Attack 2 — LPC TPM sniff (older boards)

**Target:** a discrete LPC TPM.
**Gear:** the multiplexed 33 MHz LPC bus needs a fast analyzer or an **FPGA sniffer** (ready-made bitstreams exist, e.g. `lpc_sniffer_tpm` for cheap FPGA boards).

Steps:

1. Tap `LCLK`, `LFRAME#`, `LAD[3:0]` (and `LRESET#`). See [LPC](../protocols/lpc.md).
2. Capture at boot with the FPGA sniffer; it reconstructs LPC cycles and dumps TPM traffic.
3. Locate the unseal response and pull the key bytes.

**Success condition:** VMK recovered from the decoded LPC stream.

## Decrypt the volume

```bash
# BitLocker: use the recovered VMK/FVEK
dislocker -V /dev/sdX1 -K recovered.fvek -- /mnt/bitlocker
```

The same plaintext-bus weakness exposes any secret a TPM releases without an encrypted session (other FDE schemes, attestation secrets, measured-boot keys).

## Beyond BitLocker

The same plaintext-bus weakness exposes any secret a TPM releases without an encrypted session: disk keys from other full-disk encryption schemes, attestation secrets, and keys used by measured-boot setups. The defense is identical everywhere.

## Attack 3 — TPM-Fail (timing side-channel, no bus tap)

**Target:** TPMs whose ECDSA/ECSchnorr signing leaks nonce bit-length through timing (several discrete and firmware TPMs were affected).
**Gear:** just the ability to request signatures and time them — **no physical tap**, and it works remotely over the network in some setups.
**How:** collect many signatures with precise timings, recover a few nonce bits per signature from the timing, then solve for the private key with a lattice (hidden number problem). See [side-channel](side-channel.md#named-remote-and-timing-variants).
**Success condition:** reconstructed TPM private signing key.
**Mitigation:** patched/constant-time TPM firmware.

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
