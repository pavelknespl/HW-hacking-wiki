# Automating Attacks

The difference between a demonstration and a result is often repetition. A [glitch](../attacks/fault-injection.md) that works one time in a thousand is useless by hand and routine in a script. Automation also makes extraction reproducible, lets you sweep parameter spaces, and turns an overnight run into data. This page shows how to wrap the techniques from the rest of the wiki into scripts.

## The universal attack loop

Almost every automated hardware attack is the same loop: put the target in a known state, do something, observe, record, repeat.

```python
def attempt(params):
    reset_target()                 # known starting state (pulse reset via GPIO)
    arm(params)                    # set up this iteration (glitch offset/width, input)
    trigger()                      # perform the action
    result = observe()             # read UART, check a pin, read memory
    return classify(result)        # normal | crash | success

for params in parameter_space():
    for _ in range(repeats):
        outcome = attempt(params)
        log(params, outcome)
        if outcome == "success":
            notify(params)
```

Everything else is filling in `reset_target`, `arm`, `trigger`, `observe`, and `classify` for your target, using the [GPIO and bus control](gpio-control.md) primitives.

## Scripting a flash dump

Even a simple dump benefits from scripting: read several times, compare, and only keep a verified image.

```bash
#!/usr/bin/env bash
set -euo pipefail
PROG="ch341a_spi"
for i in 1 2 3; do
    flashrom -p "$PROG" -r "dump_$i.bin"
done
if sha256sum dump_1.bin | cut -d' ' -f1 | \
   grep -qx "$(sha256sum dump_2.bin | cut -d' ' -f1)"; then
    echo "stable read -> firmware.bin"
    cp dump_1.bin firmware.bin
else
    echo "reads differ; interference likely, see spi-flash-dump troubleshooting" >&2
    exit 1
fi
```

This encodes the ["read three times and compare"](../software/flashrom.md#read-three-times-and-compare) discipline so you cannot forget it.

## Scripting bootloader interaction

Catching the [U-Boot interrupt window](../attacks/bootloader-access.md) by hand is unreliable. Script the [UART](../protocols/uart.md) to flood keys at power-on and then issue commands:

```python
import serial, time
ser = serial.Serial("/dev/serial0", 115200, timeout=0.2)

power_cycle_target()                      # e.g. toggle a relay or GPIO-controlled power
deadline = time.time() + 5
while time.time() < deadline:             # spam to hit the autoboot window
    ser.write(b"\x03")                    # try Ctrl-C; some builds want a key
    ser.write(b"\n")
    if b"=>" in ser.read(64):             # U-Boot prompt
        break

ser.write(b"setenv bootargs \"${bootargs} init=/bin/sh\"\n")
ser.write(b"boot\n")
print(ser.read(2000).decode(errors="replace"))
```

The script wins the timing race every time, where a human misses it more often than not.

## Automating a glitch campaign

Glitching is the canonical case for automation because success is rare and the parameter space is large. The structure, whether you use a [ChipWhisperer](../attacks/fault-injection.md#the-equipment) or a custom GPIO-triggered glitcher:

```python
results = []
for offset in range(OFF_MIN, OFF_MAX, OFF_STEP):      # delay after trigger
    for width in range(W_MIN, W_MAX, W_STEP):          # glitch length
        for _ in range(ATTEMPTS):
            reset_target()
            wait_for_trigger()                         # the reference moment
            glitch(offset, width)                      # crowbar / clock pulse
            r = read_response()
            state = classify(r)                        # normal | reset | SUCCESS
            results.append((offset, width, state))
            if state == "SUCCESS":
                print(f"hit at offset={offset} width={width}")
```

Afterward, plot the results: the map of normal/crash/success over the parameter space shows the window where faults land, which is usually at the boundary between "normal" and "crash." The [ChipWhisperer](https://chipwhisperer.readthedocs.io/) software provides this scan-and-plot framework out of the box, and is the recommended place to start.

## Instrumenting the observation

Reliable `classify()` is what makes a campaign work. Good success signals:

- A UART string that only appears on success (a prompt, a leaked value).
- A [debug port read](../software/openocd.md) that returns real data instead of zeros ([read-out protection](../protocols/swd.md) defeated).
- A status pin that changes level.
- A change in current draw on the [bench supply](../getting-started/lab-setup.md#powering-the-target).

Make the signal unambiguous and machine-readable so an overnight run needs no babysitting.

## Driving tools from scripts

You rarely need to reimplement protocols. Script the existing tools:

- **`flashrom`**, **`nanddump`**, **`dd`** from shell for extraction.
- **[OpenOCD](../software/openocd.md)** via its Tcl/telnet interface, or **pyOCD** from Python, to halt, dump, and flash in a loop. See [OpenOCD](../software/openocd.md).
- **`esptool.py`** for Espressif targets.
- **sigrok-cli** to capture and auto-decode a bus as part of a test. See [Logic Analyzers](../software/logic-analyzers.md).

## Logging and reproducibility

Treat a campaign like an experiment:

- Log every attempt with its parameters and outcome, in a format you can plot (CSV or JSON lines).
- Record the environment: target, voltage, temperature if relevant, tool versions.
- Save the exact scripts with the results so a successful run is reproducible later.

Reproducibility is what turns a lucky glitch into a documented, repeatable finding, which is the whole point of writing the automation in the first place.


## Related pages

- [GPIO Control](gpio-control.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [flashrom](../software/flashrom.md)
- [OpenOCD](../software/openocd.md)
