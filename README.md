# sec-lab-iot

A self-built, isolated IoT security lab, created from scratch to study **Identity and Access Management (IAM)** on real hardware. It runs a full offensive campaign against deliberately vulnerable devices, then rebuilds the system around verifiable identity.

The project is told in two parts that share the same thread:

- **Part 1 — Breaking Trust (offensive):** get onto the network, map the devices, listen to the traffic, recover a hard-coded identity from firmware, then step into the middle to hijack a physical device.
- **Part 2 — Rebuilding Trust (defensive & IAM):** *coming soon.* mTLS, one certificate-based identity per device, short-lived tokens, and revocation driven by an intrusion detection system.

Every flaw encountered makes the same case: in IoT, being on the network should not grant trust. Trust must rest on a proven identity, which is exactly what IAM provides.

---

## Legal and ethical scope

> Every attack in this repository was carried out **solely on my own isolated lab, which I own**, for educational purposes. No device is exposed to the Internet, and nothing here targets third-party systems.

The offensive material documents **what each step demonstrates and why it matters**, not a turnkey how-to against live targets. Sensitive artifacts (Wi-Fi handshake captures, recovered keys, firmware dumps, real secrets) are intentionally **kept out of the repository**.

---

## Why this lab exists

Most IoT compromises do not come from exotic exploits. They come from implicit trust: a device obeys whoever speaks on the network, a broker accepts whoever connects, an identity is a string that can be copied. This lab makes that failure mode concrete, end to end, and then shows how an identity layer closes it.

It is also a deliberate exercise in **building my own tools**: re-implementing the logic (a port scanner, a packet sniffer, a MITM proxy) to understand what actually happens on the wire, rather than assembling ready-made tools.

---

## The lab at a glance

### Hardware

- **Router** — GL-MT300N-V2 running OpenWrt. Provides the dedicated Wi-Fi, DHCP, DNS, and a zone-based firewall.
- **Raspberry Pi 4** — central server, hosts the Mosquitto MQTT broker (and, later, the IDS, CA and identity provider).
- **3 × ESP32** — the deliberately vulnerable devices (see below).
- Dev machine: ThinkPad on Fedora (VS Code + PlatformIO).

### Three devices, three pillars of security

Each device is the archetype of a real category and carries a distinct class of vulnerability. Together they cover the Confidentiality, Integrity and Availability triad.

| Device | Real-world archetype | Threatens | Core weakness |
|---|---|---|---|
| **Camera** (ESP32-CAM) | IP cameras, video intercoms | Confidentiality | Stream reachable with no authentication |
| **Sensor** (ESP32 + DHT22) | Home / industrial telemetry | Integrity (+ Confidentiality) | Readings in clear text, no proof of origin |
| **Actuator** (ESP32 + LED/button) | Smart locks, relays, breakers | Availability / Integrity | Obeys any order; identity hard-coded in firmware |

### Network

A dedicated Wi-Fi network (`IoT-Lab`, 192.168.50.0/24), segmented by **firewall zones** with a directional policy: the IoT zone cannot reach the trusted network (`iot -> lan` blocked), the trusted network can administer the devices (`lan -> iot` allowed), and devices get only DHCP and DNS from the router. This is defence in depth — a compromised device cannot spread to the rest.

---

## Repository structure

```
sec-lab-iot/
├── docs/                 Lab-wide documentation
│   ├── network setup (OpenWrt, firewall zones)
│   ├── Mosquitto broker (deliberately open, port 1883)
│   └── threat model (mapped to the OWASP IoT Top 10, 2018)
├── devices/              Firmware + setup notes for the 3 ESP32 devices
│   ├── camera/           ESP32-CAM, open video stream
│   ├── sensor/           DHT22 telemetry over unauthenticated MQTT
│   └── actuator/         LED/button, hard-coded MQTT client-id
├── offensive/            Part 1 — the offensive campaign
│   ├── wifi/             WPA2 entry (notes; captures are git-ignored)
│   ├── reverse/          Firmware extraction & analysis (Ghidra)
│   └── mitm/             ARP spoofing + transparent MQTT proxy
└── defensive/            Part 2 — IAM / zero-trust  (coming soon)
```

> Device firmware ships `secrets.example.h` templates only. Real secrets, keys and captures are never committed.

---

## Part 1 — Breaking Trust (offensive)

A single chain, each link enabling the next. Described here at the "what it proves" level.

1. **Get in — Wi-Fi.** Capture the WPA2 handshake and crack it offline against a weak passphrase. *A network protected by one shared secret is only a perimeter, and a perimeter is worth no more than the strength of its secret.*
2. **Map — recon.** A hand-written port scanner reduces the /24 to the live machines and their open services (the broker on 1883, the camera's web server on 80). *You only protect what you know is exposed.*
3. **Listen — sniffing.** A hand-written MQTT sniffer decodes the clear-text traffic: readings, states, even connection credentials. *Confidentiality cannot rest on the application protocol; it must be provided by the transport layer.*
4. **Analyse — firmware reverse.** Physical access to the serial port is enough to dump the flash (no flash encryption, no secure boot). Ghidra reconstructs the whole connection configuration, including the hard-coded client-id. *An identity based on a copyable string, on a service that verifies nothing, is not an identity — it is a label.*
5. **Hijack — MITM.** ARP spoofing places the attacker on the path; a transparent TCP proxy rewrites commands in transit and freezes the reported state, fully decoupling physical reality from supervision. *Once on the path, a clear-text, unauthenticated bus is entirely manipulable.*

Mapped against the **OWASP IoT Top 10 (2018)** throughout — see `docs/` for the full threat model.

---

## Hand-coded tooling

The philosophy is to re-implement the logic in order to understand it. Two tools grew into standalone projects with their own repositories:

- **[rmap](https://github.com/MkDriss/rmap)** — a TCP port scanner written in Rust.
- **[rnifler](https://github.com/MkDriss/rnifler)** — a defensive MQTT sniffer written in Rust.

Within this repository, the ARP spoofer and the transparent MQTT proxy (Python) are built by hand for the MITM stage. Standard, mature tools are used as-is where re-implementing them would miss the point: **Ghidra** (firmware analysis), **aircrack-ng** (handshake capture and cracking), **scapy** (ARP crafting).

---

## Part 2 — Rebuilding Trust (coming soon)

The defensive half rebuilds trust on verifiable ground:

- **mTLS** on the MQTT channel — authenticity and encryption; opening a TCP connection is no longer enough.
- **One certificate-based identity per device**, issued by the lab's own certificate authority — copying a firmware no longer lets you impersonate a device, because its private key would be required.
- **Short-lived tokens and least privilege** — each identity may only do what it needs, within a bounded window.
- **Automatic revocation** driven by an **intrusion detection system** — a compromised identity is detected and cut off.

The headline demonstration will be a before / after comparison: the forged `OFF` command that hijacks the actuator today will simply be rejected.

---

## Status

| Part | State |
|---|---|
| Network foundations, vulnerable devices, threat model | Done |
| Part 1 — offensive campaign (Wi-Fi, recon, reverse, MITM) | Done |
| Part 2 — defence & IAM (mTLS, CA, IdP, IDS-driven revocation) | Coming soon |

---

## Related projects

- **[rmap](https://github.com/MkDriss/rmap)** — TCP port scanner (Rust).
- **[rnifler](https://github.com/MkDriss/rnifler)** — defensive MQTT sniffer (Rust).
