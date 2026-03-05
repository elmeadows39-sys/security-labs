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

> **Note:** wazuh-03 is currently broken and will be deleted. See Storage section and Lessons Learned for details. Next deployment will be wazuh-04 (VM 116).

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
| Additional Storage | sdb2 (465GB) at /mnt/pve/wazuh-data — reserved for snapshots |

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

### Pre-Enrollment — Remove Old 4.7.5 Agent
Check first:
```bash
dpkg -l | grep wazuh
```

If 4.7.5 is present, remove it:
```bash
apt remove wazuh-agent -y
apt purge wazuh-agent -y
```

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
| Agent Name | LXC/VM ID | IP Address | 4.7.5 Removed? |
|------------|-----------|------------|----------------|
| pihole-01 | 101 | 192.168.1.225 | ❌ |
| unbound-01 | 102 | 192.168.1.84 | ❌ |
| uptime-kuma-01 | 103 | 192.168.1.4 | ✅ |
| vaultwarden-01 | 104 | 192.168.1.195 | ❌ |
| caddy-01 | 105 | 192.168.1.107 | ❌ |
| wireguard-01 | 109 | 192.168.1.110 | ❌ |
| ssh-01 | 108 | 192.168.1.96 | ❌ |
| jellyfin-01 | 107 | 192.168.1.227 | ❌ |
| nextcloud-01 | 112 | 192.168.1.45 | ❌ |

---

## Storage — sdb2 (NFS Snapshot Repository)

### Background
On 2026-03-05 we attempted to point the Wazuh indexer `path.data` at an NFS mount (`/mnt/pve/wazuh-data` via sdb2). This broke OpenSearch security initialization and rendered wazuh-03 unrecoverable. Research confirmed this is a known issue — **OpenSearch does not support NFS for `path.data`**.

### Correct Approach — Use sdb2 for Snapshots Only
Keep `path.data` at its default local path (`/var/lib/wazuh-indexer`). Use the NFS mount for index snapshots/backups instead. This is the approach documented by Wazuh officially.

### Setup Steps (to complete on wazuh-04)

**On Proxmox host** — verify NFS export exists:
```
/mnt/pve/wazuh-data 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)
```

**On wazuh-04 VM** — mount the share:
```bash
sudo apt install nfs-common -y
sudo mkdir -p /mnt/wazuh-data
sudo mount 192.168.1.108:/mnt/pve/wazuh-data /mnt/wazuh-data
```

Add to `/etc/fstab`:
```
192.168.1.108:/mnt/pve/wazuh-data /mnt/wazuh-data nfs defaults 0 0
```

Fix permissions:
```bash
sudo chown -R wazuh-indexer:wazuh-indexer /mnt/wazuh-data
```

**In opensearch.yml** — add snapshot repo path (do NOT change path.data):
```yaml
path.repo: /mnt/wazuh-data
```

**In Wazuh Dashboard** — register the snapshot repository:
Indexer Management → Snapshot Management → Repositories → Create Repository → Shared file system → `/mnt/wazuh-data`

---

## Notes
- DHCP assigned IP — update Pi-hole DNS and Uptime Kuma if IP ever changes
- Wazuh Indexer (OpenSearch) uses ~5-7GB RAM at rest on an 8GB VM — normal
- Index retention policy set to 90 days rolling delete
- sdb2 (465GB) at `/mnt/pve/wazuh-data` — NFS export exists, use for snapshots only, not path.data
- Previous installs: wazuh-01 (4.7.5, VM 110) broke during upgrade; wazuh-02 (4.13, VM 110) failed install due to 50GB disk; wazuh-03 (4.14.3, VM 115) broke when NFS was used for path.data
- Uptime Kuma monitor: https://wazuh.lab with TLS/SSL errors ignored

---

## Lessons Learned
- **50GB disk is not enough** — Wazuh installer itself needs headroom, use 100GB minimum
- **Never upgrade the server** — version mismatches between agent and server should always be fixed by reinstalling the agent, not upgrading the server
- **Set index retention policy before enrolling agents** — without it the indexer fills disk indefinitely
- **Expand LVM immediately after Ubuntu install** — Ubuntu only mounts ~24GB by default even on a larger disk
- **Do not use NFS for path.data** — OpenSearch security plugin fails to initialize when the data directory is on an NFS mount; use local storage for path.data and NFS only for snapshots (path.repo)
- **Remove old 4.7.5 agents before enrolling** — uptime-kuma-01 confirmed to have had 4.7.5; assume all agents do until checked
