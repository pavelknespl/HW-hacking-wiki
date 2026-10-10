# Storage, Sleep and Watchdog

Three things separate a demo from a device that runs for months: it **remembers settings** across resets, it **sleeps** when there is nothing to do, and it **recovers by itself** when something hangs. This page covers all three.

## Files (MicroPython)

MicroPython puts a small filesystem (LittleFS or FAT) in the board's flash. Normal Python file I/O works:

```python
import os, json

with open("log.csv", "a") as f:             # append a line
    f.write("1234,21.5\n")

print(os.listdir())                          # ['boot.py', 'main.py', 'log.csv']
st = os.statvfs("/")
print("free KB:", st[0] * st[3] // 1024)

# settings as JSON
def load(default):
    try:
        with open("config.json") as f:
            return json.load(f)
    except OSError:
        return default

def save(cfg):
    with open("config.json", "w") as f:
        json.dump(cfg, f)

cfg = load({"interval": 60, "name": "node-1"})
cfg["interval"] = 30
save(cfg)
```

Copy files to and from the PC with `mpremote fs cp`, see [Getting Started with MicroPython](micropython-setup.md#the-mpremote-workflow).

## Key-value settings

For a few numbers and strings, a key-value store is simpler and more robust than files.

=== "MicroPython"

    ```python
    # ESP32: NVS, the same store ESP-IDF uses
    import esp32
    nvs = esp32.NVS("cfg")
    try:
        boots = nvs.get_i32("boots")
    except OSError:                  # key does not exist yet
        boots = 0
    nvs.set_i32("boots", boots + 1)
    nvs.commit()
    print("boot #", boots + 1)
    ```

    On other ports, use a small JSON file as shown above.

=== "Arduino"

    ```cpp
    // ESP32: Preferences (backed by NVS)
    #include <Preferences.h>
    Preferences prefs;

    void setup() {
      Serial.begin(115200);
      prefs.begin("cfg", false);                 // namespace, read-only = false
      uint32_t boots = prefs.getUInt("boots", 0) + 1;
      prefs.putUInt("boots", boots);
      prefs.end();
      Serial.printf("boot #%u\n", boots);
    }
    ```

    ```cpp
    // AVR (real EEPROM) and RP2040 / STM32 (flash-emulated EEPROM)
    #include <EEPROM.h>
    struct Config { uint32_t magic; uint16_t interval; char name[16]; };

    void setup() {
      EEPROM.begin(256);                // flash-emulated cores only, remove on AVR
      Config c;
      EEPROM.get(0, c);
      if (c.magic != 0xC0FFEE01) {      // first boot or layout changed
        c = { 0xC0FFEE01, 60, "node-1" };
        EEPROM.put(0, c);
        EEPROM.commit();                // flash-emulated cores only, remove on AVR
      }
    }
    ```

    The `magic` field is a cheap way to detect uninitialised or incompatible stored data.

=== "Pico SDK"

    The RP2040 has no EEPROM, so store data in the last sector of the flash chip. Flash is memory-mapped at `XIP_BASE` for reading; writing requires erasing a 4 KB sector first and must run with interrupts off, because code is executing from the same chip.

    ```c
    #include <string.h>
    #include "hardware/flash.h"
    #include "hardware/sync.h"

    #define CFG_OFFSET (PICO_FLASH_SIZE_BYTES - FLASH_SECTOR_SIZE)

    const uint8_t *cfg_flash = (const uint8_t *)(XIP_BASE + CFG_OFFSET);

    void cfg_save(const uint8_t *data, size_t len) {   // len <= FLASH_PAGE_SIZE (256)
        uint8_t page[FLASH_PAGE_SIZE] = { 0 };
        memcpy(page, data, len);
        uint32_t irq = save_and_disable_interrupts();
        flash_range_erase(CFG_OFFSET, FLASH_SECTOR_SIZE);
        flash_range_program(CFG_OFFSET, page, FLASH_PAGE_SIZE);
        restore_interrupts(irq);
    }
    ```

    Link `hardware_flash`. If the second core is running, use `flash_safe_execute()` (SDK 2.0+) so it is paused during the write too.

=== "ESP-IDF"

    ```c
    #include "nvs_flash.h"

    ESP_ERROR_CHECK(nvs_flash_init());
    nvs_handle_t h;
    ESP_ERROR_CHECK(nvs_open("cfg", NVS_READWRITE, &h));
    int32_t boots = 0;
    nvs_get_i32(h, "boots", &boots);           // ESP_ERR_NVS_NOT_FOUND on first boot
    nvs_set_i32(h, "boots", boots + 1);
    nvs_commit(h);
    nvs_close(h);
    ```

!!! note "Flash wear"
    Flash sectors survive roughly 10,000 to 100,000 erase cycles. Saving a setting once a second wears a sector out within days. Write only when a value actually changes, batch log lines in RAM, and let NVS / LittleFS spread writes across sectors (they do wear levelling; a raw sector write does not).

## Sleep

An ESP32 draws about 80 mA with Wi-Fi idle, 10 µA in deep sleep. On a battery, sleeping is the difference between hours and months.

| Mode | What keeps running | Wake-up | Resume |
| --- | --- | --- | --- |
| Idle / light sleep | RAM, peripherals (partly) | timer, pin, UART | continues after the sleep call |
| Deep sleep | RTC only (a few KB of RTC RAM on ESP32) | timer, pin, touch | full reboot, `main` starts again |

=== "MicroPython"

    ```python
    import machine, time

    if machine.reset_cause() == machine.DEEPSLEEP_RESET:
        print("woke from deep sleep")

    do_measurement_and_send()

    machine.lightsleep(1000)        # 1 s, then continue on the next line
    machine.deepsleep(60_000)       # 60 s, then reboot into boot.py/main.py

    # ESP32: also wake on a button (GPIO must be an RTC pin)
    # import esp32
    # esp32.wake_on_ext0(pin=machine.Pin(33, machine.Pin.IN), level=esp32.WAKEUP_ALL_LOW)
    ```

=== "Arduino"

    ```cpp
    // ESP32
    RTC_DATA_ATTR int bootCount = 0;        // survives deep sleep

    void setup() {
      Serial.begin(115200);
      bootCount++;
      Serial.printf("boot %d, cause %d\n", bootCount, esp_sleep_get_wakeup_cause());
      // ... measure, send ...
      esp_sleep_enable_timer_wakeup(60ULL * 1000000);    // microseconds
      esp_deep_sleep_start();                            // never returns
    }

    void loop() {}
    ```

    On AVR, `#include <avr/sleep.h>`, then `set_sleep_mode(SLEEP_MODE_PWR_DOWN); sleep_mode();` stops the CPU until an interrupt (pin or watchdog) wakes it.

=== "Pico SDK"

    The RP2040's low-power modes (dormant, sleep with RTC wake) live in the separate `pico-extras` repository (`pico_sleep` library). In plain SDK code, `__wfi()` halts the core until the next interrupt, which already saves power in idle loops. The RP2350 adds proper power-down support through the SDK's `hardware_powman`.

=== "ESP-IDF"

    ```c
    #include "esp_sleep.h"

    esp_sleep_enable_timer_wakeup(60ULL * 1000000);
    esp_deep_sleep_start();
    ```

Real-world current also depends on the board: power LEDs, USB-serial chips, and linear regulators often draw more than the sleeping MCU. Measure the whole board.

## Watchdog

A watchdog is a hardware timer that resets the chip unless your code "feeds" it regularly. If the program hangs (a stuck bus, a network stack deadlock, an infinite loop) the feeding stops and the device reboots itself instead of staying dead.

=== "MicroPython"

    ```python
    from machine import WDT

    wdt = WDT(timeout=5000)        # ms; cannot be stopped once started
    while True:
        do_work()
        wdt.feed()
    ```

=== "Arduino"

    ```cpp
    // AVR
    #include <avr/wdt.h>
    void setup() { wdt_enable(WDTO_2S); }
    void loop()  { doWork(); wdt_reset(); }

    // RP2040 (earlephilhower): rp2040.wdt_begin(5000); rp2040.wdt_reset();
    ```

=== "Pico SDK"

    ```c
    #include "hardware/watchdog.h"

    if (watchdog_caused_reboot()) printf("recovered from a hang\n");
    watchdog_enable(5000, true);    // 5 s, pause while a debugger halts the CPU
    while (true) {
        do_work();
        watchdog_update();
    }
    ```

=== "ESP-IDF"

    ```c
    #include "esp_task_wdt.h"

    // the task watchdog is initialised by default (menuconfig); subscribe this task
    esp_task_wdt_add(NULL);
    while (1) {
        do_work();
        esp_task_wdt_reset();
        vTaskDelay(pdMS_TO_TICKS(100));
    }
    ```

=== "STM32 HAL"

    ```c
    // CubeMX: enable IWDG, e.g. prescaler 64, reload 2500 -> about 5 s on the 32 kHz LSI
    extern IWDG_HandleTypeDef hiwdg;
    while (1) {
        do_work();
        HAL_IWDG_Refresh(&hiwdg);
    }
    ```

Feed the watchdog only from the place that proves the program is healthy, typically the end of the main loop. Feeding it from a timer interrupt defeats the purpose, because the interrupt keeps firing while the main loop is stuck.
