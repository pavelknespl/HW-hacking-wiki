# Fault Injection and Glitching

Fault injection deliberately pushes a chip outside its safe operating conditions for a brief moment, so that a single instruction misbehaves. Done at the right instant, that one glitch can skip a security check, corrupt a comparison, or drop the device into a state it should never reach. It is how attackers defeat [read-out protection](../protocols/swd.md#read-out-protection-again) and [secure boot](bootloader-bypass.md) when no software flaw exists. This page explains the physics, the methods, concrete parameters, and a worked campaign on hardware you own.

## The core idea

A processor assumes a stable supply voltage and a clean clock. Violate that assumption for a few nanoseconds and the CPU can:

- **Skip an instruction**, for example the one that acts on a failed signature check.
- **Corrupt a value** being loaded or compared, so a "not equal" becomes "equal."
- **Mis-fetch an instruction**, executing something unintended.

The attacker's whole job is to land that disturbance at the precise moment a security-critical instruction executes. Everything else is tooling to achieve **timing** (a trigger plus an offset) and **repeatability** (controlled glitch shape, many attempts).

## The methods

=== "Voltage glitching"

    Briefly drop (or spike) the core supply voltage. The usual technique is a **crowbar**: a fast transistor momentarily shorts the power rail to ground for a controlled number of nanoseconds. Cheap, effective, the most common starting point.

    - Target the **core** rail (VCORE, often 0.9 to 1.2 V on an SoC, 1.2 V internal on many STM32), not the I/O rail, for logic faults.
    - Needs a fast switch (a MOSFET driven by an FPGA or a [ChipWhisperer](../software/chipwhisperer.md)) and control over glitch **width** (how long the short lasts) and **offset** (delay from a trigger).
    - Typical crowbar pulse width lands in the tens to a few hundred nanoseconds; you sweep it.

=== "Clock glitching"

    Insert a too-short or extra clock pulse so the CPU latches a value before it is stable, causing a faulted instruction. Only works on chips driven by an **external** clock you can manipulate (many modern parts use an internal oscillator, which rules this out).

    - Precise and repeatable where applicable, measured directly in clock cycles.
    - Requires access to the clock line and a glitch generator that can inject a malformed edge (the ChipWhisperer does this in hardware when run synchronous to the target clock).

=== "Electromagnetic (EMFI)"

    A fast, localized magnetic pulse from a small coil induces currents inside the die, faulting nearby logic. Works without electrical contact, aims at a region of the chip, and beats internal clocks and some voltage-glitch countermeasures. See the dedicated [EMFI](emfi.md) page, and [PicoEMP](#cheap-emfi-picoemp) below for a sub-100-dollar entry.

=== "Optical / laser"

    Decapsulate the chip and fault specific transistors with a laser. The most precise and most powerful method, and the most demanding: it needs [decapping](decapping.md), a laser station, and die-level targeting. Reserved for well-resourced labs.

## Software-triggered fault injection (no bench rig)

A whole class of published attacks induces faults **from software** by abusing the chip's own power/clock management, needing no physical access at all. The CPU exposes voltage and frequency controls for power saving; pushed out of spec, they fault the silicon.

- **Plundervolt / V0ltpwn (CVE-2019-11157).** On Intel Core CPUs, the undocumented voltage MSR `0x150` lets ring-0 software undervolt the core. A precise undervolt during an SGX enclave's AES or multiplication produces bit flips, corrupting results enough to break RSA/AES inside the enclave and even induce memory-safety faults in otherwise-correct enclave code. Mitigated by microcode that locks the MSR and by SGX attestation reflecting the voltage setting.
- **CLKSCREW.** On an ARM/Android phone, the DVFS driver's frequency/voltage regulators were pushed past their safe envelope from software to fault TrustZone — extracting an AES key from, and loading a self-signed app into, the TEE. No hardware at all.
- **VoltPillager.** The hardware cousin: a ~$30 microcontroller injects **SVID** commands on the board's serial voltage-ID bus between CPU and voltage regulator, undervolting from outside the CPU and defeating Plundervolt's software mitigations. Same primitive as a PMBus attack — see [SMBus and PMBus](../protocols/smbus-pmbus.md).

The attack loop is the same as [voltage glitching](#the-workflow) (sweep depth/timing of the undervolt around the target operation, detect a faulted result), but the "glitcher" is a register write or a bus command rather than a crowbar.

## The equipment

- **ChipWhisperer** (Nano, Lite, Husky) is the standard integrated platform: it generates glitches, provides the trigger and cycle-accurate timing, and resets the target. See the [ChipWhisperer page](../software/chipwhisperer.md) for the models and Python API. The best place to learn.
- A **custom crowbar glitcher** (a power MOSFET plus an FPGA or fast MCU) is a cheap DIY entry for voltage glitching. See [building a crowbar](#building-a-crowbar-glitcher).
- **[PicoEMP](#cheap-emfi-picoemp)** is an open-source, low-cost EMFI tool for electromagnetic glitching without a ChipSHOUTER budget.
- An **oscilloscope** to see the core rail and confirm the glitch lands where and when you think.
- A **bench supply** whose current readout doubles as a crude fault/crash indicator.

## Triggering

The trigger is the single most important and most underrated part of a glitching campaign. It is the time reference the glitch **offset** is measured from, and its quality sets how wide you must sweep. A rock-solid trigger a few microseconds before the target instruction turns a hopeless search into a short one.

### Trigger types, best to worst

- **Hard I/O trigger (best).** A digital line that changes at a known point in the boot flow: reset deassertion, a GPIO the bootloader toggles, a LED turning on, a chip-select or clock starting. Wire it to the glitcher's trigger input and glitch a fixed offset later. Lowest jitter.
- **Communication trigger.** Trigger on a byte on a bus you can see: the target prints a known character over [UART](../protocols/uart.md), or a specific [SPI](../protocols/spi.md) command appears. You trigger on the line edge, or on a pattern.
- **Power / analog trigger.** When no digital line helps, trigger on the power signature of the operation itself. The ChipWhisperer-Husky/Pro supports **SAD (sum-of-absolute-differences) pattern matching**: you record the power trace around the target op once, then the hardware fires when a live trace matches that template. Powerful on devices with no usable digital reference.
- **Edge-count trigger.** Count N edges on a line (clock ticks, UART bytes, CS assertions) after reset, then glitch. Useful when the interesting op is "the Nth thing that happens."
- **Reset-relative, free-running (worst but always available).** Release reset, wait a fixed delay, glitch, and sweep the delay across a wide window. Highest jitter because boot timing varies, but needs no insight into the target.

### Jitter is the enemy

Boot timing varies run to run (internal oscillator drift, variable-length init, cache/flash wait states, interrupts). That spread is **jitter**, and it forces a wider offset sweep, which multiplies campaign time. Reduce it by:

- Triggering **as close as possible** to the target instruction. Jitter accumulates with distance from the trigger, so a trigger 5 us before the compare beats one 50 ms before it.
- Preferring a hard I/O or SAD trigger over reset-relative timing.
- Disabling anything nondeterministic you control (interrupts, variable-length UART prompts) during the search.
- Running the glitcher **synchronous** to the target clock when possible (ChipWhisperer "clkgen" mode), so offset is counted in exact target cycles rather than wall-clock nanoseconds.

### Trigger config on a ChipWhisperer

```python
# external I/O trigger: glitch fires on the rising edge of a target GPIO
scope.trigger.triggers = "tio4"      # the line carrying your trigger
scope.trigger.module   = "basic"
scope.adc.basic_mode   = "rising_edge"
# SAD trigger (Husky/Pro): match a recorded power pattern instead of a pin
# scope.trigger.module = "SAD"; scope.SAD.reference = ref_trace; scope.SAD.threshold = 50
```

## Parameters and starting ranges

Glitching is a parameter search, so start from sane ranges and narrow in. These are starting points, not guarantees; every die is different.

| Parameter | What it is | Sensible start |
| --- | --- | --- |
| **Offset** | Delay from trigger to glitch | Whole window of the target op; narrow once you see crashes |
| **Width** | Duration of the voltage short / clock perturbation | Voltage crowbar: 20 to 400 ns, step ~10 ns. Clock: 1 to a few cycles |
| **Repeat** (CW) | Glitch length in target clock cycles | 1 to 10 |
| **Glitch voltage** | Depth of the dip (if adjustable) | Short to ground; for adjustable supplies, dip VCORE 30 to 70 percent |
| **Attempts per point** | Retries at each (offset, width) | 20 to 100; successes can be well under 1 percent |
| **EMFI pulse power** | Coil drive energy | Start low, raise until faults appear before resets dominate |
| **EMFI position** | XY coordinate over the die | Scan a grid; logic and the target block are hotspots |

!!! tip "Read the crash map"
    Classify every attempt as **normal**, **crash/reset**, or **success**, and plot it against the swept parameters. The border between the all-normal region and the all-crash region is where real faults cluster. If you see only normal, glitch harder (wider/stronger); if you see only crashes, back off. A clean normal/crash boundary with no successes usually means the offset is wrong, not the width.

## The workflow

Glitching is a search for the right parameters, so structure matters more than luck.

1. **Pick the target moment.** Identify the instruction or operation to disturb: the signature compare, the [read-protection](../mcu/stm32-rdp.md) check, a password comparison. [Firmware analysis](firmware-analysis.md) tells you where it sits in the boot flow.
2. **Establish a trigger.** Get the most reliable reference you can (see [Triggering](#triggering)). The glitch offset is measured from here, and trigger quality dominates your success rate.
3. **Instrument the outcome.** Define how you know a glitch succeeded: the device prints something it should not, a locked read returns data, a check passes. Automate detecting it, and automate target reset so the loop runs unattended.
4. **Sweep the parameters.** Scan offset and width across ranges, many attempts per point, logging every result. Build the crash map.
5. **Classify and narrow.** Mark each attempt normal / crash / success, zoom the sweep into the boundary region, and increase attempts there.

```
for offset in range(offset_min, offset_max):
    for width in range(width_min, width_max):
        for attempt in range(N):
            arm_glitch(offset, width)
            reset_target()
            result = observe()        # normal | crash | SUCCESS
            log(offset, width, attempt, result)
```

That loop is the essence of every glitching campaign, whatever the tooling.

## Worked example: voltage-glitching a boot check

A concrete campaign against a target that refuses to continue unless a byte in flash (or a signature result) is "valid," run on a [ChipWhisperer-Lite or Husky](../software/chipwhisperer.md) driving a crowbar on the target's VCORE.

**Setup.** Remove or reduce the VCORE decoupling capacitor so the crowbar can actually pull the rail down fast (big bulk caps fight the glitch). Wire the ChipWhisperer `VCCGLITCH` SMA to VCORE through the low-power crowbar MOSFET, share ground, and bring the target's "I am about to check" GPIO to `tio4` as the trigger.

```python
import time
import chipwhisperer as cw

scope  = cw.scope()
target = cw.target(scope)
scope.default_setup()

# --- glitch output: voltage crowbar via the low-power MOSFET ---
scope.glitch.clk_src     = "clkgen"      # synchronous to the target clock
scope.glitch.output      = "glitch_only"
scope.glitch.trigger_src = "ext_single"  # one glitch per external trigger
scope.io.glitch_lp       = True          # enable low-power crowbar MOSFET
scope.io.glitch_hp       = False

# --- trigger: rising edge on the target's pre-check GPIO ---
scope.trigger.triggers = "tio4"
scope.adc.basic_mode   = "rising_edge"

def reboot():
    scope.io.target_pwr = False; time.sleep(0.05)
    scope.io.target_pwr = True;  time.sleep(0.05)

for ext_offset in range(0, 3000, 1):      # cycles after trigger; narrow after first hits
    for repeat in range(1, 8):            # glitch width in target cycles
        for _ in range(30):               # attempts per point
            scope.glitch.ext_offset = ext_offset
            scope.glitch.repeat     = repeat
            reboot()
            scope.arm()
            # ... let the target run to the check, capture its UART output ...
            resp = target.read()          # your I/O here
            if b"ACCESS GRANTED" in resp or looks_unlocked(resp):
                print("SUCCESS", ext_offset, repeat); 
            elif resp == b"":
                log_crash(ext_offset, repeat)
            else:
                log_normal(ext_offset, repeat)
```

The two inner knobs are `ext_offset` (where) and `repeat` (how hard). Find the crash band first by widening `repeat`, then walk `ext_offset` through the crash band hunting the narrow offset where the fault corrupts the compare instead of killing the core. For a real STM32 read-protection downgrade, see [STM32 RDP](../mcu/stm32-rdp.md); the same loop applies with the check being the RDP-level read.

## Building a crowbar glitcher

The cheapest real voltage glitcher is a fast switch across the core rail:

- **Switch.** A logic-level N-channel power MOSFET (low Rds(on), low gate charge, for example an IRLML2502 / AO3400 class part) with drain on VCORE and source on ground. When the gate is pulsed high, it briefly shorts the rail.
- **Driver.** The gate is driven by a fast digital source with nanosecond control: an FPGA, a [ChipWhisperer](../software/chipwhisperer.md) glitch output, or even a carefully clocked fast MCU (RP2040 PIO is a popular cheap option). You need control of the pulse **width** to ~10 ns resolution and a defined **offset** from the trigger.
- **Prep the target.** Remove bulk and large decoupling caps on VCORE so the rail can actually move; leave only what the chip needs to not brown out during normal run. This is the step that most often decides whether a crowbar works at all.
- **Observe.** Put a scope probe on VCORE to confirm the dip depth and width match what you commanded, and to correlate crashes with glitch shape.

A crowbar cannot set an arbitrary voltage; it shorts to ground for a time. Depth comes from width and from how much capacitance you left on the rail. For adjustable-depth glitching you instead switch between two supply levels (a "voltage mux"), which is more work.

## Cheap EMFI: PicoEMP

[EMFI](emfi.md) used to mean a ChipSHOUTER and a four-figure budget. **PicoEMP** is an open-source, RP2040-based EM pulse generator (NewAE / Colin O'Flynn design) that brings it under ~100 dollars in parts:

- It charges a capacitor to a few hundred volts and dumps it through an injection **tip** (a small hand-wound coil) over the target package, inducing a fault with no electrical contact and no decap.
- It is deliberately **not** HV-isolated for lab-grade safety the way commercial gear is, so treat the high-voltage section with respect: it can bite, and it can destroy the target. Discharge before handling.
- Drive and trigger it like any other glitcher: feed it the same trigger you would a crowbar, and sweep **pulse timing**, **position** (move the tip over the die on a manual or motorized XY stage), and **pulse energy**.
- It pairs naturally with a ChipWhisperer for triggering and campaign control, and reaches targets with internal clocks or supply filtering that defeat a crowbar.

See [EMFI](emfi.md) for the method in depth and [decapping](decapping.md) for when you need to get the coil closer to the die.

## What it is used for

- **Defeating [read-out protection](../protocols/swd.md)** to dump internal MCU flash the debug port refuses to read (see [STM32 RDP](../mcu/stm32-rdp.md), [nRF52 APPROTECT](../mcu/nrf52-approtect.md)).
- **Bypassing [secure boot](bootloader-bypass.md)** by faulting the signature comparison, or faulting the [boot ROM](../platform/bootrom-exploitation.md) before it verifies the next stage.
- **Skipping a password or PIN check** in a bare-metal device.
- **Reaching a hidden state**, such as a serial loader or debug path normally gated by a check, or faulting the read of a lock [eFuse](../fundamentals/efuse-otp.md).

## Realistic expectations

Glitching is probabilistic, slow, and device-specific. You will spend most of the time finding a reliable trigger and a parameter window, and many targets include countermeasures: supply filtering, voltage and clock sensors, redundant checks, and randomized timing. It is the right tool only when cheaper software and memory routes are exhausted, which is why it sits near the bottom of the [methodology](../getting-started/methodology.md).

## Defender's view

Make the critical check fault-resistant: perform it twice with different code, verify the result in more than one place, require multiple independent faults to pass, and add random delays so the attacker cannot find a stable trigger (jitter is your friend here, their enemy). Use a chip with voltage and clock glitch sensors where the threat warrants it, keep decoupling that resists crowbar dips, and detect brownouts. A single comparison that decides everything is exactly what fault injection targets.

## Related pages

- [Electromagnetic Fault Injection (EMFI)](emfi.md)
- [ChipWhisperer](../software/chipwhisperer.md)
- [Side-Channel Attacks](side-channel.md)
- [Bootloader Bypass](bootloader-bypass.md)
- [STM32 RDP](../mcu/stm32-rdp.md)
- [nRF52 APPROTECT](../mcu/nrf52-approtect.md)
- [eFuse and OTP Memory](../fundamentals/efuse-otp.md)
- [SWD](../protocols/swd.md)
- [Automating Attacks](../scripting/automation.md)
