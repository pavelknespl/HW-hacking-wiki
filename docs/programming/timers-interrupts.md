# Timers, Interrupts and Timing

A program that polls everything in one loop with `delay()` calls works until it has to do two things at once. Then it starts missing button presses, dropping UART bytes, and drifting in time. The fixes are the same on every MCU: **non-blocking loops** for slow tasks, **interrupts** for events that must not be missed, and **hardware timers** for anything periodic.

## Non-blocking loops

Instead of sleeping, remember when something last happened and check whether enough time has passed. Several tasks can then share one loop.

=== "MicroPython"

    ```python
    import time
    from machine import Pin

    led = Pin("LED", Pin.OUT)
    last_blink = last_report = time.ticks_ms()

    while True:
        now = time.ticks_ms()
        if time.ticks_diff(now, last_blink) >= 250:
            led.toggle()
            last_blink = now
        if time.ticks_diff(now, last_report) >= 1000:
            print("still alive")
            last_report = now
        # other work here runs without waiting
    ```

    Always use `ticks_diff()` instead of subtracting: the tick counter wraps around and `ticks_diff` handles it.

=== "Arduino"

    ```cpp
    unsigned long lastBlink = 0, lastReport = 0;

    void loop() {
      unsigned long now = millis();
      if (now - lastBlink >= 250) {         // unsigned subtraction survives the wrap
        digitalWrite(LED_BUILTIN, !digitalRead(LED_BUILTIN));
        lastBlink = now;
      }
      if (now - lastReport >= 1000) {
        Serial.println("still alive");
        lastReport = now;
      }
    }
    ```

## Pin interrupts

A pin interrupt calls a function the moment an edge arrives, even if the main loop is busy. The function is an **interrupt service routine** (ISR), and it interrupts whatever the CPU was doing.

=== "MicroPython"

    ```python
    from machine import Pin
    import micropython

    micropython.alloc_emergency_exception_buf(100)   # readable errors from ISRs

    presses = 0

    def on_press(pin):
        global presses
        presses += 1

    btn = Pin(14, Pin.IN, Pin.PULL_UP)
    btn.irq(trigger=Pin.IRQ_FALLING, handler=on_press)

    while True:
        if presses:
            print("pressed", presses)
            presses = 0
    ```

    Use `Pin.IRQ_RISING`, `Pin.IRQ_FALLING`, or both OR-ed together. To run heavier code in response, call `micropython.schedule(func, arg)` from the handler; `func` then runs soon after, outside the interrupt.

=== "Arduino"

    ```cpp
    volatile uint32_t presses = 0;

    void IRAM_ATTR onPress() {        // IRAM_ATTR: keep the ISR in RAM (ESP32)
      presses++;
    }

    void setup() {
      Serial.begin(115200);
      pinMode(14, INPUT_PULLUP);
      attachInterrupt(digitalPinToInterrupt(14), onPress, FALLING);
    }

    void loop() {
      noInterrupts();
      uint32_t n = presses;
      presses = 0;
      interrupts();
      if (n) Serial.println(n);
    }
    ```

    `IRAM_ATTR` is an ESP32 attribute; on other cores remove it or `#define IRAM_ATTR` to nothing. On the Uno only pins 2 and 3 support `attachInterrupt`.

=== "Pico SDK"

    ```c
    #include "pico/stdlib.h"

    static volatile uint32_t presses;

    static void on_gpio(uint gpio, uint32_t events) {
        if (gpio == 14 && (events & GPIO_IRQ_EDGE_FALL)) presses++;
    }

    int main(void) {
        stdio_init_all();
        gpio_init(14);
        gpio_pull_up(14);
        gpio_set_irq_enabled_with_callback(14, GPIO_IRQ_EDGE_FALL, true, &on_gpio);
        while (true) {
            if (presses) { printf("%lu\n", (unsigned long)presses); presses = 0; }
            sleep_ms(10);
        }
    }
    ```

    There is one GPIO callback per core. Enable more pins with `gpio_set_irq_enabled(pin, events, true)` and check `gpio` inside the callback.

=== "ESP-IDF"

    ```c
    #include "driver/gpio.h"
    #include "freertos/FreeRTOS.h"
    #include "freertos/queue.h"

    static QueueHandle_t evq;

    static void IRAM_ATTR on_gpio(void *arg) {
        uint32_t pin = (uint32_t)arg;
        xQueueSendFromISR(evq, &pin, NULL);     // hand the event to a task
    }

    void app_main(void) {
        evq = xQueueCreate(10, sizeof(uint32_t));
        gpio_config_t io = {
            .pin_bit_mask = 1ULL << GPIO_NUM_14,
            .mode = GPIO_MODE_INPUT,
            .pull_up_en = GPIO_PULLUP_ENABLE,
            .intr_type = GPIO_INTR_NEGEDGE,
        };
        gpio_config(&io);
        gpio_install_isr_service(0);
        gpio_isr_handler_add(GPIO_NUM_14, on_gpio, (void *)GPIO_NUM_14);

        uint32_t pin;
        while (xQueueReceive(evq, &pin, portMAX_DELAY)) {
            printf("edge on GPIO%lu\n", (unsigned long)pin);
        }
    }
    ```

=== "STM32 HAL"

    ```c
    // CubeMX: PC13 as GPIO_EXTI13, falling edge, enable "EXTI line[15:10]" in NVIC.
    volatile uint32_t presses;

    void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {   // overrides the weak HAL symbol
        if (GPIO_Pin == GPIO_PIN_13) presses++;
    }
    ```

## Rules for interrupt handlers

- **Keep them short.** Set a flag, increment a counter, push to a queue. Do the real work in the main loop or a task.
- **Mark shared variables `volatile`** in C so the compiler re-reads them instead of caching them in a register.
- **Protect multi-byte reads.** A 32-bit counter on an 8-bit AVR is read in four steps, and an interrupt can fire in between. Copy it with interrupts disabled (`noInterrupts()`, `machine.disable_irq()`, `save_and_disable_interrupts()`).
- **No blocking calls.** No `delay`, no `Serial.print`, no `printf`, no waiting for a bus inside an ISR.
- **MicroPython hard IRQs cannot allocate memory.** No creating lists, strings, or floats. Ports where IRQs are "soft" by default (rp2, esp32 pin IRQs) are more forgiving but have more latency. Pass `hard=True` when you need low latency and follow the rule strictly.

## Periodic hardware timers

A timer interrupt fires at a fixed interval independent of what the main loop is doing. Use it for sampling a sensor at an exact rate, refreshing a display, or a watchdog-style heartbeat.

=== "MicroPython"

    ```python
    from machine import Timer, Pin

    led = Pin("LED", Pin.OUT)
    tim = Timer(mode=Timer.PERIODIC, freq=2, callback=lambda t: led.toggle())   # rp2
    # ESP32 needs a hardware timer id: Timer(0, mode=Timer.PERIODIC, period=500, callback=...)
    # one-shot: Timer(mode=Timer.ONE_SHOT, period=1000, callback=...)

    tim.deinit()     # stop it
    ```

=== "Arduino"

    ```cpp
    // ESP32, arduino-esp32 core 3.x
    hw_timer_t *timer;
    volatile bool tick;

    void IRAM_ATTR onTimer() { tick = true; }

    void setup() {
      timer = timerBegin(1000000);              // 1 MHz timer clock
      timerAttachInterrupt(timer, &onTimer);
      timerAlarm(timer, 500000, true, 0);       // every 500000 ticks = 0.5 s, autoreload
    }

    void loop() {
      if (tick) { tick = false; /* periodic work */ }
    }
    ```

    Timer APIs differ between Arduino cores. On AVR, the `TimerOne` library gives `Timer1.initialize(500000); Timer1.attachInterrupt(cb);`. On RP2040 the Pico SDK functions in the next tab are available directly.

=== "Pico SDK"

    ```c
    static bool on_tick(struct repeating_timer *t) {
        gpio_xor_mask(1u << 25);
        return true;                  // keep repeating
    }

    struct repeating_timer timer;
    add_repeating_timer_ms(500, on_tick, NULL, &timer);
    // a negative period means "start to start", positive means "end to start"

    // one-shot alarm:
    // add_alarm_in_ms(1000, callback, user_data, true);
    ```

=== "ESP-IDF"

    ```c
    #include "esp_timer.h"

    static void on_tick(void *arg) {
        // runs in the esp_timer task, so short normal code is fine
    }

    const esp_timer_create_args_t args = { .callback = on_tick, .name = "tick" };
    esp_timer_handle_t t;
    esp_timer_create(&args, &t);
    esp_timer_start_periodic(t, 500000);      // microseconds
    ```

=== "STM32 HAL"

    ```c
    // CubeMX: TIM3, Prescaler 8399, Period 4999 with an 84 MHz clock -> 2 Hz,
    // enable TIM3 global interrupt in NVIC.
    extern TIM_HandleTypeDef htim3;

    HAL_TIM_Base_Start_IT(&htim3);            // in main(), after init

    void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
        if (htim->Instance == TIM3) HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
    }
    ```

## Measuring time and pulses

| Need | MicroPython | Arduino | Pico SDK | ESP-IDF |
| --- | --- | --- | --- | --- |
| Milliseconds since boot | `time.ticks_ms()` | `millis()` | `to_ms_since_boot(get_absolute_time())` | `esp_timer_get_time() / 1000` |
| Microseconds | `time.ticks_us()` | `micros()` | `time_us_64()` | `esp_timer_get_time()` |
| Busy-wait µs | `time.sleep_us(n)` | `delayMicroseconds(n)` | `busy_wait_us(n)` | `esp_rom_delay_us(n)` |
| Pulse width on a pin | `machine.time_pulse_us(pin, 1)` | `pulseIn(pin, HIGH)` | GPIO IRQ + `time_us_64()` | GPIO ISR + `esp_timer_get_time()` |

Example: an HC-SR04 ultrasonic distance sensor, which answers a 10 µs trigger pulse with an echo pulse as long as the sound's round trip:

=== "MicroPython"

    ```python
    from machine import Pin, time_pulse_us
    import time

    trig = Pin(2, Pin.OUT, value=0)
    echo = Pin(3, Pin.IN)

    def distance_cm():
        trig.value(1); time.sleep_us(10); trig.value(0)
        t = time_pulse_us(echo, 1, 30_000)      # timeout 30 ms
        return t / 58 if t > 0 else None        # 58 us per cm, round trip

    print(distance_cm())
    ```

=== "Arduino"

    ```cpp
    float distanceCm() {
      digitalWrite(2, HIGH); delayMicroseconds(10); digitalWrite(2, LOW);
      unsigned long t = pulseIn(3, HIGH, 30000);
      return t ? t / 58.0 : -1;
    }
    ```

The HC-SR04 is a 5 V part: power it from 5 V and divide its echo output down before it reaches a 3.3 V pin.

### How precise is it?

| Platform | Realistic timing resolution |
| --- | --- |
| MicroPython, plain code | about 10 to 50 µs, with jitter from garbage collection |
| MicroPython, `@viper` or hard IRQ | a few µs |
| C with interrupts | under 1 µs, jitter from other interrupts |
| C with hardware timers / PWM / PIO | one clock cycle (8 ns at 125 MHz on the RP2040) |

When you need repeatable sub-microsecond timing, let a hardware peripheral generate the signal (a timer output compare, PWM, or the RP2040's [PIO](bit-banging-pio.md)) instead of toggling the pin from code.
