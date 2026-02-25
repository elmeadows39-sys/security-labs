# Wazuh SIEM Setup
**Date:** 2026-02-24
**VM ID:** 110
**Hostname:** wazuh-01
**IP:** 192.168.1.153
**OS:** Ubuntu 22.04 LTS

---

## Overview
Wazuh is a self-hosted, open-source SIEM platform providing security event monitoring, log analysis, policy compliance, and vulnerability detection across all homelab services. Replaces the previous log-ubuntu VM (VM 100), which was deleted after Wazuh was deployed.

---

## Access
| Item | Value |
|------|-------|
| Web UI (Dashboard) | https://192.168.1.153 |
| Default Admin User | admin |
| Credentials | Stored in Vaultwarden |

> **Note:** Browser will show a certificate warning on first access — this is expected. Click Advanced > Proceed to continue.

---

## VM Specs
| Item | Value |
|------|-------|
| vCPUs | 4 |
| RAM | 8GB |
| Boot Disk | 50GB (local-lvm) |
| Additional Storage | sdb2 (465GB) reserved at /mnt/pve/wazuh-data for future indexer data expansion |

---

## Installation
Deployed using the Wazuh all-in-one installer (single-node):

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh && sudo bash wazuh-install.sh -a
```

This installs three components together:
- **Wazuh Manager** — receives and processes agent data
- **Wazuh Indexer** — stores and indexes events (OpenSearch-based)
- **Wazuh Dashboard** — web UI for viewing events and alerts

---

## Active Agents
All agents installed using DEB amd64 package. Each LXC required `lsb-release` to be installed first and does not have `sudo` — run dpkg commands directly as root.

| Agent Name | LXC ID | IP Address |
|------------|--------|------------|
| pihole-01 | 101 | 192.168.1.225 |
| unbound-01 | 102 | 192.168.1.84 |
| uptime-kuma-01 | 103 | 192.168.1.4 |
| vaultwarden-01 | 104 | 192.168.1.195 |
| caddy-01 | 105 | 192.168.1.107 |
| wireguard-01 | 109 | 192.168.1.110 |
| ssh-01 | 108 | 192.168.1.96 |

### Agent Install Commands (per LXC)
```bash
# Step 1 - install dependency
apt install lsb-release -y

# Step 2 - download and install agent (replace NAME with agent name)
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.5-1_amd64.deb
WAZUH_MANAGER='192.168.1.153' WAZUH_AGENT_NAME='<agent-name>' dpkg -i ./wazuh-agent_4.7.5-1_amd64.deb

# Step 3 - enable and start
systemctl daemon-reload && systemctl enable wazuh-agent && systemctl start wazuh-agent
```

---

## Features in Use
- **Security Events** — real-time log ingestion and alerting across all agents
- **SCA (Security Configuration Assessment)** — automatic CIS Benchmark scanning of all LXCs
- **MITRE ATT&CK mapping** — detected events mapped to threat framework
- **Policy Monitoring** — compliance checks against GDPR, HIPAA, PCI-DSS standards

---

## Notes
- Wazuh Indexer (Elasticsearch) is memory-intensive — expect ~5GB RAM usage at rest on an 8GB VM
- sdb2 partition reserved at `/mnt/pve/wazuh-data` for future migration of indexer data directory if 50GB boot disk fills up
- VM 100 (log-ubuntu-01) was deleted after Wazuh deployment — it had no active log shipping configured
- Qemu Guest Agent installed — IP visible in Proxmox summary

---

## Future Improvements
- Migrate Wazuh indexer data directory to sdb2 (465GB) as log volume grows
- Complete CIS benchmark hardening on all LXCs based on SCA findings
- Customize Wazuh dashboard views and alert thresholds
- Add VM 107 (jellyfin-01) agent once Jellyfin setup is complete
