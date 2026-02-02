# security-labs
This repository documents a segmented home network and homelab platform.

The environment is designed with a trusted home network and a separate lab network to safely support learning, experimentation, and resume-building projects. The lab is intentionally isolated to reduce risk to everyday devices while enabling hands-on work with infrastructure, security, and system design.

Over time, this project may include media services, experimental workloads, and security labs, all built on the same underlying network foundation.

## High-Level Network Layout

Internet → ASUS Router (Trusted Network)
- Main PC
- Phones, TV, IoT
- Omni Wi-Fi extender (same SSID)

ASUS Router → Netgear Router (Lab Network)
- Old PC used for testing and experiments

The lab network is isolated from the trusted home network and is intended for learning, experimentation, and future homelab services.
