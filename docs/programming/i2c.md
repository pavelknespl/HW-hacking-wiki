# I2C

I2C is a two-wire bus (`SDA` data, `SCL` clock) where one controller talks to many devices, each with a 7-bit address. Sensors, EEPROMs, displays, RTCs, and power-management chips all speak it. The [I2C protocol page](../protocols/i2c.md) explains the wire-level details and how to attack a bus on a target. This page is about **driving I2C from your own microcontroller**: initialising it, finding devices, and reading and writing their registers with as little code as possible.

## Wiring

```
  MCU 3V3 ──┬────────┬──────────── device VCC
            │        │
          4.7k     4.7k        (pull-ups, often already on breakout boards)
            │        │
  MCU SDA ──┴────────┼──────────── device SDA
  MCU SCL ───────────┴──────────── device SCL
  MCU GND ──────────────────────── device GND
```

- Both lines **must** be pulled up. Most breakout boards (Adafruit, SparkFun, generic modules) include 4.7 kΩ or 10 kΩ pull-ups. A bare chip needs external ones. The MCU's internal pulls (30 to 50 kΩ) only work for short wires at 100 kHz.
- Use the default pins from the [section overview](index.md#pin-numbers-used-in-the-examples): Pico GP4/GP5, ESP32 GPIO21/22, Uno A4/A5.
- Keep the device at the same voltage as the MCU, or use a MOSFET [level shifter](../fundamentals/level-shifting.md) that handles open-drain lines.

## The three operations you will use 90% of the time

Almost every I2C device is a set of numbered **registers**. You only ever need:

1. **Scan**: which addresses respond?
2. **Write a register**: send `[address+W] [reg] [value...]`.
3. **Read a register**: send `[address+W] [reg]`, then a **repeated start**, then `[address+R]` and read `n` bytes.

The examples below use an MPU-6050 accelerometer (address `0x68`, `WHO_AM_I` register `0x75` returns `0x68`) because it is everywhere and cheap. Swap the address and register for your chip.

## Initialise and scan

=== "MicroPython"

    ```python
    from machine import I2C, Pin

    i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400_000)   # Pico I2C0
    # ESP32: I2C(0, sda=Pin(21), scl=Pin(22), freq=400_000)

    print([hex(a) for a in i2c.scan()])   # e.g. ['0x3c', '0x68']
    ```

    That is the whole setup. `scan()` probes every address from `0x08` to `0x77` and returns the ones that acknowledged.

=== "Arduino"

    ```cpp
    #include <Wire.h>

    void setup() {
      Serial.begin(115200);
      while (!Serial) {}
      Wire.begin();                    // Uno: A4/A5. ESP32: Wire.begin(21, 22);
      // RP2040 (earlephilhower): Wire.setSDA(4); Wire.setSCL(5); Wire.begin();
      Wire.setClock(400000);

      for (uint8_t addr = 0x08; addr < 0x78; addr++) {
        Wire.beginTransmission(addr);
        if (Wire.endTransmission() == 0) {   // 0 = ACK received
          Serial.print("found 0x");
          Serial.println(addr, HEX);
        }
      }
    }

    void loop() {}
    ```

=== "Pico SDK"

    ```c
    #include <stdio.h>
    #include "pico/stdlib.h"
    #include "hardware/i2c.h"

    int main(void) {
        stdio_init_all();
        sleep_ms(2000);                       // give USB serial time to connect

        i2c_init(i2c0, 400 * 1000);
        gpio_set_function(4, GPIO_FUNC_I2C);  // SDA
        gpio_set_function(5, GPIO_FUNC_I2C);  // SCL
        gpio_pull_up(4);
        gpio_pull_up(5);

        for (int addr = 0x08; addr < 0x78; addr++) {
            uint8_t dummy;
            // a 1-byte read returns PICO_ERROR_GENERIC if nobody ACKs
            if (i2c_read_blocking(i2c0, addr, &dummy, 1, false) >= 0)
                printf("found 0x%02x\n", addr);
        }
    }
    ```

    Link with `target_link_libraries(app pico_stdlib hardware_i2c)`.

=== "ESP-IDF"

    ```c
    #include <stdio.h>
    #include "driver/i2c_master.h"   // new driver, ESP-IDF 5.2+

    i2c_master_bus_handle_t bus;

    void app_main(void) {
        i2c_master_bus_config_t cfg = {
            .i2c_port = I2C_NUM_0,
            .sda_io_num = GPIO_NUM_21,
            .scl_io_num = GPIO_NUM_22,
            .clk_source = I2C_CLK_SRC_DEFAULT,
            .glitch_ignore_cnt = 7,
            .flags.enable_internal_pullup = true,
        };
        ESP_ERROR_CHECK(i2c_new_master_bus(&cfg, &bus));

        for (uint8_t addr = 0x08; addr < 0x78; addr++) {
            if (i2c_master_probe(bus, addr, 50) == ESP_OK)
                printf("found 0x%02x\n", addr);
        }
    }
    ```

=== "STM32 HAL"

    ```c
    // CubeMX: enable I2C1 (PB8 SCL, PB9 SDA), Standard or Fast mode.
    // MX_I2C1_Init() is generated and called from main().
    extern I2C_HandleTypeDef hi2c1;

    for (uint8_t addr = 0x08; addr < 0x78; addr++) {
        // HAL wants the 8-bit address: 7-bit address shifted left by one
        if (HAL_I2C_IsDeviceReady(&hi2c1, addr << 1, 1, 10) == HAL_OK)
            printf("found 0x%02x\n", addr);
    }
    ```

!!! tip "Nothing found?"
    An empty scan almost always means missing pull-ups, swapped SDA/SCL, no common ground, or an unpowered device. If the scan *hangs* or every address responds, a line is stuck low or shorted. See [bus recovery](#bus-recovery) below.

## Read a register

=== "MicroPython"

    ```python
    MPU = 0x68
    who = i2c.readfrom_mem(MPU, 0x75, 1)       # returns bytes, e.g. b'h' (0x68)
    print(hex(who[0]))

    data = i2c.readfrom_mem(MPU, 0x3B, 6)      # 6 bytes starting at 0x3B
    ```

    `readfrom_mem(addr, reg, n)` does the write-register, repeated-start, read sequence in one call. Use `readfrom_mem_into(addr, reg, buf)` with a preallocated `bytearray` inside fast loops.

=== "Arduino"

    ```cpp
    uint8_t readReg(uint8_t addr, uint8_t reg) {
      Wire.beginTransmission(addr);
      Wire.write(reg);
      Wire.endTransmission(false);        // false = repeated start, keep the bus
      Wire.requestFrom(addr, (uint8_t)1);
      return Wire.read();
    }

    // multi-byte read into a buffer, returns how many bytes arrived
    size_t readRegs(uint8_t addr, uint8_t reg, uint8_t *buf, size_t n) {
      Wire.beginTransmission(addr);
      Wire.write(reg);
      if (Wire.endTransmission(false) != 0) return 0;
      size_t got = Wire.requestFrom(addr, (uint8_t)n);
      for (size_t i = 0; i < got; i++) buf[i] = Wire.read();
      return got;
    }

    // usage
    Serial.println(readReg(0x68, 0x75), HEX);   // 68
    ```

=== "Pico SDK"

    ```c
    int read_regs(uint8_t addr, uint8_t reg, uint8_t *buf, size_t n) {
        // nostop = true leaves the bus claimed, so the read starts with a repeated start
        if (i2c_write_blocking(i2c0, addr, &reg, 1, true) < 0) return -1;
        return i2c_read_blocking(i2c0, addr, buf, n, false);
    }

    uint8_t who;
    read_regs(0x68, 0x75, &who, 1);
    printf("WHO_AM_I = 0x%02x\n", who);
    ```

=== "ESP-IDF"

    ```c
    i2c_device_config_t dev_cfg = {
        .dev_addr_length = I2C_ADDR_BIT_LEN_7,
        .device_address = 0x68,
        .scl_speed_hz = 400000,
    };
    i2c_master_dev_handle_t mpu;
    ESP_ERROR_CHECK(i2c_master_bus_add_device(bus, &dev_cfg, &mpu));

    uint8_t reg = 0x75, who;
    // write reg, repeated start, read 1 byte; -1 = wait forever
    ESP_ERROR_CHECK(i2c_master_transmit_receive(mpu, &reg, 1, &who, 1, -1));
    printf("WHO_AM_I = 0x%02x\n", who);
    ```

=== "STM32 HAL"

    ```c
    uint8_t who;
    HAL_I2C_Mem_Read(&hi2c1, 0x68 << 1, 0x75, I2C_MEMADD_SIZE_8BIT,
                     &who, 1, HAL_MAX_DELAY);
    ```

    `HAL_I2C_Mem_Read` / `HAL_I2C_Mem_Write` are the register helpers. The plain `HAL_I2C_Master_Transmit` / `HAL_I2C_Master_Receive` send raw bytes without a register address.

## Write a register

=== "MicroPython"

    ```python
    i2c.writeto_mem(0x68, 0x6B, b'\x00')       # PWR_MGMT_1 = 0: wake the MPU-6050
    i2c.writeto_mem(0x68, 0x1C, bytes([0x10])) # ACCEL_CONFIG = +/-8 g
    ```

=== "Arduino"

    ```cpp
    void writeReg(uint8_t addr, uint8_t reg, uint8_t val) {
      Wire.beginTransmission(addr);
      Wire.write(reg);
      Wire.write(val);
      Wire.endTransmission();               // true (default) = send STOP
    }

    writeReg(0x68, 0x6B, 0x00);
    ```

=== "Pico SDK"

    ```c
    void write_reg(uint8_t addr, uint8_t reg, uint8_t val) {
        uint8_t buf[2] = { reg, val };
        i2c_write_blocking(i2c0, addr, buf, 2, false);
    }

    write_reg(0x68, 0x6B, 0x00);
    ```

=== "ESP-IDF"

    ```c
    uint8_t buf[2] = { 0x6B, 0x00 };
    ESP_ERROR_CHECK(i2c_master_transmit(mpu, buf, sizeof buf, -1));
    ```

=== "STM32 HAL"

    ```c
    uint8_t val = 0x00;
    HAL_I2C_Mem_Write(&hi2c1, 0x68 << 1, 0x6B, I2C_MEMADD_SIZE_8BIT,
                      &val, 1, HAL_MAX_DELAY);
    ```

## Complete example: read an accelerometer

Putting it together. Wake the MPU-6050, then read 14 bytes (accel X/Y/Z, temperature, gyro X/Y/Z), each a big-endian signed 16-bit value.

=== "MicroPython"

    ```python
    from machine import I2C, Pin
    import struct, time

    i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400_000)
    MPU = 0x68
    i2c.writeto_mem(MPU, 0x6B, b'\x00')            # wake up

    while True:
        raw = i2c.readfrom_mem(MPU, 0x3B, 14)
        ax, ay, az, t, gx, gy, gz = struct.unpack('>7h', raw)
        print("acc g: %.2f %.2f %.2f  temp C: %.1f" %
              (ax / 16384, ay / 16384, az / 16384, t / 340 + 36.53))
        time.sleep_ms(200)
    ```

=== "Arduino"

    ```cpp
    #include <Wire.h>
    const uint8_t MPU = 0x68;

    void setup() {
      Serial.begin(115200);
      Wire.begin();
      Wire.beginTransmission(MPU);
      Wire.write(0x6B); Wire.write(0x00);           // wake up
      Wire.endTransmission();
    }

    void loop() {
      Wire.beginTransmission(MPU);
      Wire.write(0x3B);
      Wire.endTransmission(false);
      Wire.requestFrom(MPU, (uint8_t)14);
      int16_t v[7];
      for (int i = 0; i < 7; i++) v[i] = (Wire.read() << 8) | Wire.read();
      Serial.printf("acc g: %.2f %.2f %.2f  temp C: %.1f\n",
                    v[0] / 16384.0, v[1] / 16384.0, v[2] / 16384.0, v[3] / 340.0 + 36.53);
      delay(200);
    }
    ```

    `Serial.printf` exists on ESP32 and RP2040 cores. On AVR use `Serial.print` calls, and note that `%f` is not supported by AVR's `printf`.

=== "Pico SDK"

    ```c
    #include <stdio.h>
    #include "pico/stdlib.h"
    #include "hardware/i2c.h"

    #define MPU 0x68

    int main(void) {
        stdio_init_all();
        i2c_init(i2c0, 400 * 1000);
        gpio_set_function(4, GPIO_FUNC_I2C);
        gpio_set_function(5, GPIO_FUNC_I2C);
        gpio_pull_up(4); gpio_pull_up(5);

        uint8_t wake[2] = { 0x6B, 0x00 };
        i2c_write_blocking(i2c0, MPU, wake, 2, false);

        while (true) {
            uint8_t reg = 0x3B, raw[14];
            i2c_write_blocking(i2c0, MPU, &reg, 1, true);
            i2c_read_blocking(i2c0, MPU, raw, 14, false);
            int16_t v[7];
            for (int i = 0; i < 7; i++) v[i] = (raw[2 * i] << 8) | raw[2 * i + 1];
            printf("acc g: %.2f %.2f %.2f  temp C: %.1f\n",
                   v[0] / 16384.0, v[1] / 16384.0, v[2] / 16384.0, v[3] / 340.0 + 36.53);
            sleep_ms(200);
        }
    }
    ```

## Devices without registers

Some chips take **commands** instead of register addresses, then return data on a separate read. The SHT3x humidity sensor is a typical case: write the two-byte command `0x2400`, wait for the measurement, then read six bytes.

=== "MicroPython"

    ```python
    i2c.writeto(0x44, b'\x24\x00')     # start a measurement
    time.sleep_ms(20)
    raw = i2c.readfrom(0x44, 6)        # T(2) CRC(1) RH(2) CRC(1)
    t  = -45 + 175 * ((raw[0] << 8) | raw[1]) / 65535
    rh = 100 * ((raw[3] << 8) | raw[4]) / 65535
    ```

    `writeto(addr, buf, stop=False)` followed by `readfrom()` gives a repeated start when a device requires it.

=== "Arduino"

    ```cpp
    Wire.beginTransmission(0x44);
    Wire.write(0x24); Wire.write(0x00);
    Wire.endTransmission();
    delay(20);
    Wire.requestFrom((uint8_t)0x44, (uint8_t)6);
    uint8_t raw[6];
    for (int i = 0; i < 6; i++) raw[i] = Wire.read();
    ```

=== "Pico SDK"

    ```c
    uint8_t cmd[2] = { 0x24, 0x00 }, raw[6];
    i2c_write_blocking(i2c0, 0x44, cmd, 2, false);
    sleep_ms(20);
    i2c_read_blocking(i2c0, 0x44, raw, 6, false);
    ```

=== "ESP-IDF"

    ```c
    // sht: device handle added with i2c_master_bus_add_device(), address 0x44
    uint8_t cmd[2] = { 0x24, 0x00 }, raw[6];
    i2c_master_transmit(sht, cmd, 2, -1);
    vTaskDelay(pdMS_TO_TICKS(20));
    i2c_master_receive(sht, raw, 6, -1);
    ```

=== "STM32 HAL"

    ```c
    uint8_t cmd[2] = { 0x24, 0x00 }, raw[6];
    HAL_I2C_Master_Transmit(&hi2c1, 0x44 << 1, cmd, 2, HAL_MAX_DELAY);
    HAL_Delay(20);
    HAL_I2C_Master_Receive(&hi2c1, 0x44 << 1, raw, 6, HAL_MAX_DELAY);
    ```

## EEPROMs: 16-bit addresses, pages, and write cycles

24Cxx EEPROMs (address `0x50` to `0x57`) are the I2C chip you are most likely to dump or modify on a target. Three details matter:

- **Address width.** Up to 24C16 use a single address byte (larger ones in that range borrow address pins as high bits). From **24C32 upwards the memory address is two bytes**, high byte first.
- **Page writes.** A write may cover at most one page (8 bytes on 24C02, 32 on 24C32/64, 64 on 24C256) and must not cross a page boundary, otherwise it wraps within the page.
- **Write cycle.** After a write the chip is busy for up to 5 ms and does not ACK. Wait, or poll until it ACKs again.

Reads have no page limit: a single read can stream the whole chip.

=== "MicroPython"

    ```python
    import time
    EE = 0x50
    PAGE = 64                                       # 24C256

    def ee_read(mem, n):
        return i2c.readfrom_mem(EE, mem, n, addrsize=16)

    def ee_write(mem, data):
        i = 0
        while i < len(data):
            chunk = min(PAGE - (mem % PAGE), len(data) - i)   # stop at page boundary
            i2c.writeto_mem(EE, mem, data[i:i + chunk], addrsize=16)
            while True:                                       # ACK polling
                try:
                    i2c.writeto(EE, b'')
                    break
                except OSError:
                    time.sleep_ms(1)
            mem += chunk
            i += chunk

    ee_write(0x0000, b'hello from micropython')
    print(ee_read(0x0000, 22))
    ```

    For a one-byte-address chip such as a 24C02, drop `addrsize=16` (8 is the default) and set `PAGE = 8`.

=== "Arduino"

    ```cpp
    const uint8_t EE = 0x50;

    void eeRead(uint16_t mem, uint8_t *buf, uint8_t n) {   // n <= 32 on AVR
      Wire.beginTransmission(EE);
      Wire.write(mem >> 8);
      Wire.write(mem & 0xFF);
      Wire.endTransmission(false);
      Wire.requestFrom(EE, n);
      for (uint8_t i = 0; i < n; i++) buf[i] = Wire.read();
    }

    void eeWriteByte(uint16_t mem, uint8_t val) {
      Wire.beginTransmission(EE);
      Wire.write(mem >> 8);
      Wire.write(mem & 0xFF);
      Wire.write(val);
      Wire.endTransmission();
      do {                                       // ACK polling
        Wire.beginTransmission(EE);
      } while (Wire.endTransmission() != 0);
    }
    ```

    The AVR `Wire` library has a **32-byte buffer**, which includes the two address bytes on writes. Read in chunks of 32 and write at most 30 data bytes per transaction.

=== "Pico SDK"

    ```c
    #include <string.h>
    #define EE 0x50

    int ee_read(uint16_t mem, uint8_t *buf, size_t n) {
        uint8_t a[2] = { mem >> 8, mem & 0xFF };
        i2c_write_blocking(i2c0, EE, a, 2, true);
        return i2c_read_blocking(i2c0, EE, buf, n, false);   // n can be the whole chip
    }

    void ee_write_page(uint16_t mem, const uint8_t *data, size_t n) {  // n <= page, no crossing
        uint8_t buf[2 + 64];
        buf[0] = mem >> 8; buf[1] = mem & 0xFF;
        memcpy(buf + 2, data, n);
        i2c_write_blocking(i2c0, EE, buf, n + 2, false);
        uint8_t dummy;
        while (i2c_read_blocking(i2c0, EE, &dummy, 1, false) < 0) { }  // ACK polling
    }
    ```

=== "STM32 HAL"

    ```c
    uint8_t buf[64];
    HAL_I2C_Mem_Read(&hi2c1, 0x50 << 1, 0x0000, I2C_MEMADD_SIZE_16BIT,
                     buf, sizeof buf, HAL_MAX_DELAY);

    HAL_I2C_Mem_Write(&hi2c1, 0x50 << 1, 0x0000, I2C_MEMADD_SIZE_16BIT,
                      (uint8_t *)"hello", 5, HAL_MAX_DELAY);
    while (HAL_I2C_IsDeviceReady(&hi2c1, 0x50 << 1, 1, 10) != HAL_OK) { }
    ```

## Acting as an I2C target (device)

Your MCU can also *be* the device. On a target board this lets you replace a configuration EEPROM with something you control and watch what the SoC reads, or feed it modified values. Put the MCU on the bus at the EEPROM's address (with the real chip removed or its address pins changed).

=== "MicroPython"

    ```python
    from machine import I2CTarget, Pin   # MicroPython 1.26+

    mem = bytearray(256)                 # emulated 24C02 contents
    mem[0:6] = b'\xde\xad\xbe\xef\x00\x01'
    t = I2CTarget(0, addr=0x50, mem=mem, scl=Pin(5), sda=Pin(4))
    # the controller can now read and write mem; it runs in the background
    ```

    With `mem` set, MicroPython handles the "set address pointer, then read or write" behaviour of a memory device for you. `mem_addrsize=16` emulates a larger EEPROM.

=== "Arduino"

    ```cpp
    #include <Wire.h>

    uint8_t mem[256] = { 0xDE, 0xAD, 0xBE, 0xEF };
    volatile uint8_t ptr = 0;

    void onReceive(int n) {            // controller wrote: [ptr] [data...]
      if (n < 1) return;
      ptr = Wire.read();
      while (--n > 0) mem[ptr++] = Wire.read();
    }

    void onRequest() {                 // controller reads from ptr
      for (uint8_t i = 0; i < 32; i++) Wire.write(mem[(uint8_t)(ptr + i)]);
    }

    void setup() {
      Wire.begin(0x50);                // passing an address = target mode
      Wire.onReceive(onReceive);
      Wire.onRequest(onRequest);
    }

    void loop() {}
    ```

    The callbacks run in interrupt context, so keep them short and never use `Serial` or `delay` inside them.

=== "Pico SDK"

    ```c
    #include "pico/i2c_slave.h"     // link pico_i2c_slave

    static uint8_t mem[256];
    static uint8_t ptr;
    static bool ptr_set;

    static void handler(i2c_inst_t *i2c, i2c_slave_event_t ev) {
        switch (ev) {
        case I2C_SLAVE_RECEIVE:      // controller wrote a byte
            if (!ptr_set) { ptr = i2c_read_byte_raw(i2c); ptr_set = true; }
            else mem[ptr++] = i2c_read_byte_raw(i2c);
            break;
        case I2C_SLAVE_REQUEST:      // controller wants a byte
            i2c_write_byte_raw(i2c, mem[ptr++]);
            break;
        case I2C_SLAVE_FINISH:       // stop or repeated start
            ptr_set = false;
            break;
        default: break;
        }
    }

    // in main():
    gpio_set_function(4, GPIO_FUNC_I2C); gpio_set_function(5, GPIO_FUNC_I2C);
    gpio_pull_up(4); gpio_pull_up(5);
    i2c_init(i2c0, 100 * 1000);
    i2c_slave_init(i2c0, 0x50, &handler);
    ```

## Software I2C

Hardware I2C peripherals are tied to certain pins. When those are taken, or you need a second bus, use a bit-banged implementation on any two pins:

```python
from machine import SoftI2C, Pin
i2c = SoftI2C(sda=Pin(10), scl=Pin(11), freq=100_000)   # same API as I2C
```

In C, the Arduino cores and most SDKs ship a software I2C option, or you can toggle two open-drain GPIOs yourself.

## Errors and timeouts

| Framework | No ACK (wrong address, device absent) | Other |
| --- | --- | --- |
| MicroPython | `OSError: [Errno 19] ENODEV` (or `EIO`) | `ETIMEDOUT` (110) if SCL is held low |
| Arduino | `endTransmission()` returns 2 (address NACK) or 3 (data NACK) | 4 other error, 5 timeout |
| Pico SDK | return value `PICO_ERROR_GENERIC` | use `i2c_*_timeout_us()` variants to avoid hanging |
| ESP-IDF | `ESP_ERR_INVALID_STATE` / `ESP_FAIL` | `ESP_ERR_TIMEOUT` |
| STM32 HAL | `HAL_ERROR`, `HAL_I2C_GetError()` = `HAL_I2C_ERROR_AF` | `HAL_TIMEOUT` |

In MicroPython, wrap bus calls when the device may be absent:

```python
try:
    v = i2c.readfrom_mem(0x68, 0x75, 1)[0]
except OSError as e:
    print("I2C error", e)
```

## Bus recovery

If a device was reset in the middle of a read, it may still be holding `SDA` low, waiting for clocks that never came. Every following transaction fails. The fix from the I2C specification is to clock `SCL` up to nine times until the device lets go, then send a STOP:

```python
from machine import Pin
import time

def i2c_recover(sda_n, scl_n):
    scl = Pin(scl_n, Pin.OPEN_DRAIN, value=1)
    sda = Pin(sda_n, Pin.IN, Pin.PULL_UP)
    for _ in range(9):
        if sda.value():
            break
        scl.value(0); time.sleep_us(5)
        scl.value(1); time.sleep_us(5)
    sda = Pin(sda_n, Pin.OPEN_DRAIN, value=0)   # STOP: SDA rises while SCL is high
    time.sleep_us(5)
    sda.value(1)

i2c_recover(4, 5)
i2c = I2C(0, sda=Pin(4), scl=Pin(5))            # re-init the peripheral afterwards
```

## Common pitfalls

- **7-bit vs 8-bit addresses.** Datasheets sometimes give the address already shifted with the R/W bit (`0xD0` for the MPU-6050 instead of `0x68`). MicroPython, Arduino, Pico SDK, and ESP-IDF want the 7-bit form. STM32 HAL wants the shifted form (`addr << 1`).
- **Missing repeated start.** Some devices reset their register pointer on STOP. Use `endTransmission(false)` in Arduino and `nostop = true` in the Pico SDK between the register write and the read.
- **Clock stretching.** Slow devices hold `SCL` low while they work. Most hardware controllers handle it. Bit-banged drivers must read back `SCL` before continuing.
- **Bus capacitance.** Long wires and many devices slow the edges. Drop to 100 kHz, use stronger (smaller) pull-ups like 2.2 kΩ, or shorten the wiring.
- **Two controllers.** If you attach to a live target bus while its SoC is also a controller, transactions collide. Hold the SoC in reset or [sniff passively](../protocols/i2c.md#sniffing-traffic) instead.

## Under the hood: AVR TWI registers

What `Wire` does on an ATmega328P, stripped to the bone. Reading this once makes any MCU's I2C chapter readable:

```c
#include <avr/io.h>

static void twi_wait(void) { while (!(TWCR & _BV(TWINT))) { } }

void twi_init(void) {
    TWSR = 0;                         // prescaler 1
    TWBR = 72;                        // SCL = 16 MHz / (16 + 2 * 72) = 100 kHz
    TWCR = _BV(TWEN);
}
void twi_start(void) { TWCR = _BV(TWINT) | _BV(TWSTA) | _BV(TWEN); twi_wait(); }
void twi_stop(void)  { TWCR = _BV(TWINT) | _BV(TWSTO) | _BV(TWEN); }
void twi_write(uint8_t b) { TWDR = b; TWCR = _BV(TWINT) | _BV(TWEN); twi_wait(); }
uint8_t twi_read(uint8_t ack) {
    TWCR = _BV(TWINT) | _BV(TWEN) | (ack ? _BV(TWEA) : 0);
    twi_wait();
    return TWDR;
}

uint8_t read_reg(uint8_t addr, uint8_t reg) {
    twi_start();  twi_write(addr << 1);         // address + W
    twi_write(reg);
    twi_start();  twi_write((addr << 1) | 1);   // repeated start, address + R
    uint8_t v = twi_read(0);                    // NACK the last byte
    twi_stop();
    return v;
}
```

A production driver also checks `TWSR & 0xF8` after each step against the status codes in the datasheet (for example `0x18` means "address + W sent, ACK received").
