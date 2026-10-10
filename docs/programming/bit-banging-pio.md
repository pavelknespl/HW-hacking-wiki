# Bit-Banging and RP2040 PIO

Hardware peripherals cover the common buses, but sometimes you need a protocol your MCU has no peripheral for, a second instance of one it has, or one on pins the peripheral cannot reach. **Bit-banging** implements the protocol in software by toggling GPIOs directly. It is simple and portable but costs CPU time and has jitter. The RP2040 and RP2350 add a third option, **PIO** (programmable I/O): tiny state machines that run your own I/O programs with cycle-exact timing while the CPU does something else.

## Bit-banging basics

Every protocol is a sequence of pin changes with timing rules. To bit-bang one you need three things from the datasheet: which line changes when, which edge the receiver samples on, and the minimum and maximum times between events.

The simplest example is a software SPI-style shift-out, clocking 8 bits MSB first:

=== "MicroPython"

    ```python
    from machine import Pin

    clk = Pin(10, Pin.OUT, value=0)
    dat = Pin(11, Pin.OUT, value=0)

    def shift_out(byte):
        for i in range(7, -1, -1):
            dat.value((byte >> i) & 1)   # set data while clock is low
            clk.value(1)                 # receiver samples on the rising edge
            clk.value(0)

    shift_out(0b10110001)                # e.g. drive a 74HC595 shift register
    ```

=== "Arduino"

    ```cpp
    // Arduino has this built in:
    shiftOut(11, 10, MSBFIRST, 0b10110001);   // dataPin, clockPin, order, value
    ```

=== "Pico SDK"

    ```c
    static void shift_out(uint clk, uint dat, uint8_t byte) {
        for (int i = 7; i >= 0; i--) {
            gpio_put(dat, (byte >> i) & 1);
            gpio_put(clk, 1);
            gpio_put(clk, 0);
        }
    }
    ```

## Bit-banged I2C

I2C is a good bit-banging exercise because both lines are **open-drain**: you never drive them high, you only pull them low or let go. Letting go is done by switching the pin to an input, so the pull-up resistor raises the line. Releasing `SCL` and waiting for it to actually go high also implements clock stretching.

```c title="soft_i2c.c (Pico SDK, works on any two pins)"
#include "pico/stdlib.h"

#define SDA 10
#define SCL 11
#define DLY() busy_wait_us(5)                  // about 100 kHz

static void low(uint p)     { gpio_set_dir(p, GPIO_OUT); }   // output latch is 0
static void release(uint p) { gpio_set_dir(p, GPIO_IN);  }   // pull-up makes it 1

static void scl_high(void) {
    release(SCL);
    while (!gpio_get(SCL)) { }                 // device may stretch the clock
    DLY();
}

void si2c_init(void) {
    for (uint p = SDA; p <= SCL; p++) {
        gpio_init(p);
        gpio_put(p, 0);
        gpio_pull_up(p);
        release(p);
    }
}

static void start(void) { release(SDA); scl_high(); low(SDA); DLY(); low(SCL); DLY(); }
static void stop(void)  { low(SDA); DLY(); scl_high(); release(SDA); DLY(); }

static bool write_byte(uint8_t b) {
    for (int i = 7; i >= 0; i--) {
        if (b & (1u << i)) release(SDA); else low(SDA);
        DLY(); scl_high(); low(SCL);
    }
    release(SDA); DLY(); scl_high();            // ninth clock: device ACKs by pulling SDA low
    bool ack = !gpio_get(SDA);
    low(SCL);
    return ack;
}

static uint8_t read_byte(bool ack) {
    uint8_t b = 0;
    release(SDA);
    for (int i = 0; i < 8; i++) {
        DLY(); scl_high();
        b = (b << 1) | gpio_get(SDA);
        low(SCL);
    }
    if (ack) low(SDA); else release(SDA);       // ACK all bytes except the last
    DLY(); scl_high(); low(SCL);
    release(SDA);
    return b;
}

int si2c_read_reg(uint8_t addr, uint8_t reg, uint8_t *buf, int n) {
    start();
    if (!write_byte(addr << 1)) { stop(); return -1; }
    write_byte(reg);
    start();                                    // repeated start
    write_byte((addr << 1) | 1);
    for (int i = 0; i < n; i++) buf[i] = read_byte(i < n - 1);
    stop();
    return n;
}
```

This is the logic every hardware I2C controller implements in silicon. Compare it with the [I2C page](i2c.md) and the [protocol description](../protocols/i2c.md).

## Why PIO exists

Bit-banged code has two weaknesses. Its timing depends on the compiler, the cache, and interrupts, so it jitters. And while the CPU toggles pins it does nothing else. The RP2040 solves both with two PIO blocks, each containing four state machines. A state machine:

- runs a program of up to 32 instructions shared per block, one instruction per clock cycle (8 ns at 125 MHz), with a fractional clock divider;
- has two 32-bit shift registers (`OSR` out, `ISR` in), two scratch registers (`X`, `Y`), and 4-word FIFOs to and from the CPU, which DMA can feed;
- can drive and sample a contiguous range of pins, and toggle "side-set" pins for free on any instruction.

There are only nine instructions:

| Instruction | Does |
| --- | --- |
| `jmp` | Jump, optionally on a condition (`!x`, `x--`, `pin`, ...) |
| `wait` | Stall until a pin or IRQ flag reaches a level |
| `in` | Shift bits from pins or a register into `ISR` |
| `out` | Shift bits from `OSR` to pins or a register |
| `push` / `pull` | Move `ISR` to the RX FIFO / TX FIFO to `OSR` |
| `mov` | Copy between registers, with optional invert or bit-reverse |
| `irq` | Set or clear an IRQ flag (to the CPU or other state machines) |
| `set` | Write an immediate value to pins or a register |

Every instruction can also carry a **delay** `[n]` (wait n extra cycles) and a **side-set** value, which is what makes tight protocols fit in a handful of instructions.

## First PIO program: a square wave

=== "MicroPython"

    ```python
    import rp2
    from machine import Pin

    @rp2.asm_pio(set_init=rp2.PIO.OUT_LOW)
    def square():
        wrap_target()
        set(pins, 1)
        set(pins, 0)
        wrap()

    # 2 instructions per period: 2 MHz instruction clock -> 1 MHz square wave
    sm = rp2.StateMachine(0, square, freq=2_000_000, set_base=Pin(15))
    sm.active(1)
    ```

    `wrap_target()` and `wrap()` mark a loop that costs zero cycles to jump back. Change `freq` and the output follows exactly, with no jitter, whatever Python is doing.

=== "Pico SDK"

    ```asm title="square.pio"
    .program square
    .wrap_target
        set pins, 1
        set pins, 0
    .wrap
    ```

    ```c title="main.c"
    #include "pico/stdlib.h"
    #include "hardware/pio.h"
    #include "square.pio.h"          // generated by pioasm

    int main(void) {
        const uint PIN = 15;
        PIO pio = pio0;
        uint offset = pio_add_program(pio, &square_program);
        uint sm = pio_claim_unused_sm(pio, true);

        pio_sm_config c = square_program_get_default_config(offset);
        sm_config_set_set_pins(&c, PIN, 1);
        sm_config_set_clkdiv(&c, 62.5f);            // 125 MHz / 62.5 = 2 MHz -> 1 MHz out
        pio_gpio_init(pio, PIN);
        pio_sm_set_consecutive_pindirs(pio, sm, PIN, 1, true);
        pio_sm_init(pio, sm, offset, &c);
        pio_sm_set_enabled(pio, sm, true);

        while (true) tight_loop_contents();         // CPU is free
    }
    ```

    ```cmake title="CMakeLists.txt (additions)"
    pico_generate_pio_header(app ${CMAKE_CURRENT_LIST_DIR}/square.pio)
    target_link_libraries(app pico_stdlib hardware_pio)
    ```

## A real protocol: UART transmit in four instructions

This is the transmitter from the official MicroPython examples. It shows the key PIO ideas: `pull` takes a byte from the CPU, `out` shifts it onto the pin one bit at a time, side-set drives the start and stop bits, and delays stretch each bit to exactly 8 cycles.

```python
import rp2
from machine import Pin

@rp2.asm_pio(sideset_init=rp2.PIO.OUT_HIGH, out_init=rp2.PIO.OUT_HIGH,
             out_shiftdir=rp2.PIO.SHIFT_RIGHT)
def uart_tx():
    pull()                                  # wait for a byte from the CPU
    set(x, 7)            .side(0)    [7]    # start bit (line low), 8 cycles
    label("bitloop")
    out(pins, 1)                     [6]    # one data bit, LSB first, 8 cycles
    jmp(x_dec, "bitloop")
    nop()                .side(1)    [6]    # stop bit (line high)

sm = rp2.StateMachine(0, uart_tx, freq=8 * 115200,
                      sideset_base=Pin(4), out_base=Pin(4))
sm.active(1)

for c in b"hello from PIO\r\n":
    sm.put(c)                               # blocks only if the 4-word FIFO is full
```

Eight state machines means up to eight extra UARTs on one Pico, on any pins. The same structure, sampling instead of driving, gives receivers and logic-analyzer style capture.

## Reading pins with PIO

Capturing data works the other way round: `in` samples pins into `ISR`, `push` sends a full word to the CPU. This program samples one pin continuously and packs 32 samples per word:

```python
@rp2.asm_pio(in_shiftdir=rp2.PIO.SHIFT_LEFT, autopush=True, push_thresh=32)
def sampler():
    in_(pins, 1)                            # 1 sample per cycle, autopush every 32

sm = rp2.StateMachine(1, sampler, freq=1_000_000, in_base=Pin(6))
sm.active(1)
words = [sm.get() for _ in range(32)]       # 1024 samples at 1 MHz
```

At high sample rates Python cannot empty the FIFO fast enough. In C, connect the state machine's RX FIFO to a DMA channel and capture straight into a RAM buffer; the `pico-examples/pio/logic_analyser` example does exactly that.

## Where to learn more

- *RP2040 Datasheet*, chapter 3 (PIO): the full instruction reference with timing diagrams.
- `pico-examples/pio/`: WS2812, I2C, SPI, UART RX/TX, quadrature encoder, logic analyser, and more.
- MicroPython's `rp2` module docs and `examples/rp2/` in the MicroPython repository.
