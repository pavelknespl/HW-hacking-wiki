# hwhacking wiki

**Live site: <https://hwhacking.pavelknespl.cz/>**

A practical, vendor-neutral wiki for **hardware hacking, firmware security, and embedded reverse engineering**. Every page is a plain Markdown file under [`docs/`](docs/), categorized and served with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

The focus is concrete, reproducible technique: real commands, real pin names, and per-attack walkthroughs (target, preconditions, gear, steps, success condition, mitigation) rather than hand-waving.

> For authorized security research and education only.

## What's covered

- **Getting started & fundamentals** — lab setup, toolkit, methodology, chip ID, PCB recon, finding pins, logic levels, memory types, eFuse/OTP, soldering.
- **Protocols & interfaces** — UART, JTAG, SWD, I2C, SPI, eMMC/SD, USB, 1-Wire, MDIO, LPC, RS-485/Modbus, SMBus/PMBus, and MCU debug/trace extensions.
- **Automotive** — CAN bus, UDS/OBD-II, other vehicle buses.
- **Tools** — host software (serial consoles, OpenOCD, flashrom, binwalk, dd, UBI/NAND, sigrok, esptool, Depthcharge, openFPGALoader, UrJTAG) and hardware (ChipWhisperer, JTAGulator, Proxmark3, Flipper Zero, Tigard, Glasgow).
- **Attacks & techniques** — firmware acquisition, bootloader access/bypass, shells, fault injection & glitching, side-channel, EMFI, cold boot, DMA, Rowhammer, transient execution, TPM sniffing, secure elements, hardware wallets, decapping.
- **Targets** — MCUs (STM32 RDP, nRF52 APPROTECT, ESP32, AVR/PIC), platforms (boot ROM, SoC recovery, UEFI/BIOS, Intel ME/AMD PSP, Qualcomm EDL/MediaTek, Android bootloaders, TEE/TrustZone), and FPGA bitstream security.
- **Radio & RF** — SDR basics, NFC/RFID, BLE, Zigbee/Thread, LoRa, GNSS spoofing, key fobs, SIM/eSIM.
- **MCU programming, scripting**, and a **reference** section (pinouts, flash chips, glossary).

The nav in [`mkdocs.yml`](mkdocs.yml) is the full map.

## Run it locally

```bash
python3 -m pip install -r requirements.txt
mkdocs serve        # live reload at http://127.0.0.1:8000
mkdocs build        # static site in site/
```

## Contributing

One page = one Markdown file in a category folder under `docs/`. To add a page, create the file and add it to the `nav:` in `mkdocs.yml`. House style:

- English prose, clear and concrete; no em dashes or en dashes.
- Prefer working commands and real pin names; cite datasheets where it matters.
- Structure attacks per-attack: **target, precondition, gear, numbered steps, success condition, mitigation**.
- Short conventional-commit messages (`feat:`, `fix:`, `docs:` …).

## License

All content is licensed under [**Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)**](LICENSE). Reuse and adapt freely with attribution, sharing derivatives under the same license.
