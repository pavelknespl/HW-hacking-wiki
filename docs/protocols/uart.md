# UART

UART, the Universal Asynchronous Receiver Transmitter, is the oldest and most useful interface in the hardware hacker's kit. It is how an embedded device prints its boot log, and very often how it offers a shell. If a board has a UART header with an open console, you may be done before you pick up a soldering iron.

## How it works

UART is a point-to-point serial link with no shared clock. Two devices agree in advance on a **baud rate** (bits per second) and a frame format, then each sends data on its transmit line at that rate.

- **TX** (transmit): the line a device sends on.
- **RX** (receive): the line a device listens on.
- **GND**: a shared ground reference, mandatory.
- Optional **VCC**, which you usually leave unconnected on the target.

The classic frame is **8N1**: 8 data bits, no parity, 1 stop bit. The line idles high, drops low for a start bit, sends 8 data bits least significant first, then returns high for the stop bit.

Because there is no clock, both sides must use the same baud rate. Get it wrong and you see garbage.

## Wiring

Connect your [USB-to-UART adapter](../getting-started/toolkit.md#usb-to-uart-adapters) to the target, crossing TX and RX:

| Adapter | Target |
| --- | --- |
| RX | TX |
| TX | RX |
| GND | GND |
| (leave unconnected) | VCC |

!!! warning "Match the voltage"
    Set your adapter to the target's logic level (usually 3.3 V, sometimes 1.8 V). A 5 V adapter on a 3.3 V UART can damage the SoC. See [Level Shifting](../fundamentals/level-shifting.md).

!!! tip "If you only connect one line, make it RX on your adapter"
    To just read the boot log, connecting your adapter's RX to the target's TX plus ground is enough. You only need your TX wired when you want to type back.

## Finding the header

A UART is a group of 3 or 4 pads near the SoC. The TX pin bursts with activity the moment the device powers on. See [Finding and Identifying Pins](../fundamentals/identifying-pins.md) for the full identification procedure.

## Determining the baud rate

Common rates, most likely first: **115200**, 57600, 38400, 19200, 9600, and sometimes 230400 or 1500000 on faster SoCs.

Two ways to find it:

1. **Measure a bit width** with a [logic analyzer](../software/logic-analyzers.md). The narrowest pulse on TX is one bit. Its inverse is the baud rate. About 8.68 microseconds means 115200.
2. **Try them in a terminal.** Open the port, cycle through the common rates, and watch for readable ASCII in the boot log. Scripts that automate this (sometimes called baudrate detectors) exist for exactly this.

## Reading the console

Open the port with a [serial terminal](../software/serial-consoles.md):

```bash
# screen
screen /dev/ttyUSB0 115200

# tio (nicer defaults, auto-reconnect)
tio -b 115200 /dev/ttyUSB0

# minicom
minicom -D /dev/ttyUSB0 -b 115200

# picocom
picocom -b 115200 /dev/ttyUSB0
```

Power-cycle the target and you should see the bootloader and kernel log stream past.

## What the console gives you

Even a read-only boot log is valuable:

- The **SoC and CPU model**, confirming the [architecture](../fundamentals/architectures.md).
- The **bootloader** (usually U-Boot) and its version, and whether you can interrupt it. See [Bootloader Access](../attacks/bootloader-access.md).
- The **kernel version**, command line, and the **memory layout** of the flash partitions.
- Driver messages that reveal chips, radios, and features.
- Sometimes a login prompt, and sometimes a **root shell with no password at all**.

## Getting a shell

The outcomes, from best to most work:

1. **Open root shell.** The log ends at a `#` prompt. You are in. See [Getting a Shell](../attacks/getting-a-shell.md).
2. **Login prompt.** Try default and discovered credentials. Firmware analysis often recovers the password hash or a hardcoded account.
3. **Interruptible bootloader.** A prompt like "Hit any key to stop autoboot" lets you break into U-Boot and change the boot arguments to force a shell. See [Bootloader Access](../attacks/bootloader-access.md).
4. **Silent or disabled console.** TX prints but RX is ignored, or the console is locked. Fall back to firmware extraction or a debug port.

## TX active but no input accepted

A common frustration: the log prints but typing does nothing. Causes and fixes:

- **RX not actually connected** or wired to the wrong pad. Re-check the pinout.
- **Console input disabled** in the kernel command line or U-Boot environment. Interrupt the bootloader earlier, or extract the firmware to find why.
- **Voltage mismatch** on the RX line so your characters are misread. Verify with a logic analyzer.
- **Flow control** expected. Try toggling hardware/software flow control in your terminal.

## Security note for defenders

Shipping a production device with an open UART root shell, or an interruptible bootloader, hands local attackers the keys. Disable the console shell, set a bootloader password, and lock the boot sequence on production builds. Attackers read this interface first precisely because it is so often left open.
