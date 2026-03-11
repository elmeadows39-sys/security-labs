# security-labs

This repository documents a home network and homelab platform built for hands-on learning in infrastructure, security operations, and system administration — with a focus on resume-building projects targeting SOC/blue team roles.

The environment runs on a flat home network (`192.168.1.0/24`) with Proxmox VE as the hypervisor. Services are deployed as Linux containers (LXCs) or VMs depending on workload requirements.

---

## Current Lab State

### Hardware
| Component | Details |
|-----------|---------|
| Hypervisor | Proxmox VE 6.17.2 — bare metal (`lab-pve`, 192.168.1.108) |
| RAM | 24GB DDR3 (upgrade to 32GB planned) |
| Boot Disk | Local SSD (upgrade to Kingston A400 240GB planned) |
| Storage | 4TB external HDD (`/dev/sdb`) — partitioned for backups, Nextcloud, and general storage |
| Network | ASUS RT-AC68U (primary router, DHCP/NAT) + Netgear R6250 (AP mode) |
| Switch | TP-Link LS108GP (unmanaged — replacement with managed TP-Link TL-SG108PE V3 ordered for VLAN support) |

### Storage Layout (4TB External HDD)
| Partition | Size | Mount | Purpose |
|-----------|------|-------|---------|
| sdb1 | 465GB | `/mnt/pve/backup-hdd` | Proxmox VM/LXC backups |
| sdb2 | 465GB | `/mnt/pve/nextcloud-aio` | Nextcloud AIO data |
| sdb3 | 2.7TB | `/mnt/pve/storage` | Immich photos, Nextcloud files |

---

## Services

### Infrastructure
| Service | Type | ID | IP | Notes |
|---------|------|----|----|-------|
| Proxmox VE | Bare metal | — | 192.168.1.108 | Hypervisor for all lab workloads |
| Pi-hole | LXC 101 | Debian 12 | 192.168.1.225 | Network-wide DNS filtering and ad blocking |
| Unbound | LXC 102 | Debian 12 | 192.168.1.84 | Recursive DNS resolver — Pi-hole upstream |
| Caddy | LXC 105 | Debian 12 | 192.168.1.107 | Reverse proxy, internal TLS for all `.lab` domains |
| WireGuard (wg-easy) | LXC 109 | Debian 12 | 192.168.1.110 | Remote VPN access via DDNS (`snoopylab23.asuscomm.com:51820`) |
| OPNsense | VM 111 | — | — | Firewall/router VM — pending managed switch for VLAN work |

### Security
| Service | Type | ID | IP | Notes |
|---------|------|----|----|-------|
| Wazuh SIEM | VM 100 | Ubuntu 22.04 | 192.168.1.140 | Full SIEM stack — manager, indexer, dashboard (v4.14.3) |

### Monitoring
| Service | Type | ID | IP | Notes |
|---------|------|----|----|-------|
| Uptime Kuma | LXC 103 | Debian 12 | 192.168.1.4 | Service uptime monitoring with Discord alerts |

### Self-Hosted Apps
| Service | Type | ID | IP | Notes |
|---------|------|----|----|-------|
| Vaultwarden | LXC 104 | Debian 12 | 192.168.1.195 | Self-hosted Bitwarden-compatible password manager |
| Jellyfin | VM 107 | Ubuntu 22.04 | 192.168.1.227 | Media server (migration from desktop Plex planned) |
| Nextcloud AIO | VM 106 | Ubuntu 22.04 | 192.168.1.56 | File storage and collaboration (v12.8.0 with built-in Collabora) |
| Immich | VM 114 | Ubuntu 22.04 | 192.168.1.219 | Self-hosted photo/video backup (Google Photos alternative) |

---

## Internal DNS + Reverse Proxy

All services are accessible via `.lab` local domains, managed by Pi-hole DNS and Caddy reverse proxy.

| Domain | Service |
|--------|---------|
| vault.lab | Vaultwarden |
| pihole.lab | Pi-hole admin |
| nextcloud.lab | Nextcloud |
| collabora.lab | Collabora (via Nextcloud AIO) |
| immich.lab | Immich |
| kuma.lab | Uptime Kuma |
| wireguard.lab | WireGuard web UI |
| jellyfin.lab | Jellyfin |
| wazuh.lab | Wazuh dashboard |

> All domains use `tls internal` (Caddy-managed self-signed certs). Migration to `*.meadows-lab.com` with Let's Encrypt wildcard cert via Cloudflare DNS challenge is in progress.

---

## Wazuh Agent Enrollment

Wazuh monitors 9 agents across the lab. All agents must match server version (4.14.3).

| Agent | ID | IP | Version | Status |
|-------|----|----|---------|--------|
| pihole-01 | 001 | 192.168.1.225 | 4.14.3 | ✅ Active |
| unbound-01 | 002 | 192.168.1.84 | 4.14.3 | ✅ Active |
| uptime-kuma-01 | 003 | 192.168.1.4 | 4.14.3 | ✅ Active |
| vaultwarden-01 | 004 | 192.168.1.195 | 4.14.3 | ✅ Active |
| caddy-01 | 005 | 192.168.1.107 | 4.14.3 | ✅ Active |
| wireguard-01 | 006 | 192.168.1.110 | 4.14.3 | ✅ Active |
| nextcloud-aio-02 | 007 | 192.168.1.56 | 4.14.0 | ✅ Active |
| jellyfin-01 | 008 | 192.168.1.227 | 4.14.3 | ✅ Active |
| immich-01 | 009 | 192.168.1.219 | 4.14.3 | ✅ Active |

---

## Backups

- Proxmox backup job runs daily at 21:00
- Retention: last 7 backups
- Target: `sdb1` (`/mnt/pve/backup-hdd`)
- Covers VMs/LXCs: 101, 102, 103, 104, 105, 107, 109, 100, 106, 114

---

## Network Diagram
![Network Diagram](diagrams/Homelab1.2.png)

---

## In Progress

- **Domain + Wildcard Cert** — `meadows-lab.com` purchased via Cloudflare. Wildcard cert via Let's Encrypt DNS challenge planned. Will migrate all `.lab` services to `.meadows-lab.com` subdomains with trusted TLS.
- **CIS Benchmark Hardening** — pihole-01 at 51%. AppArmor LXC issue pending. Rolling out to all agents after pihole-01 complete.

---

## Planned (Next Phase)

Priority ordered by resume/career impact (targeting SOC/blue team roles):

1. **Active Directory Lab** — Windows Server VM, AD DS, monitored by Wazuh. Blocked on RAM upgrade.
2. **CIS Hardening** — Complete pihole-01, roll out to remaining agents.
3. **Honeypot** — Deploy Cowrie or T-Pot, feed alerts into Wazuh.
4. **Wazuh Custom Rules** — Write detection rules, simulate attacks, document response.
5. **OPNsense + VLANs** — Pending TP-Link TL-SG108PE V3 managed switch arrival.
6. **RAM Upgrade** — 2x4GB → 2x8GB DDR3 (ordered). Unlocks Home Assistant and AD lab.
7. **Proxmox SSD Upgrade** — Kingston A400 240GB (ordered).
8. **Home Assistant** — Blocked on RAM. Can run on flat network initially.
9. **Vaultwarden Migration** — Point Bitwarden app at Vaultwarden after domain/cert setup complete.
10. **SSO (Authentik)** — Requires HTTPS/domain setup first.
11. **Update GitHub Diagram** — Reflect current lab state.

---

## Setup Docs

Detailed setup notes for each service are in the repo:

- [Pi-hole + Unbound](pihole-unbound-setup.md)
- [Uptime Kuma](uptime-kuma-setup.md)
- [WireGuard (wg-easy)](wireguard-setup.md)
- [Wazuh SIEM](wazuh-04.md)
- [Nextcloud AIO](nextcloud-setup.md)
- [Immich](immich-setup.md)
- [HDD Partition Layout](hdd-partition-layout.md)
