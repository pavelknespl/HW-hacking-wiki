# ChipWhisperer

The ChipWhisperer is an integrated, open-source platform for [fault injection](../attacks/fault-injection.md) and [side-channel analysis](../attacks/side-channel.md). It combines a synchronized capture scope, a glitch generator, and target boards with a Python API and a large tutorial set, which is why it is the recommended way to learn both disciplines rather than building a rig from scratch. This page is a practical orientation; the technique details live on the attack pages.

## The models

| Model | Role |
| --- | --- |
| **ChipWhisperer-Nano** | Cheapest, voltage glitching and basic power analysis on its built-in target. Great for learning glitching. |
| **ChipWhisperer-Lite** | The common starting point: synchronous capture plus glitching, with a target board. |
| **ChipWhisperer-Husky** | Faster, more capable capture and glitching for tougher targets. |
| **ChipWhisperer-Pro** | Higher-end capture, streaming, bigger buffers. |

Add-ons cover EM injection probes, a crowbar glitch transistor, and target boards for various MCUs.

## Why "synchronous" capture matters

The ChipWhisperer clocks its ADC from the **target's own clock** rather than a free-running sampler. That synchronization dramatically reduces jitter, so power traces line up and [correlation power analysis](../attacks/side-channel.md#differential-and-correlation-power-analysis-dpa-cpa) needs far fewer traces than with a generic oscilloscope. It is the single feature that makes side-channel practical on a budget.

## Glitching workflow

The hardware provides the trigger, the precise glitch offset/width, and the target reset, so your [glitch loop](../attacks/fault-injection.md#the-workflow) is a few lines of Python:

```python
import chipwhisperer as cw
scope = cw.scope()
target = cw.target(scope)
scope.glitch.clk_src = "clkgen"
scope.glitch.output  = "enable_only"
scope.glitch.trigger_src = "ext_single"

for width in range(w_min, w_max):
    for offset in range(o_min, o_max):
        scope.glitch.width  = width
        scope.glitch.offset = offset
        reset_target(scope)
        scope.arm()
        target.simpleserial_write(...)   # provoke the operation
        scope.capture()
        resp = target.simpleserial_read(...)
        classify(width, offset, resp)    # normal | reset | SUCCESS
```

The framework records and plots the parameter sweep, so the success window stands out visually. This is exactly the campaign described for [STM32 RDP](../mcu/stm32-rdp.md) and [nRF52 APPROTECT](../mcu/nrf52-approtect.md).

## Side-channel workflow

For [power analysis](../attacks/side-channel.md), the ChipWhisperer captures aligned traces while the target encrypts known inputs, then its analyzer (or the `scared`/`lascar` libraries) runs CPA to recover an AES key byte by byte:

```python
traces = []
for pt in plaintexts:
    scope.arm(); target.simpleserial_write('p', pt); scope.capture()
    traces.append(scope.get_last_trace())
# feed traces + plaintexts to a CPA attack with a Hamming-weight model
```

The official Jupyter tutorials walk through recovering an AES key from an unprotected target end to end, which is the best way to internalize the method.

## Why start here

Building a glitcher or a low-jitter capture rig from discrete parts is possible (a crowbar MOSFET plus an FPGA), but the ChipWhisperer removes the synchronization and timing problems that otherwise dominate the work, and its tutorials teach the concepts against known-vulnerable targets before you face a real one. It is the fastest path from "I have read about glitching" to "I have defeated a protection bit".

## Related pages

- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [Side-Channel Attacks](../attacks/side-channel.md)
- [STM32 RDP](../mcu/stm32-rdp.md)
- [nRF52 APPROTECT](../mcu/nrf52-approtect.md)
- [Automating Attacks](../scripting/automation.md)
