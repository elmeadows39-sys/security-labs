# security-labs

This repository documents a home network and homelab platform built for hands-on learning in infrastructure, security operations, and system administration — with a focus on resume-building projects targeting SOC/blue team roles.

The environment runs on Proxmox VE as the hypervisor with an OPNsense firewall managing segmented VLANs. VLAN 10 (Trusted) hosts all lab services. VLAN 20 (IoT) isolates smart-home devices. VLAN 30 (DMZ) hosts the honeypot. Services are deployed as Linux containers (LXCs) or VMs depending on workload requirements.

---

## Current Lab State

### Hardware

| Component | Details |
|-----------|---------|
| Hypervisor | Proxmox VE — bare metal (lab-pve, 192.168.1.108) |
| RAM | 32GB DDR3 (board maximum) |
| Storage | 4TB external HDD (/dev/sdc) — partitioned for backups, Nextcloud, and general storage; mounted by UUID |
| Edge Router | TP-Link Archer AX1300 (DHCP/NAT for the household network) |
| Switch | TP-Link TL-SG108PE V3 (managed, 192.168.1.200) — 802.1Q trunk to Proxmox, dedicated IoT access port |
| Wireless (IoT) | Netgear R6250 in AP mode — broadcasts the isolated IoT network (VLAN 20) |
| Firewall (staged) | HP Z240 SFF — dedicated OPNsense hardware, pending cutover from the VM |

### Storage Layout (4TB External HDD)

| Partition | Size | Mount | Purpose |
|-----------|------|-------|---------|
| sdc1 | 465GB | /mnt/pve/backup-hdd | Proxmox VM/LXC backups |
| sdc2 | 465GB | /mnt/pve/nextcloud-aio | Nextcloud AIO data (NFS to nextcloud-aio-02) |
| sdc3 | 2.7TB | /mnt/pve/storage | Immich photos, general storage (NFS to immich-01) |

### Network Layout

| Network | Subnet | Purpose |
|---------|--------|---------|
| Flat | 192.168.1.0/24 | Household devices and the Proxmox host |
| VLAN 10 (Trusted) | 192.168.10.0/24 | All lab services |
| VLAN 20 (IoT) | 192.168.20.0/24 | Smart TVs, thermostat, cameras — internet only, isolated from all private networks |
| VLAN 30 (DMZ) | 192.168.30.0/24 | Honeypot — internet only, isolated from all private networks |

OPNsense's LAN interface carries no IP address — it serves only as the parent for the three VLANs. OPNsense reaches the flat network solely through its WAN (192.168.1.154), and household DNS runs through Pi-hole → Unbound.

---

## Services

### Infrastructure

| Service | Type | OS | IP | Notes |
|---------|------|----|----|-------|
| Proxmox VE | Bare metal | — | 192.168.1.108 | Hypervisor for all lab workloads |
| OPNsense | VM 111 | FreeBSD | 192.168.10.1 (WAN 192.168.1.154) | Firewall/router for VLANs 10, 20, 30 — Kea DHCP for the VLANs |
| Pi-hole | LXC 101 | Debian 12 | 192.168.10.110 | Network-wide DNS filtering and ad blocking — DNS server for the whole household |
| Unbound | LXC 102 | Debian 12 | 192.168.10.30 | Recursive DNS resolver — Pi-hole upstream |
| Caddy | LXC 105 | Debian 12 | 192.168.10.20 | Reverse proxy with Let's Encrypt wildcard cert via Cloudflare DNS challenge |
| WireGuard (wg-easy) | LXC 109 | Debian 12 | 192.168.10.50 | Remote VPN access — being rebuilt after the router replacement |

### Security

| Service | Type | OS | IP | Notes |
|---------|------|----|----|-------|
| Wazuh SIEM | VM 100 | Ubuntu 22.04 | 192.168.10.80 | Full SIEM stack — manager, indexer, dashboard (v4.14.8), Discord alerts at level 10+ |
| Cowrie Honeypot | LXC 108 | Debian 12 | 192.168.30.10 | SSH honeypot on port 22 via iptables redirect — logs feed into Wazuh via custom rules; uses public DNS, no access to internal networks |

### Monitoring

| Service | Type | OS | IP | Notes |
|---------|------|----|----|-------|
| Uptime Kuma | LXC 103 | Debian 12 | 192.168.10.10 | Service uptime monitoring with Discord alerts |
| Homepage | Docker on LXC 110 | Debian 12 | 192.168.10.40 | Lab dashboard — Uptime Kuma, Wazuh, Proxmox, Pi-hole, OPNsense, and calendar panels, each through a dedicated read-only account or token |

### Self-Hosted Apps

| Service | Type | OS | IP | Notes |
|---------|------|----|----|-------|
| Vaultwarden | LXC 104 | Debian 12 | 192.168.10.11 | Self-hosted Bitwarden-compatible password manager |
| Jellyfin | VM 107 | Ubuntu 22.04 | 192.168.10.60 | Media server — media via SMB from Supercomputer, metadata stored locally (currently powered off) |
| Nextcloud AIO | VM 106 | Ubuntu 22.04 | 192.168.10.90 | Files, calendar, and tasks (with built-in Collabora) — calendar syncs to Android via DAVx⁵ |
| Immich | VM 114 | Ubuntu 22.04 | 192.168.10.70 | Self-hosted photo/video backup (Google Photos alternative), with ImmichFrame slideshow |
| Actual Budget | LXC 110 | Debian 12 | 192.168.10.40 | Personal finance and bill tracking (port 5006) |
| Portainer CE | LXC 110 | Debian 12 | 192.168.10.40 | Docker management UI (port 9443) |

---

## Internal DNS + Reverse Proxy

All services are accessible via .meadows-lab.com subdomains, managed by Pi-hole DNS and Caddy reverse proxy with trusted Let's Encrypt wildcard cert (*.meadows-lab.com) via Cloudflare DNS-01 challenge.

| Domain | Service |
|--------|---------|
| dash.meadows-lab.com | Homepage dashboard |
| vault.meadows-lab.com | Vaultwarden |
| pihole.meadows-lab.com | Pi-hole admin |
| nextcloud.meadows-lab.com | Nextcloud |
| immich.meadows-lab.com | Immich |
| immichframe.meadows-lab.com | ImmichFrame slideshow |
| kuma.meadows-lab.com | Uptime Kuma |
| wireguard.meadows-lab.com | WireGuard web UI |
| jellyfin.meadows-lab.com | Jellyfin |
| wazuh.meadows-lab.com | Wazuh dashboard |
| proxmox.meadows-lab.com | Proxmox VE |
| opnsense.meadows-lab.com | OPNsense |
| actual.meadows-lab.com | Actual Budget |
| portainer.meadows-lab.com | Portainer |

---

## Wazuh Agent Enrollment

Wazuh monitors 13 agents across the lab. Linux agents run v4.14.8 and are version-held so they never move ahead of the manager.

| Agent | ID | IP | Version | Status |
|-------|----|----|---------|--------|
| pihole-01 | 001 | 192.168.10.110 | 4.14.8 | ✅ Active |
| unbound-01 | 002 | 192.168.10.30 | 4.14.8 | ✅ Active |
| uptime-kuma-01 | 003 | 192.168.10.10 | 4.14.8 | ✅ Active |
| vaultwarden-01 | 004 | 192.168.10.11 | 4.14.8 | ✅ Active |
| caddy-01 | 005 | 192.168.10.20 | 4.14.8 | ✅ Active |
| wireguard-01 | 006 | 192.168.10.50 | 4.14.8 | ✅ Active |
| nextcloud-aio-02 | 007 | 192.168.10.90 | 4.14.8 | ✅ Active |
| immich-01 | 009 | 192.168.10.70 | 4.14.8 | ✅ Active |
| cowrie-01 | 010 | 192.168.30.10 | 4.14.8 | ✅ Active |
| lab-pve | 012 | 192.168.1.108 | 4.14.8 | ✅ Active |
| jellyfin-02 | 013 | 192.168.10.60 | — | ⏸️ Powered off |
| opnsense-01 | 016 | 192.168.10.1 | 4.14.5 (plugin) | ✅ Active |
| actual-01 | 017 | 192.168.10.40 | 4.14.8 | ✅ Active |

---

## Wazuh Custom Rules

Custom detection rules written and confirmed firing in Wazuh with MITRE ATT&CK tagging and Discord alerts.

| Rule ID | Description | MITRE | Level |
|---------|-------------|-------|-------|
| 100003 | Cowrie: successful login to honeypot | — | 10 |
| 100004 | Cowrie: command executed in honeypot | — | 10 |
| 100005 | Cowrie: file download attempted | — | 12 |
| 100006 | Cowrie: port scan or pivot attempt | — | 8 |
| 100010 | Brute force: 5+ failed logons within 120s (Event ID 4625) — AD lab, now retired | T1110 | 12 |
| 100020 | Kerberoasting: RC4 encrypted Kerberos service ticket requested (Event ID 4769) — AD lab, now retired | T1558.003 | 12 |
| 100030 | VirusTotal false-positive suppression (vim.tiny) | — | 0 |
| 100050 | FIM: /etc/hosts modified | T1565.001 | 12 |

---

## Backups

- Proxmox backup job runs every third day at 04:00 (scheduled clear of nightly auto-updates and file-integrity scans)
- Retention: last 3 backups
- Target: sdc1 (/mnt/pve/backup-hdd)
- Covers VMs/LXCs: 100–111, 114

---

## Network Diagram

![Network Diagram](diagrams/Diagram%202.0.png)

*Diagram predates the router replacement and IoT VLAN — to be redrawn after the dedicated OPNsense hardware cutover.*

---

## In Progress

- **Dedicated OPNsense Hardware** — HP Z240 SFF staged to replace the OPNsense VM, removing the firewall from the hypervisor and freeing host memory.
- **IoT Device Migration** — VLAN 20 live; living room TV migrated. Remaining: bedroom TV, thermostat, camera.
- **Lab Dashboard** — Homepage live with core panels; next up are custom panels for storage, backups, and bills/tasks.
- **CIS Benchmark Hardening** — caddy-01 (54%), wireguard-01, uptime-kuma-01, actual-01 hardened. vaultwarden-01, wazuh-04, immich-01 remaining. Write-up doc coming soon.
- **Cowrie Internet Exposure** — Port forward edge router → OPNsense → cowrie-01 (192.168.30.10:2222).
- **Suricata → Wazuh Integration** — BLOCKED. Known issue: Suricata 8.0.4 generates zero alerts on OPNsense running as a Proxmox VM with virtio NICs (GitHub issue #10097, April 2026). Revisit after the move to dedicated hardware.

## Planned

- **Proxmox Host off the Flat Network** — Move lab-pve onto a VLAN after the hardware firewall cutover.
- **VLAN Migration Template** — Documenting the standard LXC/VM migration process including PAM fix, netplan config, Caddy/Pi-hole/Uptime Kuma update steps.
- **Proxmox SSD Upgrade** — Larger SSD for the VM storage pool to resolve snapshot space limitations.
- **WireGuard Remote Access Fix** — Needs dynamic DNS and a new port forward after the router replacement.

---

## Setup Docs

### Infrastructure
- [Pi-hole + Unbound](docs/pihole-unbound-setup.md)
- [WireGuard (wg-easy)](docs/wireguard-setup.md)
- [HDD Partition Layout](docs/hdd-partition-layout.md)
- [Domain + Wildcard Cert Migration](docs/domain-cert-migration.md)
- [Portainer](docs/portainer-setup.md)
- [IoT VLAN 20 — Isolated Wi-Fi for Smart Devices](docs/iot-vlan20-build.md)
- VLAN Migration Template *(coming soon)*

### Security
- [Wazuh SIEM](docs/wazuh-04.md)
- [Cowrie Honeypot](docs/cowrie-setup.md)
- [Active Directory Attack Lab — Brute Force & Kerberoasting](docs/ad-kerberoasting-lab.md)
- [VirusTotal FIM Integration](docs/virustotal-fim-integration.md)
- [Router Replacement & OPNsense Routing Fix](docs/router-replacement-opnsense-routing.md)
- [Z240 Cutover Attempt & Edge Router Failure](docs/z240-cutover-and-edge-router-failure.md)
- CIS Benchmark Hardening *(coming soon)*

### Monitoring & Apps
- [Uptime Kuma](docs/uptime-kuma-setup.md)
- [Nextcloud AIO](docs/nextcloud-setup.md)
- [Immich](docs/immich-setup.md)


