# JTAGulator

The JTAGulator is a small open-source board built for one job: finding [JTAG](../protocols/jtag.md) and [UART](../protocols/uart.md) pinouts on an unknown header by brute force. When you have a row of unlabeled test points and suspect a debug interface, it drives every combination and tells you which pins are what, saving the tedious manual probing described on [Finding and Identifying Pins](../fundamentals/identifying-pins.md).

## What it does

The JTAGulator connects to up to 24 channels and systematically permutes which pins might be each signal, watching for a valid response:

- **JTAG discovery**: it drives candidate `TCK`/`TMS`/`TDI`/`TDO` combinations and runs the IDCODE and BYPASS scans. A returned **IDCODE** confirms both that JTAG is present and which pins are which.
- **UART discovery**: it drives candidate TX/RX pairs across a range of baud rates and watches for readable ASCII, identifying a console and its speed in one pass.

It handles the target voltage selection (you set the target's logic level) so it probes safely at the right level.

## Using it

1. Wire the target's unknown header pins to the JTAGulator channels, plus a common ground.
2. Set the **target voltage** to match the board (measure it first; see [Level Shifting](../fundamentals/level-shifting.md)).
3. Open its serial console and run the scan:

```
# JTAGulator serial menu (conceptual)
Set target voltage: 3.3
J  -> JTAG
   I  -> Identify JTAG pinout (IDCODE scan)
U  -> UART
   U  -> Identify UART pinout (scan baud + pins)
```

4. For JTAG, a successful IDCODE scan prints the channel assignments for TCK/TMS/TDI/TDO and the device IDCODE. For UART, it prints the TX/RX channels and the baud rate.

## When to use it versus alternatives

- **JTAGulator**: dedicated hardware, handles voltage, fast and reliable across 24 channels. The clean choice when you do this often.
- **jtagenum**: the same brute-force idea in software on an Arduino/Raspberry Pi, driving candidate pins and watching for a TAP. Cheap and effective if you already have the board; mind the logic level. See [GPIO Control](../scripting/gpio-control.md).
- **Manual** [logic-analyzer](logic-analyzers.md) + reasoning: works and teaches you the signals, but slow on a dense header.

## After identification

Once the JTAGulator gives you the pinout, connect [OpenOCD](openocd.md) over [JTAG](../protocols/jtag.md) (or a [serial terminal](serial-consoles.md) for a found UART) and proceed to halt the CPU, dump memory, or read the boot log.

## Related pages

- [Finding and Identifying Pins](../fundamentals/identifying-pins.md)
- [JTAG](../protocols/jtag.md)
- [UART](../protocols/uart.md)
- [OpenOCD](openocd.md)
- [GPIO Control](../scripting/gpio-control.md)
