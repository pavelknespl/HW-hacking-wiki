# Transient Execution Attacks

Modern CPUs run instructions speculatively and out of order, rolling back the architectural result when a guess was wrong — but the **microarchitectural** side effects (cache state, buffer contents) are not fully undone. Transient-execution attacks coax the CPU into speculatively touching secret data, then read it back through a side channel. These are microarchitectural rather than bench-hardware attacks, but they belong here because they are hardware defects you exploit, not software bugs you patch away.

!!! note "Scope"
    This is orientation and defensive context. These attacks are software-delivered and CPU-generation-specific; the point for a hardware assessor is to know the class, check the CPU against it, and apply microcode/OS mitigations.

## The common shape

1. **Set up a covert channel**, almost always a cache side channel (Flush+Reload): flush a probe array, let transient code touch `array[secret*4096]`, then time reloads — the fast line reveals `secret`.
2. **Get the CPU to transiently access a secret** it would never architecturally expose.
3. **Leak** the secret through the channel before the rollback hides the architectural trace.

The channel is the same [timing/cache side-channel](side-channel.md) math; the novelty is step 2.

## The families

- **Spectre (bounds-check / branch prediction).** Mistrain a branch predictor so the CPU speculatively runs past a bounds check (v1) or to an attacker-chosen gadget (v2, branch target injection), reading memory in the victim's address space. Hits almost every speculating CPU; mitigated with `lfence`/retpoline/IBRS and compiler barriers, not fully removed.
- **Meltdown (rogue data cache load).** On affected Intel/ARM cores, a user-mode transient load of a kernel address returns the data before the permission fault retires, leaking kernel memory from userspace. Mitigated by **KPTI** (unmapping the kernel from user page tables).
- **MDS / Fallout / RIDL / ZombieLoad.** Leak stale data from internal **microarchitectural buffers** (line-fill, store, load ports) across threads/domains. Mitigated by buffer-flush microcode (`VERW`) and disabling hyper-threading for strict isolation.
- **Foreshadow / L1TF.** Leak from the L1 cache / SGX enclaves via terminal page faults.
- **PACMAN (Apple M1).** Speculatively **brute-force ARM Pointer Authentication Codes**: a mispredicted branch lets you test a guessed PAC without the crash a wrong guess normally causes, observing success through a side channel — removing PAC as a mitigation for a separate memory bug. Microarchitectural and unpatchable in silicon; defense is fixing the underlying memory-safety bug PACMAN unmasks.

## Checking a machine

```bash
# Linux reports per-vulnerability status and whether a mitigation is active
grep . /sys/devices/system/cpu/vulnerabilities/*
lscpu | sed -n '/Vulnerabilit/,$p'       # same data, summarized
```

A line reading `Vulnerable` (not `Mitigation: ...`) is a real finding on a machine that handles secrets across trust boundaries.

## Defender's view

Keep **CPU microcode and OS mitigations current** (they ship together and carry a performance cost that is the price of isolation); **disable SMT/hyper-threading** where cross-thread leakage (MDS/L1TF) is in the threat model; compile sensitive code with Spectre barriers (retpoline, `speculation_barrier`); do not co-locate mutually distrusting workloads on the same physical core; and for PACMAN-class results remember PAC and similar mitigations are hardening, not a substitute for memory safety. Track the exact CPU generation — the fixes are in silicon over time.

## Related pages

- [Side-Channel Attacks](side-channel.md)
- [Rowhammer](rowhammer.md)
- [CPU Architectures](../fundamentals/architectures.md)
- [Secure Elements](secure-elements.md)
