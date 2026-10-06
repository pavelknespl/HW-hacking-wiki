# Serial Consoles

A serial terminal is how you read a [UART](../protocols/uart.md) boot log and drive a device shell. Several programs do the job; they differ mostly in ergonomics. This page covers the four you will actually use, with sane settings and the gotchas that trip people up.

## The adapter and the device node

Plug in your USB-to-UART adapter and find its device node:

```bash
# list likely serial devices and watch the kernel log on plug-in
ls /dev/ttyUSB* /dev/ttyACM*
sudo dmesg | tail
```

FTDI and CP210x adapters usually appear as `/dev/ttyUSB0`; CDC-ACM devices (many onboard debuggers) as `/dev/ttyACM0`.

!!! tip "Permissions"
    If you get "permission denied", add yourself to the `dialout` group (`sudo usermod -aG dialout $USER`, then log out and back in) rather than running your terminal as root.

## tio (recommended)

`tio` has the friendliest defaults: it reconnects automatically when a device re-enumerates, logs easily, and does not hijack your terminal with obscure escape sequences.

```bash
tio -b 115200 /dev/ttyUSB0
```

- `-b` sets the baud rate.
- `tio --list` shows available ports.
- Press ++ctrl+t++ then ++q++ to quit; ++ctrl+t++ then ++question++ shows the key help.
- Log a session: `tio -b 115200 --log session.log /dev/ttyUSB0`.

The auto-reconnect is a real advantage on hardware that resets or re-enumerates during boot.

## screen

Ubiquitous and already installed on most systems. Good enough, with slightly awkward exit.

```bash
screen /dev/ttyUSB0 115200
```

- Exit: ++ctrl+a++ then ++k++ (kill), confirm with ++y++. Or detach with ++ctrl+a++ then ++d++.
- Log to a file: ++ctrl+a++ then ++h++ toggles a hardcopy/log.
- If a previous `screen` left the port busy, `screen -ls` and `screen -X -S <id> quit` clean it up.

!!! warning "Stuck terminal after screen"
    If your shell behaves oddly after exiting `screen`, run `reset` to restore it.

## minicom

Feature-rich, menu-driven, the traditional choice. Worth configuring once.

```bash
minicom -D /dev/ttyUSB0 -b 115200
```

- Enter the menu with ++ctrl+a++ then ++z++.
- Turn **hardware flow control off** in the menu the first time (++ctrl+a++ ++o++, then Serial port setup). Leaving it on is the usual reason minicom shows a dead console: the device is not asserting the control lines minicom waits for.
- Save a default profile with `minicom -s`.

## picocom

Lightweight, minimal, scriptable. A nice middle ground.

```bash
picocom -b 115200 /dev/ttyUSB0
```

- Exit: ++ctrl+a++ then ++ctrl+q++.
- Add `--imap lfcrlf` if the device expects CRLF line endings and your Enter key seems ignored.

## Settings that matter

| Setting | Usual value | When to change |
| --- | --- | --- |
| Baud rate | 115200 | Try 57600, 9600 if you see garbage. See [UART](../protocols/uart.md). |
| Data/parity/stop | 8N1 | Almost never changes on embedded devices. |
| Hardware flow control | Off | Turn it **off**; a common cause of a dead console (especially minicom). |
| Software flow control | Off | Rarely needed. |
| Line ending | Device-dependent | Switch CR/LF if Enter is ignored. |

## Diagnosing a silent or garbled console

- **Pure garbage characters:** wrong baud rate. Cycle through the common rates.
- **Nothing at all on boot:** TX not connected, or your adapter RX wired to the wrong pad, or a voltage mismatch. Re-check wiring and level. See [Finding and Identifying Pins](../fundamentals/identifying-pins.md).
- **Log prints but typing does nothing:** RX side not wired, console input disabled, or flow control. See the [UART troubleshooting section](../protocols/uart.md#tx-active-but-no-input-accepted).
- **Characters doubled or missing:** local echo mismatch; toggle echo in your terminal, or CRLF handling.

## Logging is not optional

Always capture the session to a file. Boot logs scroll fast and contain the architecture, partition layout, and sometimes credentials. `tio --log`, `screen`'s hardcopy, or simply `picocom | tee` all work. Your [lab notebook](../getting-started/lab-setup.md#keep-a-lab-notebook) should end up with the full boot log of every target.
