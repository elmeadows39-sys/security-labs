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

## Current Lab State (Phase 1 – Operational)

The homelab is currently running on Proxmox VE installed on bare metal hardware within the isolated lab network. Services are deployed as individual Linux containers (LXCs) to support modular design, security isolation, and incremental expansion.

### Core Services (Operational)
- **Proxmox VE** – Bare metal hypervisor hosting all lab workloads
- **Pi-hole** – Network-wide DNS filtering and ad blocking
- **Unbound** – Recursive DNS resolver paired with Pi-hole
- **Uptime Kuma** – Service availability and uptime monitoring
- **Vaultwarden** – Self-hosted password management service
- **Caddy** – Reverse proxy container (HTTPS groundwork and internal routing)
- **Centralized Log VM** – Dedicated virtual machine for log aggregation and future analysis

All services are reachable within the lab network and configured to start automatically.

## In Progress / Provisioning

- **Jellyfin Media Server** – Media server migration planned from an existing Plex deployment on the main desktop system
- **SIEM / Log Analysis Stack** – Log VM deployed as the foundation for future security monitoring and analysis tooling

## Planned (Next Phase)

- **SOAR tooling** for security automation and response workflows
- **Nextcloud** for self-hosted file sharing and collaboration
- Additional network segmentation and security controls
