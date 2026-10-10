# 1-Wire and Addressable LEDs

Two popular device families use a single data line with strict timing instead of a clocked bus. **1-Wire** (Dallas/Maxim) carries data, and optionally power, over one wire to many addressable devices, most famously the DS18B20 temperature sensor. **WS2812** LEDs (sold as NeoPixels) chain hundreds of RGB LEDs on one line. Both are easy to use from a library and instructive to understand, because their timing is the kind of thing [PIO](bit-banging-pio.md) and hardware peripherals exist to generate.

## 1-Wire: DS18B20 temperature sensors

Wiring: `VDD` to 3.3 V, `GND` to ground, `DQ` to a GPIO **with a 4.7 kΩ pull-up to 3.3 V**. Any number of sensors can share the same three wires; each has a unique 64-bit ROM code that serves as its address. See the [1-Wire protocol page](../protocols/one-wire.md) for how reset pulses, time slots, and ROM search work on the wire.

=== "MicroPython"

    ```python
    from machine import Pin
    import onewire, ds18x20, time

    ow = onewire.OneWire(Pin(22))
    ds = ds18x20.DS18X20(ow)

    roms = ds.scan()                         # list of 8-byte ROM codes
    print("found", [r.hex() for r in roms])

    while True:
        ds.convert_temp()                    # all sensors start a conversion
        time.sleep_ms(750)                   # 12-bit conversion time
        for rom in roms:
            print(rom.hex(), ds.read_temp(rom), "C")
    ```

    `onewire` and `ds18x20` are included in the rp2 and esp32 firmware. On other ports install them with `mpremote mip install onewire ds18x20`.

=== "Arduino"

    ```cpp
    // libraries: "OneWire" and "DallasTemperature"
    #include <OneWire.h>
    #include <DallasTemperature.h>

    OneWire ow(22);
    DallasTemperature sensors(&ow);

    void setup() {
      Serial.begin(115200);
      sensors.begin();
      Serial.print("sensors: ");
      Serial.println(sensors.getDeviceCount());
    }

    void loop() {
      sensors.requestTemperatures();           // blocks for the conversion
      for (int i = 0; i < sensors.getDeviceCount(); i++) {
        Serial.print(i);
        Serial.print(": ");
        Serial.println(sensors.getTempCByIndex(i));
      }
      delay(1000);
    }
    ```

=== "Raw 1-Wire (MicroPython)"

    The same reading without the `ds18x20` helper, showing the actual commands:

    ```python
    ow = onewire.OneWire(Pin(22))
    ow.reset()
    ow.writebyte(0xCC)          # SKIP ROM: talk to every device (fine with one sensor)
    ow.writebyte(0x44)          # CONVERT T
    time.sleep_ms(750)
    ow.reset()
    ow.writebyte(0xCC)
    ow.writebyte(0xBE)          # READ SCRATCHPAD
    sp = bytearray(9)
    ow.readinto(sp)
    if ow.crc8(sp) == 0:        # CRC over all 9 bytes is 0 when valid
        raw = sp[0] | (sp[1] << 8)
        if raw & 0x8000:
            raw -= 0x10000
        print(raw / 16, "C")
    ```

!!! note "Readings of 85 °C or -127 °C"
    85 °C is the power-on value of the scratchpad, so you read before a conversion finished or the sensor reset (often parasite power trouble). -127 °C (`DEVICE_DISCONNECTED_C`) means no answer at all: check the pull-up and wiring.

## WS2812 / NeoPixel LEDs

Each LED has a tiny controller. It takes the first 24 bits of the data stream for itself (green, red, blue, 8 bits each) and passes the rest down the chain. A bit is encoded in the **width of a high pulse** at 800 kHz, so timing must be accurate to about 150 ns. A reset gap of more than about 50 µs (280 µs on newer parts) latches the colours.

Wiring:

- `5V` from a supply that can handle the load (up to 60 mA per LED at full white; 60 LEDs can draw 3.6 A).
- `GND` shared with the MCU.
- `DIN` from a GPIO through a 330 Ω resistor. 3.3 V data usually works at 5 V supply, but a 74AHCT125 level shifter makes it reliable.
- A 470 µF or larger capacitor across 5 V and GND near the strip.

=== "MicroPython"

    ```python
    from machine import Pin
    from neopixel import NeoPixel
    import time

    np = NeoPixel(Pin(16), 8)       # data pin, number of LEDs

    np[0] = (255, 0, 0)             # (R, G, B); the driver handles GRB order
    np[1] = (0, 64, 0)
    np.write()                      # nothing changes until write()

    np.fill((0, 0, 0)); np.write()  # all off

    # rainbow chase
    def wheel(pos):
        pos %= 255
        if pos < 85:  return (255 - pos * 3, pos * 3, 0)
        if pos < 170: pos -= 85;  return (0, 255 - pos * 3, pos * 3)
        pos -= 170;   return (pos * 3, 0, 255 - pos * 3)

    for j in range(255):
        for i in range(len(np)):
            np[i] = wheel(i * 32 + j)
        np.write()
        time.sleep_ms(20)
    ```

    On the rp2 port the driver uses a PIO state machine for timing; on ESP32 it uses the RMT peripheral. Both leave the CPU free.

=== "Arduino"

    ```cpp
    // library: "Adafruit NeoPixel"
    #include <Adafruit_NeoPixel.h>

    Adafruit_NeoPixel strip(8, 16, NEO_GRB + NEO_KHZ800);   // count, pin, type

    void setup() {
      strip.begin();
      strip.setBrightness(50);        // 0..255, protects your eyes and power supply
      strip.show();                   // all off
    }

    void loop() {
      for (uint16_t j = 0; j < 65536 / 256; j++) {
        for (int i = 0; i < strip.numPixels(); i++) {
          uint16_t hue = j * 256 + i * 65536 / strip.numPixels();
          strip.setPixelColor(i, strip.gamma32(strip.ColorHSV(hue)));
        }
        strip.show();
        delay(20);
      }
    }
    ```

    FastLED is the popular alternative with more effects and faster maths.

=== "Pico SDK / ESP-IDF"

    The C SDKs do not have a one-line driver, but both ship a ready example:

    - **Pico SDK:** `pico-examples/pio/ws2812` contains a PIO program (`ws2812.pio`) and `put_pixel(urgb)` helper. The [PIO page](bit-banging-pio.md) explains how such a program works.
    - **ESP-IDF:** add the component with `idf.py add-dependency "espressif/led_strip"`, then `led_strip_new_rmt_device()`, `led_strip_set_pixel(strip, i, r, g, b)`, and `led_strip_refresh(strip)`.

## Why these protocols are timing-sensitive

Bit-banging WS2812 from plain code works only on fast MCUs with interrupts disabled, because a single interrupt in the middle of a frame stretches a pulse and corrupts every LED after it. That is why the good drivers hand the job to a peripheral: PIO on the RP2040, RMT on the ESP32, or SPI/DMA tricks on STM32. 1-Wire is slower (60 µs time slots) and tolerates software timing, but still needs interrupts off during each slot.
