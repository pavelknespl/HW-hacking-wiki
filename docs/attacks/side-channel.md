# Side-Channel Attacks

A side-channel attack recovers secrets not by reading them directly but by measuring something the device leaks while it uses them: the power it draws, the time it takes, the electromagnetic field it emits. A chip computing with a secret key produces a physical signature that correlates with that key, and with enough measurements and the right analysis, the key falls out. This page covers the techniques, concrete capture setup and trace counts, and a worked CPA campaign, for cryptographic analysis of hardware you are authorized to test.

## Why it works

Digital logic is physical. Switching a bit from 0 to 1 draws a slightly different amount of current than leaving it at 0, and that difference depends on the data being processed. When that data is a secret key mixed with known input, the power trace carries a faint, exploitable correlation with the key. The attack is statistical: individual traces are noisy, but averaging over many measurements extracts the signal.

## The channels

| Channel | What you measure | Typical gear |
| --- | --- | --- |
| **Power** | Current the chip draws over time | Shunt resistor + oscilloscope, or a [ChipWhisperer](../software/chipwhisperer.md) |
| **Electromagnetic (EM)** | Near-field emissions from the die | H-field probe + low-noise amplifier + scope |
| **Timing** | How long an operation takes | Precise timing of responses |

Power and EM are closely related: EM lets you localize to a region of the chip and avoids inserting a shunt, at the cost of a probe, an amplifier, and an XY scan to find the hotspot (the same positioning problem as [EMFI](emfi.md)). Correlation analysis on EM traces is often called **CEMA**, and it is the same math as power CPA.

## Timing attacks: the simplest case

Some code leaks through *how long* it runs. The classic example is a **non-constant-time comparison**, such as a password or MAC check that returns as soon as it finds the first wrong byte:

```c
for (i = 0; i < len; i++)
    if (input[i] != secret[i]) return WRONG;   // returns early
return OK;
```

Each additional correct leading byte makes the check take marginally longer. By measuring response time while varying one byte at a time, an attacker recovers the secret byte by byte, turning an exponential search into a linear one. Over a network the per-byte delta is nanoseconds buried in jitter, so you time **hundreds to thousands** of requests per guess and compare medians or low percentiles; locally on a bus it is far easier. The defense is a **constant-time** comparison that always examines every byte. Timing attacks need no special hardware, only precise measurement, so they are the first side channel to consider.

## Simple Power Analysis (SPA)

SPA reads secrets from the *shape* of a single or a few power traces. Different operations draw visibly different power, so you can sometimes see the structure of a computation directly:

- A square-and-multiply modular exponentiation (RSA) can reveal key bits because a "multiply" step looks different from a "square" step, and multiplies happen only on 1 bits.
- The rounds of a block cipher are often countable in a trace, which helps you align measurements.
- RSA/ECC implementations that branch on key bits are the prime SPA target; the fix is constant-time, branchless scalar/exponent handling (Montgomery ladder, always-multiply).

SPA works when the leak is large and visible. When it is not, you need statistics.

## Differential and Correlation Power Analysis (DPA / CPA)

The powerful, general technique. Instead of reading one trace, you collect **many** traces with known, varying inputs and correlate them against a hypothesis about the key.

The idea, for a byte of an AES key, attacking the first-round S-box output:

1. Capture power traces while the device encrypts many known plaintexts.
2. For each of the 256 possible values of one key byte, predict an intermediate value's power for every plaintext. The standard target is `Sbox(plaintext_byte XOR key_byte)`, and the standard predictor is its **Hamming weight**.
3. Correlate each hypothesis (Pearson correlation) against the measured traces at each time sample.
4. The correct key-byte guess produces a sharp correlation peak at the sample where that intermediate is computed; wrong guesses stay in the noise.
5. Repeat for each of the 16 key bytes to recover the full key.

CPA (correlation) is the modern refinement of DPA and is what most tools implement. The beauty is that it attacks one key byte at a time, so 16 bytes of AES cost 16 small 256-way searches, not one astronomically large one.

### Leakage models

The predictor you correlate against must match how the target leaks:

- **Hamming weight (HW)** of an intermediate: good for software on a microcontroller, where a value is loaded onto a bus/register and the bus pre-charges or the register is written from zero-ish state.
- **Hamming distance (HD)** between two consecutive register states: the right model for **hardware** AES, where a register is overwritten and the power tracks the number of flipped bits, HD(old, new).
- Pick the intermediate that is **nonlinear in the key** (an S-box output), never a linear XOR, so a wrong key guess is maximally de-correlated.

## Capture setup

Good traces beat clever analysis. Most campaigns are won or lost here.

- **Power, shunt method.** Put a small resistor (commonly 1 to 50 ohm) in series with the chip's core supply and measure the voltage across it with a scope. Low-side (in the ground return) is easiest to probe safely. AC-couple or use a differential probe, and give yourself bandwidth well above the clock. Remove large decoupling caps on that rail (as in [crowbar prep](fault-injection.md#building-a-crowbar-glitcher)) so the signal is not smoothed away.
- **Power, ChipWhisperer.** The [ChipWhisperer](../software/chipwhisperer.md) integrates the measurement, the amplifier, the trigger, and (critically) **synchronous sampling**: its ADC is locked to the target clock, so every trace samples the same point in each cycle and traces are aligned for free. This is why a CW recovers an unprotected AES key in a few dozen traces where a free-running scope needs far more.
- **EM.** A small H-field probe (commercial or hand-wound coil) over the die, into a low-noise amplifier (for example ~30 dB), into the scope. Scan position on an XY stage to find where the crypto engine leaks most, then capture there.
- **Trigger.** Same discipline as [fault injection triggering](fault-injection.md#triggering): get a tight, repeatable reference at the start of the crypto op (a GPIO the target raises, or a pattern/SAD trigger). Trigger jitter directly costs you trace alignment later.

### ChipWhisperer capture config

```python
import chipwhisperer as cw
scope  = cw.scope()
target = cw.target(scope)
scope.default_setup()

scope.clock.adc_src   = "clkgen_x4"   # synchronous: sample at 4x the target clock
scope.gain.db         = 25            # raise until the trace fills the ADC without clipping
scope.adc.samples     = 5000          # enough to cover the first AES round
scope.adc.offset      = 0             # move the window to the op if needed
scope.trigger.triggers = "tio4"       # hard trigger at crypto start
```

## How many traces

Order-of-magnitude expectations for a first-round CPA, assuming a decent trigger and alignment:

| Target | Traces (rough) |
| --- | --- |
| Unprotected software AES on an MCU, ChipWhisperer | tens to a few hundred |
| Unprotected software AES, scope + shunt, free-running | thousands |
| Unprotected hardware AES (HD model) | thousands to tens of thousands |
| First-order masked implementation | 10^5 to 10^7, or a second-order attack |
| Hiding (noise, random delays) added | multiplied by the noise, plus alignment work |

If a correct peak will not appear, the usual culprits in order are: wrong trigger/alignment, wrong leakage model (HW vs HD), wrong intermediate, too few traces, or a real countermeasure.

## Worked example: CPA on AES with a ChipWhisperer

Capture traces with known plaintexts and a fixed (unknown-to-the-attack) key, then let the analyzer run CPA with the standard S-box leakage model.

```python
import chipwhisperer as cw
import chipwhisperer.analyzer as cwa

scope  = cw.scope()
target = cw.target(scope, cw.targets.SimpleSerial)
# ... scope capture config as above ...

ktp = cw.ktp.Basic()                 # random plaintexts, fixed key
project = cw.create_project("aes_cpa.cwp", overwrite=True)

N = 200
for i in range(N):
    key, pt = ktp.next()
    trace = cw.capture_trace(scope, target, pt, key)
    if trace is not None:
        project.traces.append(trace)

# --- analysis: correlate Sbox(pt ^ k) Hamming weight against the traces ---
attack  = cwa.cpa(project, cwa.leakage_models.sbox_output)   # HW of first-round Sbox out
results = attack.run()
print("recovered key:", results.key_guess())                  # list of 16 bytes
```

`cwa.leakage_models.sbox_output` is the Hamming-weight-of-first-round-S-box model; swap to `last_round_state_diff` (a Hamming-distance model) for a hardware AES that leaks on the final-round register update. `results.key_guess()` returns the best byte per subkey; `results.find_maximums()` shows the correlation margin between the top guess and the runner-up, which tells you how many traces you actually needed.

Rolling your own is a dozen lines of numpy: for each key-byte guess `k`, build the predicted leakage `L[i] = popcount(sbox[pt[i] ^ k])` across traces `i`, then take the Pearson correlation between `L` and each trace column; the `k` with the tallest absolute peak wins.

## Alignment and preprocessing

Real traces drift in time (jitter), which smears the correlation across samples and kills the peak. Practical attacks spend most of their effort here:

- **Static alignment.** Pick a sharp reference feature in one trace and cross-correlate every other trace to it, shifting each to match. Cheap and effective against small jitter.
- **Filtering.** Band-pass around the clock and its harmonics, or low-pass to kill switching noise, to raise SNR before correlating.
- **Resampling / elastic alignment.** Against **random delays** or an unstable clock, warp traces (dynamic time warping, or "elastic alignment") so features line up even when spacing varies. This is the counter to the hiding countermeasure.
- **Synchronous capture** (ChipWhisperer) sidesteps most of this by sampling in the target's own clock domain, which is why it is the recommended starting platform.

## Template and deep-learning attacks

When you can **profile** a device identical to the target (a clone you fully control):

- **Template attacks** build a statistical model (multivariate Gaussian) of the leakage for each value of the sensitive intermediate during a profiling phase, then match a *handful* of target traces against those templates. They are the most trace-efficient classical attack: profiling is expensive, but the attack on the real target can need only a few traces.
- **Deep-learning SCA** trains a neural network (MLP/CNN) to map a trace to the sensitive intermediate. CNNs are notably robust to misalignment and jitter, which lets them beat some hiding countermeasures without explicit alignment. Needs a large profiling set and tuning, but it is now a standard tool.

## Expectations and countermeasures

Side-channel analysis is measurement-intensive and sensitive to setup quality, but against an unprotected implementation it is remarkably effective, and it needs no code execution on the target. Defenses raise the cost rather than eliminate the leak:

- **Masking:** split each secret-dependent value into random shares so no single intermediate's power correlates with the key. Forces higher-order attacks and many more traces.
- **Hiding:** add noise, random delays, shuffling, or dual-rail logic to flatten and desynchronize the signal. Beaten by more traces and better alignment.
- **Constant-time code** to close timing channels.
- **Leakage-resistant primitives** and limiting how many operations an attacker can trigger (rate limiting, fresh keys).

## Defender's view

Assume physical attackers can measure power and EM. Use constant-time comparisons everywhere a secret is involved, choose cryptographic libraries with side-channel countermeasures (masked AES, constant-time RSA/ECC), mask sensitive operations, add hiding where the threat warrants, and limit the number of key operations an unauthenticated party can invoke. Treat any non-constant-time secret comparison as a bug, and test your own implementation with a leakage assessment (for example TVLA, the fixed-vs-random t-test) before shipping.

## Related pages

- [Fault Injection and Glitching](fault-injection.md)
- [Electromagnetic Fault Injection (EMFI)](emfi.md)
- [ChipWhisperer](../software/chipwhisperer.md)
- [Firmware Analysis](firmware-analysis.md)
- [Automating Attacks](../scripting/automation.md)
