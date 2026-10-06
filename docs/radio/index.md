# Radio and RF

Many devices talk wirelessly, and that radio link is an attack surface in its own right. With a software defined radio you can observe, decode, and (within legal limits) interact with these signals. This section is a starting point, not a full RF course: enough to recognize what a device is transmitting and begin analyzing it.

<div class="grid cards" markdown>

-   __SDR Basics__

    What an SDR is, the hardware tiers, and the software to capture and view signals.

    [:octicons-arrow-right-24: Read](sdr-basics.md)

-   __RF Attacks__

    Common wireless weaknesses and how analysis of them proceeds.

    [:octicons-arrow-right-24: Read](rf-attacks.md)

-   __NFC and RFID__

    125 kHz and 13.56 MHz tags, MIFARE, cloning with a Proxmark3.

    [:octicons-arrow-right-24: Read](nfc-rfid.md)

-   __Bluetooth Low Energy__

    Sniffing BLE, GATT enumeration, and pairing weaknesses.

    [:octicons-arrow-right-24: Read](ble.md)

-   __Zigbee, Thread, 802.15.4__

    Smart-home mesh networks and the Zigbee join-key weakness.

    [:octicons-arrow-right-24: Read](zigbee-thread.md)

-   __LoRa and LoRaWAN__

    Long-range IoT and where its AES keys leak.

    [:octicons-arrow-right-24: Read](lora.md)

-   __GNSS and GPS Spoofing__

    Why civilian GPS is spoofable, and how to defend against it.

    [:octicons-arrow-right-24: Read](gps-spoofing.md)

-   __Key Fobs and Rolling Codes__

    Fixed codes, rolling codes, relay attacks on car keys.

    [:octicons-arrow-right-24: Read](key-fobs.md)

</div>

!!! warning "Transmitting is regulated"
    Receiving is generally unrestricted, but **transmitting** on most frequencies requires a license or is limited to specific bands and power levels. Only transmit where you are legally permitted and on equipment and frequencies you are authorized to use. Everything here assumes lawful, authorized testing.
