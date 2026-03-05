# Wazuh SIEM Setup — wazuh-03
**Date:** 2026-03-05
**VM ID:** 115
**Hostname:** wazuh-03
**IP:** 192.168.1.116 (DHCP)
**OS:** Ubuntu 22.04 LTS
**Version:** Wazuh 4.14.3

---

## Overview
Wazuh is a self-hosted, open-source SIEM platform providing security event monitoring, log analysis, policy compliance, and vulnerability detection across all homelab services. This is the third deployment (wazuh-03). Previous installs failed due to insufficient disk space (50GB) and a broken upgrade from 4.7.5 → 4.13.

---

## Access
| Item | Value |
|------|-------|
| Web UI (local domain) | https://wazuh.lab |
| Web UI (direct IP) | https://192.168.1.116 |
| Default Admin User | admin |
| Credentials | Stored in Vaultwarden |

> **Note:** Browser will show a certificate warning on direct IP access — this is expected. wazuh.lab routes through Caddy with internal TLS so no warning there.

---

## VM Specs
| Item | Value |
|------|-------|
| VM ID | 115 |
| vCPUs | 4 |
| RAM | 8GB |
| Boot Disk | 100GB (local-lvm) |
| Additional Storage | sdb2 (465GB) at /mnt/pve/wazuh-data — reserved for indexer data expansion |

---

## Installation

### Step 1 — Post-install LVM expand (do immediately after Ubuntu installs)
```bash
sudo apt update
sudo apt upgrade -y
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

Verify with:
```bash
df -h
```
Should show ~97GB on /

### Step 2 — Run Wazuh all-in-one installer
```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

This installs three components together:
- **Wazuh Manager** — receives and processes agent data
- **Wazuh Indexer** — stores and indexes events (OpenSearch-based)
- **Wazuh Dashboard** — web UI for viewing events and alerts

> Save the admin credentials printed at the end — store in Vaultwarden immediately.

---

## Caddy Reverse Proxy Config
Add to Caddyfile on caddy-01 (192.168.1.107):
```
wazuh.lab {
    tls internal
    reverse_proxy https://192.168.1.116 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}
```

Then reload Caddy:
```bash
sudo systemctl reload caddy
```

Add DNS record in Pi-hole: `wazuh.lab` → `192.168.1.107`

---

## Index Retention Policy
Set immediately after install via **wazuh.lab → Indexer Management → Index Management → State Management Policies → Create Policy (JSON)**:

Policy ID: `wazuh-rollover-policy`

```json
{
  "policy": {
    "description": "Delete Wazuh indices older than 90 days",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [],
        "transitions": [
          {
            "state_name": "delete",
            "conditions": {
              "min_index_age": "90d"
            }
          }
        ]
      },
      {
        "name": "delete",
        "actions": [
          {
            "delete": {}
          }
        ],
        "transitions": []
      }
    ]
  }
}
```

---

## Agent Enrollment

### Important Rules
- **Never upgrade the Wazuh server to match an agent.** Always reinstall the agent at the server version.
- If an agent breaks → re-enroll it. Never touch the server.
- All agents must be version **4.14.3** to match the server.
- LXCs do not have `sudo` — run commands as root directly.
- LXCs require `lsb-release` installed before the agent package.

### Agent Install Commands (per LXC/VM)
```bash
# Step 1 - install dependency (LXCs only)
apt install lsb-release -y

# Step 2 - download agent
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.3-1_amd64.deb

# Step 3 - install agent (replace <agent-name> with the agent name)
WAZUH_MANAGER='192.168.1.116' WAZUH_AGENT_NAME='<agent-name>' dpkg -i ./wazuh-agent_4.14.3-1_amd64.deb

# Step 4 - enable and start
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

### Agents to Enroll
| Agent Name | LXC/VM ID | IP Address |
|------------|-----------|------------|
| pihole-01 | 101 | 192.168.1.225 |
| unbound-01 | 102 | 192.168.1.84 |
| uptime-kuma-01 | 103 | 192.168.1.4 |
| vaultwarden-01 | 104 | 192.168.1.195 |
| caddy-01 | 105 | 192.168.1.107 |
| wireguard-01 | 109 | 192.168.1.110 |
| ssh-01 | 108 | 192.168.1.96 |
| jellyfin-01 | 107 | 192.168.1.227 |
| nextcloud-01 | 112 | 192.168.1.45 |

---

## Storage — sdb2 Mount (TODO)
sdb2 (465GB) is formatted and mounted on the Proxmox host at `/mnt/pve/wazuh-data` but not yet passed through to the VM. This should be done before the 100GB boot disk fills up.

Steps (to be documented when completed):
1. Add sdb2 as a passthrough disk to VM 115 in Proxmox
2. Format and mount inside the VM
3. Migrate Wazuh indexer data directory to the new mount
4. Update Wazuh indexer config to point to new path

---

## Notes
- DHCP assigned IP — update Pi-hole DNS and Uptime Kuma if IP ever changes
- Wazuh Indexer (OpenSearch) uses ~5-7GB RAM at rest on an 8GB VM — normal
- Index retention policy set to 90 days rolling delete
- sdb2 reserved at `/mnt/pve/wazuh-data` for future indexer data expansion
- Previous installs: wazuh-01 (4.7.5, VM 110) broke during upgrade; wazuh-02 (4.13, VM 110) failed install due to 50GB disk
- Uptime Kuma monitor: https://wazuh.lab with TLS/SSL errors ignored

---

## Lessons Learned
- **50GB disk is not enough** — Wazuh installer itself needs headroom, use 100GB minimum
- **Never upgrade the server** — version mismatches between agent and server should always be fixed by reinstalling the agent, not upgrading the server
- **Set index retention policy before enrolling agents** — without it the indexer fills disk indefinitely
- **Expand LVM immediately after Ubuntu install** — Ubuntu only mounts ~24GB by default even on a larger disk
