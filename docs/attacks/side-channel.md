# Side-Channel Attacks

A side-channel attack recovers secrets not by reading them directly but by measuring something the device leaks while it uses them: the power it draws, the time it takes, the electromagnetic field it emits. A chip computing with a secret key produces a physical signature that correlates with that key, and with enough measurements and the right analysis, the key falls out. This page introduces the main techniques and the workflow, for cryptographic analysis of hardware you are authorized to test.

## Why it works

Digital logic is physical. Switching a bit from 0 to 1 draws a slightly different amount of current than leaving it at 0, and that difference depends on the data being processed. When that data is a secret key mixed with known input, the power trace carries a faint, exploitable correlation with the key. The attack is statistical: individual traces are noisy, but averaging over many measurements extracts the signal.

## The channels

| Channel | What you measure | Typical gear |
| --- | --- | --- |
| **Power** | Current the chip draws over time | Shunt resistor + oscilloscope, or a [ChipWhisperer](../getting-started/toolkit.md#specialist-gear) |
| **Electromagnetic (EM)** | Near-field emissions from the die | EM probe + amplifier + scope |
| **Timing** | How long an operation takes | Precise timing of responses |

Power and EM are closely related; EM lets you localize to a region of the chip and avoids inserting a shunt.

## Timing attacks: the simplest case

Some code leaks through *how long* it runs. The classic example is a **non-constant-time comparison**, such as a password or MAC check that returns as soon as it finds the first wrong byte:

```c
for (i = 0; i < len; i++)
    if (input[i] != secret[i]) return WRONG;   // returns early
return OK;
```

Each additional correct leading byte makes the check take marginally longer. By measuring response time while varying one byte at a time, an attacker recovers the secret byte by byte, turning an exponential search into a linear one. The defense is a **constant-time** comparison that always examines every byte. Timing attacks need no special hardware, only precise measurement, so they are the first side channel to consider.

## Simple Power Analysis (SPA)

SPA reads secrets from the *shape* of a single or a few power traces. Different operations draw visibly different power, so you can sometimes see the structure of a computation directly:

- A square-and-multiply modular exponentiation (RSA) can reveal key bits because a "multiply" step looks different from a "square" step, and multiplies happen only on 1 bits.
- The rounds of a block cipher are often countable in a trace, which helps you align measurements.

SPA works when the leak is large and visible. When it is not, you need statistics.

## Differential and Correlation Power Analysis (DPA / CPA)

The powerful, general technique. Instead of reading one trace, you collect **many** traces with known, varying inputs and correlate them against a hypothesis about the key.

The idea, for a byte of an AES key:

1. Capture power traces while the device encrypts many known plaintexts.
2. For each of the 256 possible values of one key byte, predict an intermediate value's power consumption (commonly the Hamming weight of an S-box output) for every plaintext.
3. Correlate each hypothesis against the measured traces at each time sample.
4. The correct key-byte guess produces a sharp correlation peak; the wrong ones stay in the noise.
5. Repeat for each key byte to recover the full key.

CPA (correlation) is the modern refinement of DPA and is what most tools implement. The beauty is that it attacks one key byte at a time, so 16 bytes of an AES key cost 16 small searches, not one astronomically large one.

## The workflow

1. **Find the leak.** Identify where the device performs the crypto (a boot-time signature check, a response to a challenge). Trigger measurement at that moment, as in [fault injection](fault-injection.md#the-workflow).
2. **Instrument the power.** Insert a small shunt resistor in the core supply and probe the voltage across it, or use an EM probe over the die. A ChipWhisperer integrates capture and trigger.
3. **Collect traces.** Record many traces (thousands to millions depending on the target and countermeasures) with known inputs, keeping them time-aligned.
4. **Analyze.** Run CPA with a leakage model (Hamming weight/distance) using a toolkit such as the **ChipWhisperer** software, `scared`, or `lascar`.
5. **Recover and verify** the key byte by byte, then confirm it decrypts correctly.

## Alignment and preprocessing

Real traces drift in time (jitter), which smears the correlation. Practical attacks spend effort on **alignment** (lining traces up on a reference feature), filtering, and averaging. Countermeasures like random delays exist specifically to defeat alignment, and resynchronization techniques exist to undo them. This preprocessing is often where the real work is.

## Expectations and countermeasures

Side-channel analysis is measurement-intensive and sensitive to setup quality, but against an unprotected implementation it is remarkably effective, and it needs no code execution on the target. Defenses raise the cost rather than eliminate the leak:

- **Masking:** randomize intermediate values so their power no longer correlates with the key.
- **Hiding:** add noise, random delays, or dual-rail logic to flatten the signal.
- **Constant-time code** to close timing channels.
- **Leakage-resistant primitives** and limiting how many operations an attacker can trigger.

## Defender's view

Assume physical attackers can measure power and EM. Use constant-time comparisons everywhere a secret is involved, choose cryptographic libraries with side-channel countermeasures, mask sensitive operations, and limit the number of key operations an unauthenticated party can invoke. Treat any non-constant-time secret comparison as a bug.


## Related pages

- [Fault Injection and Glitching](fault-injection.md)
- [Firmware Analysis](firmware-analysis.md)
- [Automating Attacks](../scripting/automation.md)
