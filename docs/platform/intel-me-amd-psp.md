# Intel ME and AMD PSP

Modern x86 platforms contain a second computer you did not ask for: a small, always-on coprocessor with its own firmware, running beside and beneath the main CPU, with deep access to the system. Intel calls it the **Management Engine (ME)**; AMD calls it the **Platform Security Processor (PSP)**. They are the root of the platform's trust and, because of their privilege and opacity, a long-standing subject of security research. This page explains what they are and what a researcher can realistically do.

## What they are

- **Intel ME**: an embedded coprocessor (historically an ARC core, later x86-based Minute IA) in the chipset, running its own OS. It handles boot verification, power management, firmware TPM, AMT remote management, and more. It runs even when the machine is "off" (in standby).
- **AMD PSP** (now "AMD Secure Processor"): an ARM Cortex-A/TrustZone core inside the SoC, serving the equivalent role: secure boot root, fTPM, key storage, memory encryption setup.

Both are the **real root of trust**: they verify the main firmware before the x86 cores even start. They have access to system memory and run code you cannot normally inspect.

## Why researchers care

- They are the **most privileged code on the machine** (effectively "ring -3"), so a vulnerability there is total.
- They are **opaque and always on**, which is a concern in itself for trust and privacy.
- They enforce the platform's [chain of trust](../attacks/chain-of-trust.md), so understanding them is understanding what the machine actually trusts.

## What you can actually do

Realistically, on your own hardware:

- **Extract the ME/PSP firmware** from the [BIOS dump](uefi-bios.md): the Intel Flash Descriptor marks the ME region, and tools pull it out.

    ```bash
    # locate and extract the ME region from a full flash dump
    me_cleaner -S -O cleaned.bin bios.bin     # also neutralizes ME (see below)
    UEFITool bios.bin                         # browse regions including ME
    ```

- **Analyze the ME firmware** with community tools (`MEAnalyzer`, `unME`, Ghidra with ARC/x86 as appropriate) to learn versions and modules. Full RE is hard; the cores and formats are proprietary.
- **Check ME version against known CVEs**: the ME has had serious published vulnerabilities (for example the SA-00086 class and AMT auth bypasses). `chipsec` and `MEInfo`-type tools report the version so you can match advisories.
- **Reduce the attack surface** with `me_cleaner`, which strips or disables most of the ME firmware while leaving the minimum the platform needs to boot. A mitigation, not an attack.

## The limits

You will not casually read PSP/ME secrets or dump their protected memory. These are hardened, signed, encrypted subsystems; the serious attacks on them have come from dedicated research teams using [glitching](../attacks/fault-injection.md), [side-channel](../attacks/side-channel.md), undocumented debug interfaces (Intel's JTAG-over-USB "DCI"), and deep vulnerability research, not routine bench work. Treat this page as orientation: know what the subsystem is, extract and version-check its firmware, and understand it anchors the trust chain.

## Defender's view

Keep platform firmware updated so ME/PSP vulnerabilities are patched; disable AMT/remote management if unused; consider `me_cleaner` on systems where the ME's full functionality is not needed; and recognize that these subsystems mean "the machine is off" and "the machine is trustworthy" are more complicated than they look.

## Related pages

- [UEFI and BIOS](uefi-bios.md)
- [Chain of Trust and Secure Boot](../attacks/chain-of-trust.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [TPM Sniffing](../attacks/tpm-sniffing.md)
- [Platform Security overview](index.md)
