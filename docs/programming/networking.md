# Wi-Fi and Networking

The ESP32 family and the Pico W / Pico 2 W have Wi-Fi on board. That turns a sensor node into something you can reach from a browser, push data to a server, or control over MQTT. The examples below go from "connect" to "serve a web page" with the least code each framework allows.

!!! warning "Keep credentials out of your code repository"
    Put the SSID and password in a separate file (`secrets.py`, `secrets.h`) and add it to `.gitignore`. Firmware is easy to dump, so anything compiled into a device should be treated as readable by whoever holds it. That is a point this wiki makes from the other side in [Firmware Analysis](../attacks/firmware-analysis.md).

## Connect to a network

=== "MicroPython"

    ```python
    import network, time
    from secrets import SSID, PASSWORD      # secrets.py on the board

    wlan = network.WLAN(network.STA_IF)     # station (client) mode
    wlan.active(True)
    wlan.connect(SSID, PASSWORD)

    for _ in range(20):
        if wlan.isconnected():
            break
        time.sleep(0.5)

    print("IP:", wlan.ifconfig()[0])        # newer firmware: wlan.ipconfig("addr4")
    ```

    Put this in `boot.py` so the board is online before `main.py` starts. For an access point instead, use `network.WLAN(network.AP_IF)` and `ap.config(ssid="pico", password="12345678")`.

=== "Arduino"

    ```cpp
    #include <WiFi.h>              // ESP32 and RP2040 (Pico W) cores
    #include "secrets.h"          // #define SSID "..."  #define PASSWORD "..."

    void setup() {
      Serial.begin(115200);
      WiFi.mode(WIFI_STA);
      WiFi.begin(SSID, PASSWORD);
      while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print('.');
      }
      Serial.print("\nIP: ");
      Serial.println(WiFi.localIP());
    }

    void loop() {}
    ```

=== "Pico SDK"

    ```c
    #include "pico/stdlib.h"
    #include "pico/cyw43_arch.h"

    int main(void) {
        stdio_init_all();
        if (cyw43_arch_init()) return 1;
        cyw43_arch_enable_sta_mode();
        if (cyw43_arch_wifi_connect_timeout_ms(SSID, PASSWORD,
                CYW43_AUTH_WPA2_AES_PSK, 30000)) {
            printf("connect failed\n");
            return 1;
        }
        printf("connected\n");
        while (true) sleep_ms(1000);
    }
    ```

    Build with `-DPICO_BOARD=pico_w`, link `pico_cyw43_arch_lwip_threadsafe_background`, and provide an `lwipopts.h` (copy the one from `pico-examples/pico_w/wifi`). Pass `-DSSID=... -DPASSWORD=...` from CMake rather than hardcoding them.

=== "ESP-IDF"

    The ESP-IDF Wi-Fi setup is event driven and longer. Start from the bundled example, which is the canonical version:

    ```bash
    cp -r $IDF_PATH/examples/wifi/getting_started/station my-station
    cd my-station
    idf.py menuconfig        # "Example Configuration": set SSID and password
    idf.py -p /dev/ttyUSB0 flash monitor
    ```

    The core calls are `esp_netif_create_default_wifi_sta()`, `esp_wifi_init()`, `esp_wifi_set_config(WIFI_IF_STA, &cfg)`, `esp_wifi_start()`, and `esp_wifi_connect()`, with an event handler that waits for `IP_EVENT_STA_GOT_IP`. The `protocol_examples_common` component wraps all of this in a single `example_connect()` call for quick experiments.

## HTTP requests

=== "MicroPython"

    ```python
    import requests                 # older firmware: import urequests as requests

    r = requests.get("http://worldtimeapi.org/api/timezone/Europe/Prague")
    print(r.status_code, r.json()["datetime"])
    r.close()                       # always close, sockets are scarce

    r = requests.post("http://192.168.1.10:8000/data", json={"temp": 21.5})
    r.close()
    ```

    If `import requests` fails, install it with `mpremote mip install requests`.

=== "Arduino"

    ```cpp
    #include <HTTPClient.h>          // ESP32 and RP2040 cores

    void fetch() {
      HTTPClient http;
      http.begin("http://worldtimeapi.org/api/timezone/Europe/Prague");
      int code = http.GET();
      if (code > 0) Serial.println(http.getString());
      http.end();
    }
    ```

## A tiny web server

Serving a page from the board is the easiest remote control panel. This one shows a reading and toggles the LED:

=== "MicroPython"

    ```python
    import socket
    from machine import Pin, ADC

    led = Pin("LED", Pin.OUT)
    adc = ADC(26)

    s = socket.socket()
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind(("0.0.0.0", 80))
    s.listen(1)

    while True:
        cl, addr = s.accept()
        req = cl.recv(1024)
        if b"GET /toggle" in req:
            led.toggle()
        v = adc.read_u16() * 3.3 / 65535
        body = f"<h1>Pico</h1><p>ADC0: {v:.2f} V</p><p><a href='/toggle'>toggle LED</a></p>"
        cl.send(b"HTTP/1.0 200 OK\r\nContent-Type: text/html\r\n\r\n")
        cl.send(body.encode())
        cl.close()
    ```

    This serves one client at a time and blocks the loop in `accept()`. For anything bigger, use `asyncio` with `asyncio.start_server`, or the `microdot` framework (`mpremote mip install github:miguelgrinberg/microdot/src/microdot/microdot.py`).

=== "Arduino"

    ```cpp
    #include <WiFi.h>
    #include <WebServer.h>           // ESP32 and RP2040 cores

    WebServer server(80);

    void setup() {
      // ... connect to Wi-Fi as above ...
      pinMode(LED_BUILTIN, OUTPUT);
      server.on("/", [] {
        server.send(200, "text/html",
                    "<h1>ESP32</h1><a href='/toggle'>toggle LED</a>");
      });
      server.on("/toggle", [] {
        digitalWrite(LED_BUILTIN, !digitalRead(LED_BUILTIN));
        server.sendHeader("Location", "/");
        server.send(303);
      });
      server.begin();
    }

    void loop() {
      server.handleClient();          // must be called often
    }
    ```

## MQTT

MQTT is a small publish/subscribe protocol and the usual way to connect sensors to Home Assistant, Node-RED, or your own scripts. Run a broker such as Mosquitto on a PC or server, then:

=== "MicroPython"

    ```python
    # mpremote mip install umqtt.simple
    from umqtt.simple import MQTTClient
    import time

    def on_msg(topic, msg):
        print(topic, msg)
        if msg == b"on":
            led.on()

    c = MQTTClient("pico-1", "192.168.1.10")   # client id, broker address
    c.set_callback(on_msg)
    c.connect()
    c.subscribe(b"lab/pico-1/led")

    while True:
        c.publish(b"lab/pico-1/temp", b"21.5")
        c.check_msg()                           # non-blocking: handle incoming
        time.sleep(5)
    ```

=== "Arduino"

    ```cpp
    // library: "PubSubClient"
    #include <WiFi.h>
    #include <PubSubClient.h>

    WiFiClient net;
    PubSubClient mqtt(net);

    void onMsg(char *topic, byte *payload, unsigned int len) {
      Serial.printf("%s: %.*s\n", topic, len, payload);
    }

    void setup() {
      // ... connect to Wi-Fi ...
      mqtt.setServer("192.168.1.10", 1883);
      mqtt.setCallback(onMsg);
    }

    void loop() {
      if (!mqtt.connected() && mqtt.connect("esp32-1")) {
        mqtt.subscribe("lab/esp32-1/led");
      }
      mqtt.loop();
      static unsigned long last;
      if (millis() - last > 5000) {
        last = millis();
        mqtt.publish("lab/esp32-1/temp", "21.5");
      }
    }
    ```

Test from the PC with `mosquitto_sub -h 192.168.1.10 -t 'lab/#' -v` and `mosquitto_pub -h 192.168.1.10 -t lab/pico-1/led -m on`.

## Reliability tips

- **Reconnect.** Wi-Fi drops. Check `wlan.isconnected()` / `WiFi.status()` periodically and reconnect instead of assuming the link stays up.
- **Close sockets.** MicroPython boards have only a handful of sockets. A leaked one per request eventually stops all networking.
- **Use a [watchdog](storage-power.md#watchdog).** Network stacks can hang in ways your code cannot catch.
- **Power.** Wi-Fi transmit bursts draw 300 mA or more. A weak USB cable or regulator causes brownout resets exactly when the radio turns on.
- **ESP32 ADC2.** ADC2 pins stop working while Wi-Fi is active. Use ADC1 pins (GPIO32 to GPIO39) for analog inputs.
