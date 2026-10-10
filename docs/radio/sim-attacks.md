# SIM and eSIM Attacks

A SIM is a smartcard (a [secure element](../attacks/secure-elements.md)) that holds the subscriber identity and the keys that authenticate a phone to the mobile network, and it runs its own applications on top. That makes it both a credential store worth cloning and a remotely-reachable execution environment worth attacking. eSIM (eUICC) moves the same card into a soldered chip provisioned over the air, changing the physical attack surface but not the fundamentals.

!!! warning "Authorized use only"
    For cards and accounts you own or are authorized to test. Cloning a SIM you do not own is fraud.

## What a SIM actually is

- A smartcard speaking **ISO 7816** (contact) with a filesystem of **elementary files** (EFs) and applications, administered with **APDU** commands.
- It stores the **IMSI** (identity) and a secret key **Ki**, and runs the network **authentication algorithm** (COMP128 v1/v2/v3 in GSM, or **MILENAGE** in 3G/4G/5G) to answer the network's challenge without revealing Ki.
- It can run **applets** (the SIM Toolkit / S@T browser) that the carrier pushes and that can be triggered by special SMS.

Read it with a smartcard reader (`pcsc_scan`, `opensc-tool`) and tools like **SIMtrace** (sniff the phone↔SIM APDU traffic) or **pySim**.

```bash
pcsc_scan                       # see the ATR when the card is inserted
pySim-read.py -p 0              # dump readable EFs (ICCID, IMSI, phonebook, SMS)
```

---

## Attack 1 — clone a weak-crypto SIM (COMP128v1)

**Target:** old GSM SIMs using **COMP128v1**, whose authentication function is cryptographically broken.
**Why it works:** COMP128v1 leaks Ki through a chosen-challenge collision attack; a few thousand to tens of thousands of `RUN GSM ALGORITHM` queries recover Ki, after which the SIM can be duplicated.

Steps:

1. Insert the card in a reader; confirm it is COMP128v1.
2. Run the collision attack (repeated authentication with crafted RANDs) to recover **Ki** (classic tools automate this).
3. Program Ki + IMSI onto a programmable SIM → a working clone.

**Success condition:** the clone authenticates to the network as the original.
**Mitigation:** COMP128v2/v3 or MILENAGE (modern cards) resist this; the key is never extractable this way.

## Attack 2 — Simjacker / WIB (applet attacks via SMS)

**Target:** SIMs that ship the **S@T Browser** (Simjacker) or **WIB** applet with weak security settings.
**Why it works:** a specially-formatted binary SMS carries commands the vulnerable applet executes on the card — retrieving location/IMEI, sending SMS, opening URLs — without the user seeing anything. The handset just relays it to the SIM.

**Steps (defensive/research framing):** identify whether the SIM exposes S@T/WIB and whether its minimum security level permits unauthenticated commands; a permissive card executes attacker SMS.
**Mitigation:** carriers disable/secure these applets and filter binary SMS at the network; set the applet minimum security level to require signing.

## Attack 3 — physically attack the card (secure-element methods)

**Target:** the Ki on a modern (MILENAGE) card where the crypto is sound.
**Idea:** fall back to the [secure-element playbook](../attacks/secure-elements.md) — [side-channel](../attacks/side-channel.md) (power/EM DPA on the authentication) and [fault injection](../attacks/fault-injection.md) against the card's crypto, or [invasive](../attacks/decapping.md) attacks on the die. Hard, part-specific, and dependent on that card's countermeasures.

## eSIM / eUICC differences

- The card is a soldered **eUICC** chip provisioned by downloading a **profile** over the air (SGP.22 RSP). No card to swap, so cloning via physical swap is out.
- The attack surface shifts to the **provisioning/RSP flow** (profile download, the SM-DP+ server, the activation QR/code), the eUICC's own secure-element strength, and the LPA (Local Profile Assistant) software on the phone.
- The baseband/[secure element](../attacks/secure-elements.md) holding the eUICC is the physical target; the logical targets are the profile-management APDUs and the remote provisioning servers.

## Decision flow

```mermaid
graph TD
  A[Identify card + algo] --> B{COMP128v1?}
  B -->|yes| C[Attack 1: clone via Ki recovery]
  B -->|no| D{S@T / WIB applet exposed?}
  D -->|yes| E[Attack 2: Simjacker/WIB SMS]
  D -->|no| F[Attack 3: SE side-channel / FI]
  G[eSIM] --> H[Target RSP provisioning + eUICC SE]
```

## Defender's view

Use MILENAGE cards (not COMP128v1), disable or lock down S@T/WIB applets and set signed-command minimum security levels, filter binary/OTA SMS in the network, and treat Ki as a secret that must never leave the card. For eSIM, secure the RSP chain (authenticated SM-DP+, protected activation codes) and rely on a strong eUICC secure element. See [secure elements](../attacks/secure-elements.md).

## Related pages

- [Secure Elements](../attacks/secure-elements.md)
- [NFC and RFID](nfc-rfid.md)
- [Side-Channel Attacks](../attacks/side-channel.md)
- [Fault Injection and Glitching](../attacks/fault-injection.md)
- [RF Attacks](rf-attacks.md)
