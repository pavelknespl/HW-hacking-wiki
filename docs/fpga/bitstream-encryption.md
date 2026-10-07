# Bitstream Encryption and Attacks

To stop cloning and reverse engineering, FPGA vendors encrypt the bitstream. The FPGA holds a key, the config flash holds ciphertext, and the fabric decrypts on load. This raises the bar, but the scheme has a long history of being broken by [side-channel](../attacks/side-channel.md) analysis, protocol flaws, and [fault injection](../attacks/fault-injection.md). This page explains how the protection works and where it has failed. For authorized research.

## How it works

A modern encrypted-bitstream flow looks like this:

1. The design is encrypted with **AES** (AES-256-CBC on Xilinx 7-series, GCM on UltraScale) at build time.
2. The key is programmed into the FPGA's own non-volatile store: **BBRAM** (battery-backed RAM, erasable) or **eFUSE** (one-time, permanent). See [eFuse and OTP memory](../fundamentals/efuse-otp.md).
3. At boot the FPGA reads the ciphertext from [config flash](bitstream-extraction.md), decrypts internally, and configures. Optionally it checks an **HMAC or RSA signature** so a modified bitstream is rejected (authentication, separate from confidentiality).
4. Security bits disable [JTAG readback](bitstream-extraction.md) and force encrypted-only loading so plaintext bitstreams are refused.

Confidentiality (AES) and authentication (HMAC/RSA) are separate properties. Several attacks work precisely because an older device encrypts but does not properly authenticate, or authenticates in a way that can be sidestepped.

## Attack: Starbleed (Xilinx 7-series, 2020)

The headline break. Xilinx 7-series (and Virtex-6) encrypt with AES-256-CBC but let the attacker abuse the configuration logic as a decryption oracle:

- The bitstream contains configuration-register writes. One register, **WBSTAR** (warm boot start address), survives a reset and can be read back over [JTAG](../protocols/jtag.md).
- By crafting a malicious encrypted command stream, an attacker gets the FPGA to decrypt a block and write the plaintext into WBSTAR, then resets and reads WBSTAR out. CBC malleability lets this be done block by block.
- Result: full plaintext recovery of the bitstream, 32 bits at a time, using only the device itself. No key extraction needed, because the device decrypts for you.

The lesson is classic: encryption without authentication of the command stream turns the decryptor into an oracle. UltraScale (GCM, authenticated) is not vulnerable in the same way.

## Attack: side-channel key recovery

Older families (and the AES engine itself) have fallen to [power and EM side-channel analysis](../attacks/side-channel.md):

- Capturing power or EM traces during the decryption at boot, then doing **differential/correlation power analysis (DPA/CPA)** to recover the AES key. This has been demonstrated against Xilinx Virtex, Altera Stratix, and others by research groups.
- Once the key is recovered, every bitstream encrypted under it is readable, and clones can be produced.
- A [ChipWhisperer](../software/chipwhisperer.md) is the standard tool for this class of work.

## Attack: fault injection

[Voltage, clock, or EM glitching](../attacks/fault-injection.md) during boot can skip the security check, corrupt the authentication comparison, or force the device out of encrypted-only mode. [EMFI](../attacks/emfi.md) is attractive here because it needs no decap and can be positioned over the config logic.

## Attack: key storage

The key itself lives in [BBRAM or eFUSE](../fundamentals/efuse-otp.md), so invasive and semi-invasive methods against that store apply: [decapping](../attacks/decapping.md) and optical/SEM reading of fuses, or passive voltage contrast. This is the hard, expensive path and depends on the exact part.

## Vendor and family notes

- **Xilinx/AMD 7-series**: AES-256-CBC, vulnerable to [Starbleed](bitstream-encryption.md). UltraScale/UltraScale+ moved to AES-GCM with authentication and RSA boot.
- **Intel/Altera**: AES bitstream encryption across Stratix/Arria/Cyclone; various side-channel results in the literature.
- **Lattice ECP5**: AES encryption available; iCE40 generally has no bitstream encryption, so iCE40 designs are usually recoverable straight from [flash](bitstream-extraction.md).
- **Microchip/Microsemi (PolarFire, SmartFusion)**: flash-based with stronger key management and anti-tamper, a harder target.

## Defender's view

Use an authenticated mode (GCM, or AES plus HMAC/RSA) so the command stream cannot be abused as an oracle, prefer families without known [oracle attacks](bitstream-encryption.md), store keys in [eFUSE rather than BBRAM](../fundamentals/efuse-otp.md) where permanence is acceptable, disable [readback](bitstream-extraction.md) and force encrypted-only loading, and add [side-channel countermeasures](../attacks/side-channel.md#defenders-view) if the AES engine is exposed. Treat the config flash and the ciphertext as public.

## Related pages

- [FPGA Security overview](index.md)
- [Bitstream Extraction](bitstream-extraction.md)
- [Side-Channel Attacks](../attacks/side-channel.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [EMFI](../attacks/emfi.md)
- [eFuse and OTP Memory](../fundamentals/efuse-otp.md)
- [ChipWhisperer](../software/chipwhisperer.md)
