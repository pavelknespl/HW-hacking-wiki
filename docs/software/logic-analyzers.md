# Logic Analyzers (sigrok)

A logic analyzer captures the state of several digital lines over time and lets you decode them back into protocol transactions. It turns "I think these pads are a [UART](../protocols/uart.md)" into "this is a 115200 baud UART and here is the boot log." For most hardware hacking an inexpensive 8-channel analyzer and the open-source [sigrok](https://sigrok.org/) stack cover the work.

## Hardware

- **Cheap 8-channel clones** (the ubiquitous "Saleae clone" based on the Cypress FX2) sample up to around 24 MS/s and are supported by sigrok as `fx2lafw`. Fine for UART, [I2C](../protocols/i2c.md), and slow [SPI](../protocols/spi.md).
- **Genuine Saleae Logic** units sample faster and have excellent software; sigrok also supports many.
- **Faster/standalone analyzers** (DSLogic, and various USB units) handle fast SPI and [JTAG](../protocols/jtag.md).

The limiting factor is **sample rate**: to capture a signal cleanly you want to sample at least four times, ideally ten times, its fastest edge rate. A 24 MS/s clone comfortably reads a 10 MHz SPI bus; it will struggle with a 50 MHz one.

## sigrok and PulseView

- **`sigrok-cli`** is the command-line capture and decode tool.
- **PulseView** is the GUI: capture, zoom, and stack protocol decoders visually. It is the friendlier starting point.

Install from your distribution or from sigrok.org, then confirm the device is seen:

```bash
sigrok-cli --scan
```

## Wiring

Connect analyzer channels to the signals of interest and, critically, tie the analyzer ground to the target ground. Without a shared ground the capture is meaningless. See [Level Shifting](../fundamentals/level-shifting.md) and confirm the signals are within the analyzer's input voltage range (most FX2 clones are 3.3 V/5 V tolerant; verify for 1.8 V targets, which may read marginally).

Label your channels as you connect them so the decoder setup is unambiguous later.

## Capturing from the command line

```bash
# capture 10 million samples at 24 MHz on channels D0-D3, save to a file
sigrok-cli --driver fx2lafw --config samplerate=24m \
  --channels D0=TX,D1=RX,D2=CLK,D3=CS \
  --samples 10000000 --output-file capture.sr
```

Trigger on an edge so you catch the start of activity (for example the first falling edge of a UART start bit):

```bash
sigrok-cli --driver fx2lafw --config samplerate=24m \
  --channels D0 --triggers D0=f --samples 5000000 --output-file boot.sr
```

## Decoding

sigrok ships protocol decoders for UART, SPI, I2C, JTAG, and many more. Decode a captured file:

=== "UART"

    ```bash
    sigrok-cli -i boot.sr -P uart:rx=D0:baudrate=115200 -A uart
    ```

    The decoded ASCII is the boot log, recovered purely from the wire. Handy when you cannot connect a proper serial adapter but can tap the line.

=== "SPI"

    ```bash
    sigrok-cli -i capture.sr \
      -P spi:clk=D2:mosi=D0:miso=D1:cs=D3 -A spi
    ```

    Reveals the commands the SoC sends to a flash and the data returned, a partial [firmware read](../protocols/spi.md) captured at boot.

=== "I2C"

    ```bash
    sigrok-cli -i capture.sr -P i2c:scl=D2:sda=D0 -A i2c
    ```

    Shows device addresses and byte values, exposing [EEPROM](../protocols/i2c.md) reads and configuration.

In PulseView the same decoders are added from a menu and rendered as annotated lanes under the raw traces, which is far easier for exploration.

## Using it to identify pins

A logic analyzer is the best tool for [identifying unknown pins](../fundamentals/identifying-pins.md):

- Capture at boot and watch which line bursts first (UART TX) and which toggle periodically (clocks).
- Measure the narrowest pulse to compute a UART baud rate or a clock frequency.
- Confirm a guessed pinout by decoding and checking the output is sensible.

## Analyzer versus oscilloscope

A logic analyzer sees digital states (high/low) across many channels and decodes protocols, which is what you want 90% of the time. An **oscilloscope** sees the analog waveform on one or two channels and is the right tool when the question is analog: is this edge clean, how much does the power rail dip, did my [glitch](../attacks/fault-injection.md) land, what is the exact timing of a transient. Own both eventually; reach for the logic analyzer first for bus work.


## Related pages

- [Finding and Identifying Pins](../fundamentals/identifying-pins.md)
- [UART](../protocols/uart.md)
- [SPI](../protocols/spi.md)
- [I2C](../protocols/i2c.md)
