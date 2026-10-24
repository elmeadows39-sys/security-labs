# security-labs

This repository documents a home network and homelab platform built for hands-on learning in infrastructure, security operations, and system administration — with a focus on resume-building projects targeting SOC/blue team roles.

The network edge is a dedicated **OPNsense firewall on bare-metal hardware (HP Z240 SFF)** routing the household LAN and three segmented VLANs. Lab workloads run on **Proxmox VE**, whose host now lives inside the Trusted VLAN alongside the services it hosts. VLAN 10 (Trusted) hosts all lab services. VLAN 20 (IoT) isolates smart-home devices. VLAN 30 (DMZ) hosts the honeypot and is isolated in both directions. Services are deployed as Linux containers (LXCs) or VMs depending on workload requirements.

---

## Current Lab State

### Hardware

| Component | Details |
|-----------|---------|
| Edge Firewall/Router | HP Z240 SFF — OPNsense on bare metal (ZFS), Intel I340 4-port NIC (igb). WAN direct to the ISP fiber ONT; LAN trunk carries the household network plus VLANs 10/20/30 |
| Hypervisor | Proxmox VE — bare metal (lab-pve, 192.168.10.5 on VLAN 10) |
| RAM | 32GB DDR3 (board maximum) |
| Storage | 4TB external HDD (/dev/sdc) — partitioned for backups, Nextcloud, and general storage; mounted by UUID |
| Switch | TP-Link TL-SG108PE V3 (managed, PoE) — 802.1Q trunks to OPNsense and Proxmox, dedicated IoT access port |
| Wireless (Household) | TP-Link Archer AX1300 in access-point mode — bridges Wi-Fi onto the household LAN; channels fixed to avoid overlap |
| Wireless (IoT) | Netgear R6250 in AP mode — broadcasts the isolated IoT network (VLAN 20) |
| Rack | 15U open-frame rack — firewall, switch and access points on a dedicated surge-protected strip |

### Storage Layout (4TB External HDD)

| Partition | Size | Mount | Purpose |
|-----------|------|-------|---------|
| sdc1 | 465GB | /mnt/pve/backup-hdd | Proxmox VM/LXC backups |
| sdc2 | 465GB | /mnt/pve/nextcloud-aio | Nextcloud AIO data (NFS to nextcloud-aio-02) |
| sdc3 | 2.7TB | /mnt/pve/storage | Immich photos, general storage (NFS to immich-01) |

### Network Layout

| Network | Subnet | Gateway | Purpose |
|---------|--------|---------|---------|
| LAN (household) | 192.168.1.0/24 | 192.168.1.1 | Household devices and Wi-Fi — Kea DHCP on OPNsense |
| VLAN 10 (Trusted) | 192.168.10.0/24 | 192.168.10.1 | All lab services and the Proxmox host |
| VLAN 20 (IoT) | 192.168.20.0/24 | 192.168.20.1 | Smart TVs, thermostat, cameras — internet only, plus a narrow rule letting the TVs reach Plex |
| VLAN 30 (DMZ) | 192.168.30.0/24 | 192.168.30.1 | Honeypot — internet only; blocked from every private network and unreachable from the LAN and Trusted networks |

All household and IoT DNS runs through **Pi-hole → Unbound** (recursive, with serve-expired caching). The firewall's WAN blocks private and bogon networks and accepts only WireGuard inbound.

### Firewall Policy Highlights

- **DMZ isolation (both directions):** the DMZ can't reach any private network, and LAN/Trusted can't reach the DMZ — with a single pass rule so Uptime Kuma can still health-check the honeypot.
- **IoT isolation:** IoT devices get DNS (Pi-hole only) and internet. One aliased rule (`IoT_TVs`) lets the TVs reach the Plex server; a reject rule makes them fail fast instead of timing out against the public address.
- **Remote access:** WireGuard only — dynamic DNS kept current by OPNsense (Cloudflare API, scoped token), one port forward to the VPN container.
- **Config backups:** OPNsense automatically uploads its configuration to Nextcloud on every change.
- **Gateway monitoring:** WAN quality (latency/loss) is recorded continuously for outage investigation.

---

## Services

### Infrastructure

| Service | Type | OS | IP | Notes |
|---------|------|----|----|-------|
| Proxmox VE | Bare metal | — | 192.168.10.5 | Hypervisor for all lab workloads — host moved onto VLAN 10 |
| OPNsense | Bare metal (HP Z240) | FreeBSD | 192.168.1.1 / 192.168.10.1 / .20.1 / .30.1 | Edge firewall/router, Kea DHCP for all networks, DDNS, WireGuard port forward, Wazuh agent |
| Pi-hole | LXC 101 | Debian 12 | 192.168.10.110 | Network-wide DNS filtering and ad blocking — DNS server for the whole household |
| Unbound | LXC 102 | Debian 12 | 192.168.10.30 | Recursive DNS resolver — Pi-hole upstream, serve-expired enabled |
| Caddy | LXC 105 | Debian 12 | 192.168.10.20 | Reverse proxy with Let's Encrypt wildcard cert via Cloudflare DNS challenge |
| WireGuard (wg-easy) | LXC 109 | Debian 12 | 192.168.10.50 | Remote VPN access — working, endpoint kept current by dynamic DNS |

### Security

| Service | Type | OS | IP | Notes |
|---------|------|----|----|-------|
| Wazuh SIEM | VM 100 | Ubuntu 22.04 | 192.168.10.80 | Full SIEM stack — manager, indexer, dashboard (v4.14.8), Discord alerts at level 10+ |
| Cowrie Honeypot | LXC 108 | Debian 12 | 192.168.30.10 | SSH honeypot on port 22 via iptables redirect — logs feed into Wazuh via custom rules; public DNS, isolated in the DMZ |

### Monitoring

| Service | Type | OS | IP | Notes |
|---------|------|----|----|-------|
| Uptime Kuma | LXC 103 | Debian 12 | 192.168.10.10 | 15 monitors with Discord alerts, including the Proxmox host |
| Homepage | Docker on LXC 110 | Debian 12 | 192.168.10.40 | Lab dashboard in tabs (Lab / Health / Calendar) — Kuma, Wazuh alert count + latest alert, Proxmox, swap, backups, storage, Pi-hole, OPNsense, calendar, bills; each panel through a dedicated read-only account or token |

### Self-Hosted Apps

| Service | Type | OS | IP | Notes |
|---------|------|----|----|-------|
| Vaultwarden | LXC 104 | Debian 12 | 192.168.10.11 | Self-hosted Bitwarden-compatible password manager — the only password manager in the household |
| Nextcloud AIO | VM 106 | Ubuntu 22.04 | 192.168.10.90 | Files, calendar, tasks and Office — calendar and task lists sync to iOS over CalDAV |
| Immich | VM 114 | Ubuntu 22.04 | 192.168.10.70 | Self-hosted photo/video backup (Google Photos alternative), with ImmichFrame slideshow |
| Actual Budget | LXC 110 | Debian 12 | 192.168.10.40 | Personal finance and bill tracking (port 5006) |
| Bills ping | Python + systemd on LXC 110 | Debian 12 | 192.168.10.40 | 7 AM push (Pushover, ntfy backup) of bills due today/tomorrow, read from Actual through actual-http-api; systemd retry on transient failures |
| Health trackers | Python (stdlib) on LXC 110 | Debian 12 | 192.168.10.40 | Small self-written weight, exercise and step tracker apps with SQLite + JSON APIs; steps pushed from Apple Health by an iPhone Shortcut |
| Portainer CE | LXC 110 | Debian 12 | 192.168.10.40 | Docker management UI (port 9443) |
| Jellyfin | VM 107 | Ubuntu 22.04 | 192.168.10.60 | Media server (currently powered off — deprioritized) |

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
| weight.meadows-lab.com | Weight + exercise tracker |
| steps.meadows-lab.com | Step tracker |

---

## Wazuh Agent Enrollment

Wazuh monitors 12 active agents across the lab. Linux agents run v4.14.8 and are version-held so they never move ahead of the manager.

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
| jellyfin-02 | 013 | 192.168.10.60 | — | ⏸️ Powered off |
| actual-01 | 017 | 192.168.10.40 | 4.14.8 | ✅ Active |
| lab-pve | 018 | 192.168.10.5 | 4.14.8 | ✅ Active |
| opnsense-01 (Z240) | 019 | 192.168.10.1 | 4.14.5 (plugin) | ✅ Active |

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
- Covers all VMs/LXCs
- OPNsense configuration: automatic upload to Nextcloud on every change, plus dated manual save points before major changes
- Switch and access point configs: dated backups kept in Nextcloud

---

## Network Diagram

![Network Diagram](diagrams/Diagram%203.0.png)

---

## Recently Completed

- **Dedicated OPNsense Hardware** — HP Z240 replaced the OPNsense VM as the edge router, removing the double NAT and the firewall's dependency on the hypervisor. ([write-up](docs/z240-edge-cutover.md))
- **Proxmox Host off the Flat Network** — lab-pve moved onto VLAN 10; NFS now stays inside one VLAN with no routing.
- **WireGuard Remote Access** — restored with OPNsense-managed dynamic DNS and a single port forward. ([write-up](docs/wireguard-setup.md))
- **IoT Device Migration** — both TVs and the thermostat on VLAN 20 with DHCP reservations and an alias-based Plex rule.
- **DMZ Isolation** — LAN and Trusted blocked from the DMZ, with a monitoring-only exception.

## In Progress

- **OPNsense 26.7 Upgrade** — the 26.1 series is end of life; the upgrade also closes a critical PHP remote-code-execution advisory.
- **Cowrie Internet Exposure** — WAN port forward → cowrie-01 (192.168.30.10:2222) after the 26.7 upgrade, then analysis of real outside attacks.
- **Suricata → Wazuh Integration** — previously blocked on the VM (virtio NICs generated zero alerts, upstream issue #10097). Now unblocked on the Z240's Intel NICs; scheduled after the 26.7 upgrade.
- **Lab Dashboard** — next panels: calories (Apple Health → lab) and tasks.

## Planned

- **VLAN Migration Template** — Documenting the standard LXC/VM migration process including PAM fix, netplan config, Caddy/Pi-hole/Uptime Kuma update steps.
- **Proxmox Cluster** — second node (refurbished mini PC) plus a QDevice tiebreaker for more RAM and failover.
- **Proxmox SSD Upgrade** — Larger SSD for the VM storage pool to resolve snapshot space limitations.
- **CIS Benchmark Hardening** — caddy-01 (54%), wireguard-01, uptime-kuma-01, actual-01 hardened; remaining hosts deferred.

---

## Setup Docs

### Infrastructure
- [Pi-hole + Unbound](docs/pihole-unbound-setup.md)
- [WireGuard (wg-easy)](docs/wireguard-setup.md)
- [HDD Partition Layout](docs/hdd-partition-layout.md)
- [Domain + Wildcard Cert Migration](docs/domain-cert-migration.md)
- [Portainer](docs/portainer-setup.md)
- [IoT VLAN 20 — Isolated Wi-Fi for Smart Devices](docs/iot-vlan20-build.md)
- [Z240 Edge Cutover — OPNsense on Dedicated Hardware](docs/z240-edge-cutover.md)
- VLAN Migration Template *(coming soon)*

### Security
- [Wazuh SIEM](docs/wazuh-04.md)
- [Cowrie Honeypot](docs/cowrie-setup.md)
- [Active Directory Attack Lab — Brute Force & Kerberoasting](docs/ad-kerberoasting-lab.md)
- [VirusTotal FIM Integration](docs/virustotal-fim-integration.md)
- [Router Replacement & OPNsense Routing Fix](docs/router-replacement-opnsense-routing.md)
- [Z240 Cutover Attempt & Edge Router Failure](docs/z240-cutover-and-edge-router-failure.md)

### Monitoring & Apps
- [Uptime Kuma](docs/uptime-kuma-setup.md)
- [Nextcloud AIO](docs/nextcloud-setup.md)
- [Immich](docs/immich-setup.md)
