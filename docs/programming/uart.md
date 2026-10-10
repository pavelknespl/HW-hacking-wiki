# UART

A UART sends bytes over two wires, one per direction: your `TX` goes to the other side's `RX` and vice versa. There is no clock line, so both ends must agree on the **baud rate** (bits per second) and the frame format, almost always **8N1** (8 data bits, no parity, 1 stop bit). GPS modules, Bluetooth and GSM modules, other MCUs, and the debug consoles described on the [UART protocol page](../protocols/uart.md) all talk this way.

## Wiring

```
  MCU TX  ───────── device RX
  MCU RX  ───────── device TX
  MCU GND ───────── device GND
```

Crossing TX and RX is the rule, and forgetting it is the most common mistake. Check the voltage too: a 5 V or RS-232 (±12 V) device needs a [level shifter](../fundamentals/level-shifting.md) or a MAX3232 before it touches a 3.3 V pin.

Most boards already use one UART for the USB console (`Serial` in Arduino, the REPL in MicroPython). The examples below use a **second** UART so the console stays free for debugging: Pico UART0 on GP0/GP1, ESP32 UART2 on GPIO17/16, `Serial1` on Arduino boards that have one.

## Initialise, send and receive

=== "MicroPython"

    ```python
    from machine import UART, Pin

    uart = UART(0, baudrate=115200, tx=Pin(0), rx=Pin(1))   # Pico UART0
    # ESP32: UART(2, baudrate=115200, tx=17, rx=16)

    uart.write(b"hello\r\n")

    if uart.any():                 # bytes waiting in the receive buffer?
        data = uart.read()         # everything available, or None
        print(data)

    line = uart.readline()         # up to b"\n", or None on timeout
    ```

    Pass `timeout=` (ms) to the constructor to control how long `read()` and `readline()` wait, and `rxbuf=` to enlarge the receive buffer for bursty devices.

=== "Arduino"

    ```cpp
    void setup() {
      Serial.begin(115200);                       // USB console
      Serial1.begin(115200);                      // hardware UART
      // ESP32: Serial1.begin(115200, SERIAL_8N1, 16, 17);   // RX, TX
      // RP2040: Serial1.setTX(0); Serial1.setRX(1); Serial1.begin(115200);
      Serial1.println("hello");
    }

    void loop() {
      while (Serial1.available()) {
        char c = Serial1.read();
        Serial.write(c);                          // echo to the console
      }
    }
    ```

=== "Pico SDK"

    ```c
    #include "pico/stdlib.h"
    #include "hardware/uart.h"

    int main(void) {
        stdio_init_all();                         // console on USB
        uart_init(uart0, 115200);
        gpio_set_function(0, GPIO_FUNC_UART);     // TX
        gpio_set_function(1, GPIO_FUNC_UART);     // RX

        uart_puts(uart0, "hello\r\n");
        while (true) {
            if (uart_is_readable(uart0)) {
                char c = uart_getc(uart0);
                putchar(c);
            }
        }
    }
    ```

    Link `hardware_uart`. Set `pico_enable_stdio_uart(app 0)` so `printf` does not also use UART0.

=== "ESP-IDF"

    ```c
    #include "driver/uart.h"

    #define PORT UART_NUM_2

    void app_main(void) {
        uart_config_t cfg = {
            .baud_rate = 115200,
            .data_bits = UART_DATA_8_BITS,
            .parity    = UART_PARITY_DISABLE,
            .stop_bits = UART_STOP_BITS_1,
            .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
            .source_clk = UART_SCLK_DEFAULT,
        };
        uart_driver_install(PORT, 1024, 0, 0, NULL, 0);   // RX buffer 1 KB
        uart_param_config(PORT, &cfg);
        uart_set_pin(PORT, 17, 16, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);

        uart_write_bytes(PORT, "hello\r\n", 7);
        uint8_t buf[128];
        while (1) {
            int n = uart_read_bytes(PORT, buf, sizeof buf, pdMS_TO_TICKS(100));
            if (n > 0) printf("%.*s", n, buf);
        }
    }
    ```

=== "STM32 HAL"

    ```c
    // CubeMX: USART1 asynchronous, 115200 8N1. (USART2 is wired to the ST-LINK
    // virtual COM port on Nucleo boards, handy as a console.)
    extern UART_HandleTypeDef huart1;

    HAL_UART_Transmit(&huart1, (uint8_t *)"hello\r\n", 7, HAL_MAX_DELAY);

    uint8_t c;
    if (HAL_UART_Receive(&huart1, &c, 1, 10) == HAL_OK) {
        // got one byte within 10 ms
    }
    ```

    Polling with `HAL_UART_Receive` drops bytes if the main loop is busy. For real use, receive with `HAL_UART_Receive_IT` or `HAL_UARTEx_ReceiveToIdle_DMA` and handle data in the callback.

## Reading lines without blocking

Most text protocols (GPS NMEA, AT commands, consoles) are line based. Collect bytes into a buffer and act on each complete line, without ever stopping the main loop:

=== "MicroPython"

    ```python
    buf = b""
    while True:
        if uart.any():
            buf += uart.read()
            while b"\n" in buf:
                line, buf = buf.split(b"\n", 1)
                handle(line.strip())
        # ... other work here ...
    ```

=== "Arduino"

    ```cpp
    char line[128];
    size_t len = 0;

    void loop() {
      while (Serial1.available()) {
        char c = Serial1.read();
        if (c == '\n') {
          line[len] = '\0';
          handle(line);
          len = 0;
        } else if (c != '\r' && len < sizeof line - 1) {
          line[len++] = c;
        }
      }
      // ... other work here ...
    }
    ```

## Example: AT commands

Many modules (ESP-01, SIM800, HC-05, LoRa modems) take text commands and answer `OK` or `ERROR`. A small helper that sends a command and waits for the reply covers most of them:

```python
import time

def at(cmd, timeout_ms=1000):
    uart.read()                                  # drop stale input
    uart.write(cmd + b"\r\n")
    deadline = time.ticks_add(time.ticks_ms(), timeout_ms)
    resp = b""
    while time.ticks_diff(deadline, time.ticks_ms()) > 0:
        if uart.any():
            resp += uart.read()
            if b"OK" in resp or b"ERROR" in resp:
                break
    return resp

print(at(b"AT"))           # b'AT\r\nOK\r\n'
```

## USB to UART bridge

Forwarding bytes in both directions between the USB console and a hardware UART turns any board into a USB serial adapter. It is useful when you have no FTDI cable at hand, and it is the starting point for logging or filtering traffic.

=== "Arduino"

    ```cpp
    void setup() {
      Serial.begin(115200);
      Serial1.begin(115200);
    }

    void loop() {
      while (Serial.available())  Serial1.write(Serial.read());
      while (Serial1.available()) Serial.write(Serial1.read());
    }
    ```

=== "MicroPython"

    ```python
    import sys, select
    from machine import UART, Pin

    uart = UART(0, 115200, tx=Pin(0), rx=Pin(1))
    poll = select.poll()
    poll.register(sys.stdin, select.POLLIN)

    while True:
        if uart.any():
            sys.stdout.buffer.write(uart.read())
        if poll.poll(0):
            uart.write(sys.stdin.buffer.read(1))
    ```

    Run it with `mpremote run bridge.py`. The REPL's ++ctrl+c++ handling still intercepts that key, so the C version is better for binary data.

## Finding an unknown baud rate

If you see garbage, the baud rate is wrong. Standard rates are 9600, 19200, 38400, 57600, 115200, 230400, 460800, and 921600, and trying them in order is usually fastest. Measuring is more precise: the shortest low pulse on the line is one bit time, so `baud = 1 / shortest_pulse`. A [logic analyzer](../software/logic-analyzers.md) shows this directly, and the [UART protocol page](../protocols/uart.md) covers it in detail.

## Common pitfalls

- **TX to TX.** Swap the wires.
- **No common ground.** The signals have no reference and you get noise.
- **Buffer overruns.** At 115200 baud a byte arrives every 87 µs. A blocking `delay()` or a slow `print` in the loop loses data. Use larger RX buffers, interrupts, or DMA.
- **Boot messages on the console UART.** The ESP32 prints its ROM log on UART0 at 115200 at every reset, and some pins toggle during boot. Use a different UART for devices that react to stray bytes.
- **Inverted signals.** RS-232 levels are inverted and up to ±12 V. Never connect them directly; use a MAX3232.
