# USB

USB is both a target and a tool. As a target, a device's USB interface exposes descriptors, class drivers, and sometimes hidden vendor commands or bootloaders that make a useful attack surface. As a tool, a programmable USB device lets you emulate hardware, fuzz hosts, and reach debug and recovery modes. This page covers reading what a device presents and the tooling to probe or emulate it.

## How enumeration works

When a device is plugged in, the host asks it to describe itself through a tree of **descriptors**:

- **Device descriptor**: vendor ID (VID), product ID (PID), device class, and USB version.
- **Configuration descriptor**: power and the set of interfaces.
- **Interface descriptors**: each function (mass storage, HID, CDC-serial, vendor-specific) and its class.
- **Endpoint descriptors**: the pipes data flows through, and their direction and type.
- **String descriptors**: human-readable manufacturer, product, and serial strings.

The VID/PID pair identifies the device and drives which host driver binds. A **vendor-specific** interface class is always worth attention: it means custom commands the manufacturer defined, which are often undocumented and under-hardened.

## Reading what a device presents

On Linux, enumerate and dump descriptors:

```bash
lsusb                                  # VID:PID and product strings
lsusb -v -d 1234:5678                  # full descriptor tree for one device
lsusb -t                               # topology, which driver bound each interface
dmesg | tail                           # kernel messages on plug-in
```

Capture the actual USB traffic to see the commands a host and device exchange:

```bash
sudo modprobe usbmon
# then capture in Wireshark on the usbmonN interface, or:
sudo tcpdump -i usbmon1 -w usb.pcap
```

Wireshark decodes USB and most class protocols, so you can watch a vendor tool talk to the device and reverse the command set.

## The interesting surfaces

- **Vendor-specific control requests.** Custom `bRequest` values on the control endpoint, frequently used for configuration, debug, or firmware update. Enumerate them by fuzzing the control transfer space and observing responses.
- **DFU (Device Firmware Upgrade).** A standard class for flashing. If a device exposes a DFU interface, `dfu-util` can often read or write firmware directly:

    ```bash
    dfu-util -l                        # list DFU-capable devices
    dfu-util -U dump.bin               # upload (read) firmware if allowed
    ```

- **Mass storage and CDC.** A device that exposes storage may leak files; a CDC-ACM interface is a [serial console](../software/serial-consoles.md) over USB.
- **Hidden recovery modes.** Many SoCs drop into a USB download mode (for example NXP, Allwinner FEL, TI, Espressif) that accepts code over USB, sometimes bypassing other protections.

## Emulation and fuzzing with Facedancer

The powerful technique is to **be** a USB device in software, so you control every byte of every descriptor and response. **Facedancer** (with compatible hardware such as a GreatFET) lets Python emulate arbitrary USB devices:

- Present malformed or boundary-case descriptors to **fuzz a host's** USB stack and drivers.
- Emulate a specific device to study how a host interacts with it.
- Implement a class (HID, mass storage) with deliberately hostile behavior for testing.

```python
# conceptual Facedancer device: respond to enumeration with custom descriptors
# the framework handles the USB state machine; you define descriptors and handlers
```

Host-side fuzzing matters because USB drivers run early, in privileged kernel context, and parse attacker-controlled descriptors, which has produced real kernel vulnerabilities. **umap2** builds on Facedancer specifically to fuzz hosts by class.

## USB as a bridge to other buses

Many of the adapters elsewhere in this wiki are USB devices that bridge to [UART](uart.md), [SPI](spi.md), [I2C](i2c.md), or [JTAG](jtag.md). An FT2232H, for example, is a USB device whose vendor interface `pyftdi` drives to speak those buses. So USB is often the layer your tools ride on, not only a target in itself.

## Related pages

- [Pin Reference](../reference/pin-reference.md)
- [Serial Consoles](../software/serial-consoles.md)
- [Firmware Extraction](../attacks/firmware-extraction.md)
- [Firmware Analysis](../attacks/firmware-analysis.md)
- [The Toolkit](../getting-started/toolkit.md)
- [Logic Analyzers (sigrok)](../software/logic-analyzers.md)
