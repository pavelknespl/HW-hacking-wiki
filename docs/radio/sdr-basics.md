# SDR Basics

A software defined radio moves most of the radio into software: the hardware digitizes a slice of the spectrum and hands the raw samples to your computer, where software demodulates and decodes them. One SDR can therefore receive almost any signal in its range, which makes it the universal tool for understanding a device's wireless link.

## What an SDR gives you

Instead of a fixed receiver built for one protocol, an SDR delivers **IQ samples**: a stream of complex numbers representing the signal across a chunk of bandwidth. From those samples software can:

- **Visualize** the spectrum (a waterfall) to see what is transmitting and when.
- **Demodulate** any scheme (AM, FM, ASK/OOK, FSK, PSK) by choosing the right software block.
- **Record** a capture for offline analysis, so you study the signal at leisure.

The two numbers that define what an SDR can do are its **frequency range** (what it can tune to) and its **bandwidth** (how wide a slice it can capture at once).

## Hardware tiers

| Device | Range | Bandwidth | TX? | Notes |
| --- | --- | --- | --- | --- |
| **RTL-SDR** | about 24 MHz to 1.7 GHz | about 2.4 MHz | No | The 30 dollar entry point. Receive only, but hugely capable for learning. |
| **HackRF One** | 1 MHz to 6 GHz | up to 20 MHz | Yes (half duplex) | The common do-everything transceiver. |
| **LimeSDR** | 100 kHz to 3.8 GHz | up to 60 MHz | Yes (full duplex) | More bandwidth and full duplex. |
| **USRP** (Ettus) | wide, model dependent | large | Yes | Research grade, expensive. |

Start with an **RTL-SDR**. It cannot transmit, but receiving and decoding is where most analysis happens, and you cannot break the law with a receiver.

## The software stack

- **GQRX** or **SDR++**: friendly spectrum viewers and demodulators for exploring and listening. Start here to find signals.
- **GNU Radio**: a visual dataflow environment (GNU Radio Companion) for building custom demodulators block by block. The serious tool once you know what you are decoding.
- **Universal Radio Hacker (URH)**: purpose-built for reverse engineering digital radio protocols. It records, detects modulation and bit timing, and helps you decode frames. For sub-GHz device protocols, URH is often the fastest path.
- **rtl_433**: decodes hundreds of common 433/868/915 MHz devices (sensors, weather stations, remotes) out of the box, a great first thing to run.

## A first capture

1. Plug in the SDR and confirm it is seen (`rtl_test` for RTL-SDR).
2. Open GQRX or SDR++, tune near the device's known frequency (check the [FCC ID](../attacks/recon.md#gather-the-public-record-first) for the exact band), and watch the waterfall while you trigger the device (press the remote, let the sensor report).
3. A burst appears on the waterfall when the device transmits. Note the center frequency and the width of the burst.
4. Record a capture of that burst for analysis.

## Finding the frequency

- The **FCC filing** lists the operating frequency and often the modulation. This is the fastest route. See [Reconnaissance](../attacks/recon.md).
- Common unlicensed bands to sweep: **433.92 MHz** and **868/915 MHz** (remotes, sensors, sub-GHz IoT), **2.4 GHz** (Wi-Fi, Bluetooth, Zigbee), and **125 kHz / 13.56 MHz** (RFID/NFC, which usually need a dedicated reader rather than a wideband SDR).
- Watch the waterfall across a band while triggering the device; the signal reveals itself.

## From samples to bits

Once you have a recording, the workflow in [URH](https://github.com/jopohl/urh) is typical:

1. **Identify the modulation** (ASK/OOK and FSK dominate cheap devices).
2. **Recover the symbol rate** from the shortest pulse.
3. **Demodulate to a bitstream.**
4. **Find structure**: preamble, sync word, payload, and a checksum or CRC.
5. **Capture several transmissions** and diff them to see which bits are fixed (addressing, command) and which change (counters, sensor values).

That decoded structure is what the [RF Attacks](rf-attacks.md) page builds on.


## Related pages

- [RF Attacks](rf-attacks.md)
- [Firmware Analysis](../attacks/firmware-analysis.md)
