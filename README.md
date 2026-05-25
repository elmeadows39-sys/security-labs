# security-labs

This repository documents a home network and homelab platform built for hands-on learning in infrastructure, security operations, and system administration — with a focus on resume-building projects targeting SOC/blue team roles.

The environment runs on Proxmox VE as the hypervisor with an OPNsense firewall managing segmented VLANs. VLAN 10 (Trusted) hosts all lab services. VLAN 30 (DMZ) hosts the honeypot. Services are deployed as Linux containers (LXCs) or VMs depending on workload requirements.

---

## Current Lab State

### Hardware

| Component | Details |
|-----------|---------|
| Hypervisor | Proxmox VE — bare metal (lab-pve, 192.168.1.108) |
| RAM | 32GB DDR3 |
| Storage | 4TB external HDD (/dev/sdb) — partitioned for backups, Nextcloud, and general storage |
| Network | ASUS RT-AC68U (primary router, DHCP/NAT) + Netgear R6250 (AP mode) |
| Switch | TP-Link TL-SG108PE V3 (managed, 192.168.1.200) |

### Storage Layout (4TB External HDD)

| Partition | Size | Mount | Purpose |
|-----------|------|-------|---------|
| sdb1 | 465GB | /mnt/pve/backup-hdd | Proxmox VM/LXC backups |
| sdb2 | 465GB | /mnt/pve/nextcloud-aio | Nextcloud AIO data |
| sdb3 | 2.7TB | /mnt/pve/storage | Immich photos, general storage |

### Network Layout

| VLAN | Subnet | Purpose |
|------|--------|---------|
| Flat | 192.168.1.0/24 | Legacy — remaining unmigrated devices |
| VLAN 10 (Trusted) | 192.168.10.0/24 | All lab services |
| VLAN 30 (DMZ) | 192.168.30.0/24 | Honeypot (internet-exposed) |

---

## Services

### Infrastructure

| Service | Type | ID | IP | Notes |
|---------|------|----|----|-------|
| Proxmox VE | Bare metal | — | 192.168.1.108 | Hypervisor for all lab workloads |
| OPNsense | VM 111 | — | 192.168.1.254 | Firewall/router — manages VLAN 10 and VLAN 30 |
| Pi-hole | LXC 101 | Debian 12 | 192.168.1.225 | Network-wide DNS filtering and ad blocking |
| Unbound | LXC 102 | Debian 12 | 192.168.10.30 | Recursive DNS resolver — Pi-hole upstream |
| Caddy | LXC 105 | Debian 12 | 192.168.10.20 | Reverse proxy with Let's Encrypt wildcard cert via Cloudflare DNS challenge |
| WireGuard (wg-easy) | LXC 109 | Debian 12 | 192.168.10.50 | Remote VPN access via DDNS (snoopylab23.asuscomm.com:51820) |

### Security

| Service | Type | ID | IP | Notes |
|---------|------|----|----|-------|
| Wazuh SIEM | VM 100 | Ubuntu 22.04 | 192.168.1.140 | Full SIEM stack — manager, indexer, dashboard (v4.14.5) |
| Cowrie Honeypot | LXC 108 | Debian 12 | 192.168.30.10 | SSH honeypot on port 22 via iptables redirect — logs feed into Wazuh via custom rules |
| win-server-01 | VM 113 | Windows Server 2019 | 192.168.1.150 | Active Directory domain controller (meadows-lab.local) — AD attack lab target, Wazuh agent enrolled |

### Monitoring

| Service | Type | ID | IP | Notes |
|---------|------|----|----|-------|
| Uptime Kuma | LXC 103 | Debian 12 | 192.168.10.10 | Service uptime monitoring with Discord alerts |

### Self-Hosted Apps

| Service | Type | ID | IP | Notes |
|---------|------|----|----|-------|
| Vaultwarden | LXC 104 | Debian 12 | 192.168.10.11 | Self-hosted Bitwarden-compatible password manager |
| Jellyfin | VM 107 | Ubuntu 22.04 | 192.168.10.60 | Media server — media via SMB from Supercomputer, metadata stored locally |
| Nextcloud AIO | VM 106 | Ubuntu 22.04 | 192.168.1.56 | File storage and collaboration (v12.9.2 with built-in Collabora) |
| Immich | VM 114 | Ubuntu 22.04 | 192.168.1.219 | Self-hosted photo/video backup (Google Photos alternative) |
| Authentik | VM 112 | Ubuntu 22.04 | 192.168.1.44 | SSO — stopped, slated for review |
| Actual Budget | LXC 110 | Debian 12 | 192.168.10.40 | Personal finance tracker (port 5006) |
| Portainer CE | LXC 110 | Debian 12 | 192.168.10.40 | Docker management UI (port 9443) |

---

## Internal DNS + Reverse Proxy

All services are accessible via .meadows-lab.com subdomains, managed by Pi-hole DNS and Caddy reverse proxy with trusted Let's Encrypt wildcard cert (*.meadows-lab.com) via Cloudflare DNS-01 challenge.

| Domain | Service |
|--------|---------|
| vault.meadows-lab.com | Vaultwarden |
| pihole.meadows-lab.com | Pi-hole admin |
| nextcloud.meadows-lab.com | Nextcloud |
| immich.meadows-lab.com | Immich |
| kuma.meadows-lab.com | Uptime Kuma |
| wireguard.meadows-lab.com | WireGuard web UI |
| jellyfin.meadows-lab.com | Jellyfin |
| wazuh.meadows-lab.com | Wazuh dashboard |
| proxmox.meadows-lab.com | Proxmox VE |
| authentik.meadows-lab.com | Authentik SSO |
| actual.meadows-lab.com | Actual Budget |
| portainer.meadows-lab.com | Portainer |

---

## Wazuh Agent Enrollment

Wazuh monitors 15 agents across the lab. All active agents at v4.14.5.

| Agent | ID | IP | Version | Status |
|-------|----|----|---------|--------|
| pihole-01 | 001 | 192.168.1.225 | 4.14.5 | ✅ Active |
| unbound-01 | 002 | 192.168.10.30 | 4.14.5 | ✅ Active |
| uptime-kuma-01 | 003 | 192.168.10.10 | 4.14.5 | ✅ Active |
| vaultwarden-01 | 004 | 192.168.10.11 | 4.14.5 | ✅ Active |
| caddy-01 | 005 | 192.168.10.20 | 4.14.5 | ✅ Active |
| wireguard-01 | 006 | 192.168.10.50 | 4.14.5 | ✅ Active |
| nextcloud-aio-02 | 007 | 192.168.1.56 | 4.14.5 | ✅ Active |
| immich-01 | 009 | 192.168.1.219 | 4.14.5 | ✅ Active |
| cowrie-01 | 010 | 192.168.30.10 | 4.14.5 | ✅ Active |
| lab-pve | 012 | 192.168.1.108 | 4.14.5 | ✅ Active |
| jellyfin-02 | 013 | 192.168.10.60 | 4.14.5 | ✅ Active |
| authentik | 014 | 192.168.1.44 | 4.14.3 | ⚠️ Disconnected |
| win-server-01 | 015 | 192.168.1.150 | 4.14.3 | ⚠️ Disconnected |
| opnsense-01 | 016 | 192.168.1.254 | 4.14.3 | ✅ Active |
| actual-01 | 017 | 192.168.10.40 | 4.14.5 | ✅ Active |

---

## Wazuh Custom Rules

Custom detection rules written and confirmed firing in Wazuh with MITRE ATT&CK tagging and Discord alerts.

| Rule ID | Description | MITRE | Level |
|---------|-------------|-------|-------|
| 100003 | Cowrie: successful login to honeypot | — | 10 |
| 100004 | Cowrie: command executed in honeypot | — | 10 |
| 100005 | Cowrie: file download attempted | — | 12 |
| 100006 | Cowrie: port scan or pivot attempt | — | 8 |
| 100010 | Brute force: 5+ failed logons within 120s (Event ID 4625) | T1110 | 12 |
| 100020 | Kerberoasting: RC4 encrypted Kerberos service ticket requested (Event ID 4769) | T1558.003 | 12 |

---

## Backups

- Proxmox backup job runs daily at 21:00
- Retention: last 7 backups
- Target: sdb1 (/mnt/pve/backup-hdd)
- Covers VMs/LXCs: 100, 101, 102, 103, 104, 105, 106, 107, 109, 112, 114, 115

---

## Network Diagram

![Network Diagram](diagrams/Diagram%202.0.png)

---

## In Progress

- **CIS Benchmark Hardening** — caddy-01 (54%), wireguard-01, uptime-kuma-01, actual-01 hardened. vaultwarden-01, wazuh-04, immich-01 remaining. Write-up doc coming soon.
- **VLAN Migrations** — VLAN 10 and VLAN 30 live. Remaining: nextcloud-aio-02, immich-01, wazuh-04, pihole-01, lab-pve.
- **Cowrie Internet Exposure** — Port forward ASUS port 22 → cowrie-01 (192.168.30.10:2222). Pending DMZ VLAN stable confirmation.
- **Suricata → Wazuh Integration** — BLOCKED. Known issue: Suricata 8.0.4 generates zero alerts on OPNsense running as a Proxmox VM with virtio NICs (GitHub issue #10097, April 2026). Workaround under investigation.

## Planned

- **VLAN Migration Template** — Documenting the standard LXC/VM migration process including PAM fix, netplan config, Caddy/Pi-hole/Uptime Kuma update steps.
- **Proxmox SSD Upgrade** — Kingston A400 240GB to resolve snapshot space limitations.
- **WireGuard Remote Access Fix** — Client endpoint configs need updating after wireguard-01 VLAN migration.

---

## Setup Docs

### Infrastructure
- [Pi-hole + Unbound](pihole-unbound-setup.md)
- [WireGuard (wg-easy)](wireguard-setup.md)
- [HDD Partition Layout](hdd-partition-layout.md)
- [Domain + Wildcard Cert Migration](domain-cert-migration.md)
- [Portainer](portainer-setup.md)
- VLAN Migration Template *(coming soon)*

### Security
- [Wazuh SIEM](wazuh-04.md)
- [Cowrie Honeypot](cowrie-setup.md)
- [Active Directory Attack Lab — Brute Force & Kerberoasting](ad-kerberoasting-lab.md)
- [VirusTotal FIM Integration](virustotal-fim-integration.md)
- CIS Benchmark Hardening *(coming soon)*

### Monitoring & Apps
- [Uptime Kuma](uptime-kuma-setup)
- [Nextcloud AIO](nextcloud-setup.md)
- [Immich](immich-setup.md)

