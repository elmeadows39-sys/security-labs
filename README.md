# security-labs
This repository documents a home network and homelab platform.

The environment is designed with a trusted home network and a separate lab network to safely support learning, experimentation, and resume-building projects. The lab is intentionally isolated to reduce risk to everyday devices while enabling hands-on work with infrastructure, security, and system design.

Over time, this project may include media services, experimental workloads, and security labs, all built on the same underlying network foundation.

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

## High-Level Network Layout (Current)
Internet → ASUS Router (Primary / Trusted Network)
DHCP, NAT, firewall handled here
Flat subnet (192.168.1.0/24) by design

Trusted Devices
Main PC
Phones, TV, IoT
Omni Wi-Fi extender (same SSID)

Lab Access Layer
Netgear router in Access Point mode
No routing, NAT, or DHCP
Acts as Wi-Fi AP and switch only

Current Lab State (Phase 1 – Operational)
The homelab runs on Proxmox VE installed on bare-metal hardware.
Workloads are primarily deployed as Linux containers (LXCs) for efficiency, isolation, and modular growth, with select VMs used where full virtualization is required.

Core Services (Operational)
Proxmox VE – Bare-metal hypervisor
Pi-hole – Network-wide DNS filtering
Unbound – Recursive DNS resolver paired with Pi-hole
Uptime Kuma – Service availability monitoring
Vaultwarden – Self-hosted password manager
Caddy – Internal reverse proxy and HTTPS groundwork
Centralized Log VM – Virtual machine for logging and future security analysis
All services are reachable on the LAN and configured to start automatically.

Data Protection & Reliability
External USB backup storage attached to Proxmox
Daily automated backups scheduled at the datacenter level
Retention policy: last 7 backups
Snapshots used selectively for pre-change rollback
Backups validated by test restores

In Progress
Jellyfin Media Server – Planned VM-based deployment (migration from desktop Plex)
SIEM / Log Analysis – Log VM in place as foundation for future tooling
Planned (Next Phase)
VLAN-based network segmentation
Firewall rule refinement and hardening
SOAR tooling for security automation
Nextcloud for self-hosted file sharing
## Network Diagram
![Network Diagram](diagram/Homelab1.1.png)

## Network Architecture Notes

### Current Network Design
The lab currently operates on a flat network (192.168.1.1/24) managed by the ASUS RT-AC68U. The Netgear R6250 operates in Access Point mode, extending WiFi coverage with no routing or DHCP of its own.

### Security Considerations
True VLAN-based segmentation between the trusted and lab networks is not currently implemented due to hardware limitations on the ASUS router. Network isolation is achieved physically where possible.

### Planned Improvements
- VLAN-based network segmentation (requires managed switch or router upgrade)
- Firewall rule refinement between trusted and lab zones
- Full traffic isolation for Proxmox and lab workloads

## Additional Services

**SSH Bastion (LXC 108)** — Dedicated SSH access point for secure console access to lab services.

**Local AI / Ollama (Supercomputer)** — Llama 3.2 deployed locally for private, offline AI inference. Open WebUI setup planned for browser-based chat interface.

**WireGuard VPN (LXC 109 - 192.168.1.110)** — Remote access via snoopylab23.asuscomm.com:51820. 
Running as wg-easy in Docker with --network host. Web UI at http://192.168.1.110:51821. 
DNS routed through Pi-hole (192.168.1.225) for ad blocking on mobile. Phone client tested and working.

**Uptime Kuma** — Monitoring for Pi-hole, Unbound, Vaultwarden, and WireGuard. 
Discord webhook notifications configured for downtime alerts.
