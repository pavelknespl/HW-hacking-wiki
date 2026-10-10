# DMA Attacks

Some ports and buses can read and write system memory **directly**, without the CPU or OS mediating. Direct Memory Access is a performance feature (a device moving data without interrupting the processor), but a malicious device on a DMA-capable port can use it to read secrets out of RAM or patch the running OS. This is the fastest way to pull keys and defeat lock screens on a machine you have physical access to, for authorized testing of your own systems.

## Why DMA is dangerous

DMA-capable interfaces let an attached device access physical memory directly:

- **Read** arbitrary RAM: disk-encryption keys, passwords, the lock-screen state.
- **Write** arbitrary RAM: patch the kernel to disable authentication, unlock the screen, or inject code.

Because it bypasses the CPU, the OS's own protections do not see it unless the hardware **IOMMU** is enforcing boundaries (the key defense, below).

## The DMA-capable ports

| Interface | Notes |
| --- | --- |
| **Thunderbolt** (and USB4) | PCIe tunneled over the port, so a Thunderbolt device is a PCIe device with DMA. The classic modern vector. |
| **PCIe** (incl. ExpressCard, M.2, oculink) | Direct PCIe slots and adapters. |
| **FireWire (IEEE 1394)** | The historic DMA attack port; largely gone but iconic. |
| **CardBus / PCMCIA** | Legacy. |

A plugged-in malicious PCIe/Thunderbolt device, or an FPGA emulating one, is the attack platform.

## PCILeech and screamer hardware

The standard toolkit is **PCILeech** with an FPGA board (for example a "PCIe Screamer" or compatible). The FPGA presents as a PCIe device and PCILeech drives it to read and write host memory:

```bash
# dump host physical memory over the FPGA DMA device
pcileech dump -device fpga -out mem.raw

# search memory / patch a target (e.g. unlock, with the right signature)
pcileech kmdload -device fpga          # load a kernel module into the victim
pcileech wx64_unlock ...               # known memory patches for lock screens
```

PCILeech ships known techniques to dump memory, inject a kernel module, and patch common OS authentication checks, turning raw DMA into a working lock-screen bypass or key dump.

## Attacks it enables

Each maps to a PCILeech workflow against a machine whose IOMMU is off/misconfigured or in the pre-boot window:

- **Attack 1 — key extraction from RAM.** `pcileech dump` the physical memory, then run `aeskeyfind`/`bitlocker2john` on it (same prize as a [cold boot attack](cold-boot.md), but live and without power games). Success: a disk key recovered from the dump.
- **Attack 2 — lock-screen bypass.** Use PCILeech's known memory patches (`wx64_unlock` and friends) to patch the OS authentication check in kernel memory. Success: screen unlocks without credentials.
- **Attack 3 — kernel implant / persistence.** `pcileech kmdload` injects a kernel module into the victim, giving code execution and persistence. Success: injected module runs.

All three require the IOMMU to be absent, disabled, or not yet initialized.

## The defense: IOMMU

The hardware **IOMMU** (Intel VT-d, AMD-Vi) restricts what physical memory a device may access, confining DMA to buffers the OS grants. When enabled and used:

- **Kernel DMA Protection** (modern Windows/Linux/macOS) uses the IOMMU to sandbox Thunderbolt/PCIe devices, blocking arbitrary memory access.
- Older systems, or ones with the IOMMU off or misconfigured, remain wide open, which is why DMA attacks still land on plenty of hardware.

Pre-boot and early-boot windows (before the OS sets up IOMMU protections) are a recurring soft spot even on protected systems.

## Defender's view

- Enable the **IOMMU / Kernel DMA Protection**; it is the real mitigation.
- Disable or restrict **Thunderbolt** (require user authorization for devices, or block PCIe tunneling) on sensitive machines.
- Combine with full-disk encryption that **evicts keys on lock/suspend** so a DMA read finds no key (same logic as the [cold boot](cold-boot.md) defense).
- Treat any exposed PCIe/Thunderbolt port on an unattended powered machine as a memory read/write hole.

## Related pages

- [Cold Boot Attacks](cold-boot.md)
- [TPM Sniffing](tpm-sniffing.md)
- [Chain of Trust and Secure Boot](chain-of-trust.md)
- [Side-Channel Attacks](side-channel.md)
