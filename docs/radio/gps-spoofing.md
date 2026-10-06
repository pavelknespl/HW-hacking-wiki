# GNSS and GPS Spoofing

GPS (and the broader family of GNSS: GLONASS, Galileo, BeiDou) feeds position and precise time into navigation, drones, vehicles, phones, and timing-dependent infrastructure. The civilian signal is unauthenticated, which makes **spoofing** (transmitting fake satellite signals so a receiver computes a false position or time) a recognized threat. This page explains the concept for research and defensive understanding.

!!! danger "Transmitting GNSS is heavily regulated and dangerous"
    GPS frequencies are protected. Transmitting, even at low power, can disrupt aviation, emergency services, and timing systems far beyond your bench, and is illegal in most jurisdictions. Any experimentation must be fully **wired/shielded** (RF-tight enclosure or direct cable into the receiver, never over the air) and on receivers you own. This page is for understanding the attack and defending against it.

## Why it works

The civilian GPS **L1 C/A** signal has no authentication. A receiver trusts whatever satellite-like signals it hears and solves for position and time from their timing. If an attacker produces signals that look like a valid constellation but encode a different position or time, the receiver believes them. The signals are also extremely weak as received (below the noise floor), so a local transmitter easily overpowers the real ones.

Two related attacks:

- **Jamming**: drown the band in noise so the receiver loses lock. Denial of service.
- **Spoofing**: feed coherent fake signals so the receiver reports a chosen, wrong position or time. The more dangerous and subtle attack.

## How spoofing is done (conceptually)

- A transmit-capable SDR (**HackRF**, **USRP**, **bladeRF**) plus open-source signal generators (for example **gps-sdr-sim**) synthesize a baseband GPS signal for a chosen location and time, which the SDR plays back.
- A careful spoofer first **matches the real signal** (same apparent position) and then slowly **drags** the solution away, so the receiver never loses lock and the victim does not notice a jump.
- **Time spoofing** matters as much as position: many power grids, telecom networks, and financial systems rely on GPS for precise time, so a false time is its own attack.

Everything above must be done into a cable or shielded box, never radiated.

## What it affects

- **Navigation**: drones redirected or geofences defeated, vehicles misled.
- **Timing**: disruption of systems that discipline their clocks to GPS.
- **Logging/evidence**: falsified location trails.

## Defending against it

- **Sanity checks**: reject impossible jumps in position/velocity/time; cross-check against inertial sensors (IMU), wheel odometry, or a known good clock.
- **Multi-constellation and multi-frequency** receivers are harder to spoof consistently.
- **Signal-quality monitoring**: sudden uniform increase in signal strength across satellites, or inconsistent carrier-phase, flags a spoofer.
- **Authenticated GNSS**: Galileo's **OSNMA** adds navigation-message authentication; use it where available.
- **Antenna techniques** (angle-of-arrival, multi-antenna) detect that all "satellites" arrive from one direction.

## Related pages

- [SDR Basics](sdr-basics.md)
- [RF Attacks](rf-attacks.md)
- [Key Fobs and Rolling Codes](key-fobs.md)
- [Automotive overview](../automotive/index.md)
