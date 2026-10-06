# Escaping Restricted Shells

Many devices do not hand you a full shell. Instead the console lands in a vendor menu, a limited CLI, or a restricted shell (`rbash` and friends) that allows only a handful of commands. The goal of this page is to turn that constrained prompt into a real, general-purpose shell. These are standard techniques for assessing a device you are authorized to test.

## Recognize what you are in

First, figure out the restriction:

- A **custom menu** (numbered options, a product-specific CLI). Type `help`, `?`, or an invalid command to see what it accepts and how it reacts.
- A **restricted shell** like `rbash` or a BusyBox shell with a locked `PATH`. Symptoms: you can run some commands but cannot `cd`, use `/`, redirect output, or set variables.
- A **jailed environment** (`chroot`) where the filesystem looks truncated.

Probe gently and watch error messages; they reveal the shell type and what is blocked.

## Breaking out of a restricted shell

Restricted shells block obvious escapes but frequently leave an interpreter reachable. If any of these binaries is allowed, they spawn an unrestricted shell:

```bash
# via common interpreters, if present
python -c 'import os; os.system("/bin/sh")'
perl -e 'exec "/bin/sh";'
awk 'BEGIN{system("/bin/sh")}'
lua -e 'os.execute("/bin/sh")'

# via editors with shell escapes
vi            # then :!/bin/sh   or  :set shell=/bin/sh then :shell
# less / man  then  !/bin/sh

# via find's exec
find . -exec /bin/sh \; -quit
```

The principle is the same throughout: any program that can execute another program is a potential door out. BusyBox systems often ship `vi` and `awk`, which are two of the most reliable escapes on embedded Linux.

### PATH and builtins

If the restriction is a limited `PATH`, you may be able to call binaries by absolute path, or reset the environment:

```bash
/bin/sh
export PATH=/bin:/sbin:/usr/bin:/usr/sbin
```

`rbash` blocks absolute paths and `PATH` changes, but a nested unrestricted `sh` invoked through one of the escapes above is not restricted.

## Attacking a custom menu CLI

Vendor menus are often thin wrappers over shell commands, and the wrapping is frequently exploitable:

- **Command injection.** If a menu option takes input (a filename, an IP, a ping target) and passes it to a shell, inject with `;`, `|`, `&&`, `$(...)`, or backticks:

    ```
    ping 127.0.0.1; /bin/sh
    ping 127.0.0.1 | sh
    ```

    The ping-target field in a router CLI is a time-honored example on devices people own and test.

- **Hidden commands.** Menus often retain debug or factory commands not shown in `help`. The [firmware](firmware-analysis.md) reveals them: search the CLI binary's strings for command names.
- **Argument abuse.** An option that runs a known tool may accept extra flags that do more than intended (for example a "show config" that accepts a path and will read any file).

Reversing the menu binary in [Ghidra](../reverse-engineering/ghidra-setup.md) is the systematic way to find these: locate where it calls `system`/`popen` and trace what reaches those calls.

## Escaping a chroot jail

If you have a shell but it is `chroot`-ed, and you have root inside the jail, classic chroot escapes apply (creating a new chroot and walking up with an open directory file descriptor). Root inside a weak jail is usually escapable; an unprivileged jail is harder and may not be worth it when other routes (bootloader, flash dump) exist.

## When software escapes fail, change the filesystem

If you cannot escape from inside, change the environment from outside:

- [Interrupt U-Boot](bootloader-access.md) and boot with `init=/bin/sh`, bypassing the menu entirely.
- [Dump the flash](spi-flash-dump.md), edit the startup script or the menu config to launch a real shell, and write it back (on a device you own).

These sidestep the restriction rather than fighting it, and are often faster than hunting for an injection.

## Defender's view

A restricted interface is only as strong as its weakest command. Never pass menu input to a shell without strict validation, remove interpreters and editors from production images, do not rely on `rbash` as a security boundary, and lock the bootloader so the menu cannot be bypassed with `init=`.
