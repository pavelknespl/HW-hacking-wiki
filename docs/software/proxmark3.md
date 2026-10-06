# Proxmark3

The Proxmark3 is the reference tool for [RFID and NFC](../radio/nfc-rfid.md) work. It reads, writes, emulates, and sniffs both LF (125/134 kHz) and HF (13.56 MHz) cards, and it implements the attacks that recover keys from weak cards like MIFARE Classic. Where a [Flipper Zero](flipper-zero.md) handles the easy cases, the Proxmark3 is what you reach for when the card fights back. For authorized testing of cards and systems you own.

## Hardware

The common versions are the **Proxmark3 RDV4** (the polished, well-supported one) and cheaper **Easy** clones. All run the open-source Iceman firmware/client, which is the de facto standard; flash it first.

## The client

You drive it from a console (`pm3` / `proxmark3`), which auto-detects the antenna field and offers LF and HF command trees.

```
pm3 --> hw status          # check the device and antenna tuning
pm3 --> lf search          # identify an LF tag in the field
pm3 --> hf search          # identify an HF tag
```

## LF: read and clone

```
lf search                  # detect EM4100 / HID / etc.
lf em 410x reader          # read an EM4100 ID
lf t55xx detect            # identify a T5577 blank
lf em 410x clone --id <ID> # write that ID to a T5577
```

Most LF access control is a static ID, so this read-then-clone is often the whole engagement. See [NFC and RFID](../radio/nfc-rfid.md).

## HF: MIFARE Classic key recovery

The Proxmark3's signature capability. Crypto-1 is broken, and the client implements the recovery attacks:

```
hf 14a info                # identify the HF card
hf mf autopwn              # run the full key-recovery + dump pipeline
# or step by step:
hf mf darkside             # recover a key with no known keys
hf mf nested --key <k>     # recover remaining keys given one
hf mf hardnested ...       # for hardened Crypto-1 variants
hf mf dump                 # dump all sectors once keys are known
```

`autopwn` ties the attacks together: recover keys, then dump. With the dump you clone to a **magic card** (writable UID):

```
hf mf cload -f dump.bin    # load the dump onto a magic card
```

## Sniffing and emulation

- **Sniff** reader-to-card traffic to understand a protocol or catch secrets:

    ```
    hf 14a sniff            # capture an exchange, then: hf 14a list
    ```

- **Emulate** a card so a reader talks to the Proxmark instead:

    ```
    hf mf sim --1k -u <UID> # emulate a MIFARE Classic
    ```

These turn the Proxmark into a man-in-the-middle or a cloned credential for testing a reader.

## Stronger cards

Against **DESFire EV1/EV2**, **iCLASS SE**, and EMV, the broken-cipher attacks do not apply. The Proxmark still reads, sniffs, and tests these, but findings come from implementation flaws, legacy-mode downgrade, or keys recovered elsewhere (for example from [firmware](../attacks/firmware-analysis.md)), not from breaking the cryptography.

## Defender's view

If a Proxmark3 clones your badges in minutes, you are using LF static IDs or MIFARE Classic; move to DESFire EV2 or equivalent with diversified AES keys and mutual authentication. See the [NFC/RFID defender notes](../radio/nfc-rfid.md#defenders-view).

## Related pages

- [NFC and RFID](../radio/nfc-rfid.md)
- [Flipper Zero](flipper-zero.md)
- [Key Fobs and Rolling Codes](../radio/key-fobs.md)
- [Secure Elements](../attacks/secure-elements.md)
- [Side-Channel Attacks](../attacks/side-channel.md)
