# Wazuh SIEM Setup — wazuh-04
**Last Updated:** 2026-03-07
**VM ID:** 100
**Hostname:** wazuh-04
**IP:** 192.168.1.140 (DHCP)
**OS:** Ubuntu 22.04 LTS
**Wazuh Version:** 4.14.3

---

## Overview
Wazuh is a self-hosted, open-source SIEM platform providing security event monitoring, log analysis, policy compliance, and vulnerability detection across all homelab services. wazuh-04 is the current stable deployment. See [Deployment History](#deployment-history) for background on previous installs.

---

## Access
| Item | Value |
|------|-------|
| Web UI (local domain) | https://wazuh.lab |
| Web UI (direct IP) | https://192.168.1.140 |
| Admin User | admin |
| Credentials | Stored in Vaultwarden |

> **Note:** Browser will show a certificate warning on direct IP access — this is expected. wazuh.lab routes through Caddy with internal TLS so no warning there.

---

## VM Specs
| Item | Value |
|------|-------|
| VM ID | 100 |
| vCPUs | 4 |
| RAM | 8GB |
| Boot Disk | 100GB (local-lvm) |

> **Note:** sdb2 (465GB) exists on the Proxmox host and was originally reserved for Wazuh data. It is no longer used for Wazuh — Proxmox VM-level snapshots and backups replace the need for an NFS snapshot repository. sdb2 is available for other uses (e.g. Nextcloud).

---

## Installation

### Step 1 — Post-install LVM expand (do immediately after Ubuntu installs)
Ubuntu only uses ~24GB by default even on a larger disk. Expand immediately:
```bash
sudo apt update && sudo apt upgrade -y
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
df -h  # verify ~97GB on /
```

### Step 2 — Run Wazuh all-in-one installer
```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

This installs three components together:
- **Wazuh Manager** — receives and processes agent data
- **Wazuh Indexer** — stores and indexes events (OpenSearch-based)
- **Wazuh Dashboard** — web UI for viewing events and alerts

> Save the admin credentials printed at the end of the install — store in Vaultwarden immediately.

---

## Caddy Reverse Proxy Config
Add to Caddyfile on caddy-01:
```
wazuh.lab {
    tls internal
    reverse_proxy https://192.168.1.140 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}
```

Reload Caddy:
```bash
sudo systemctl reload caddy
```

Add DNS record in Pi-hole: `wazuh.lab` → caddy-01 IP

---

## Index Retention Policy
Set immediately after install. Without this the indexer will fill the disk indefinitely.

**Path:** wazuh.lab → ☰ → Indexer Management → Index Management → State Management Policies → Create Policy → JSON editor

**Policy ID:** `wazuh-90-day-retention`

```json
{
  "policy": {
    "description": "Delete Wazuh alerts indices after 90 days",
    "default_state": "retention_state",
    "states": [
      {
        "name": "retention_state",
        "actions": [],
        "transitions": [
          {
            "state_name": "delete_alerts",
            "conditions": {
              "min_index_age": "90d"
            }
          }
        ]
      },
      {
        "name": "delete_alerts",
        "actions": [
          {
            "retry": {
              "count": 3,
              "backoff": "exponential",
              "delay": "1m"
            },
            "delete": {}
          }
        ],
        "transitions": []
      }
    ],
    "ism_template": [
      {
        "index_patterns": ["wazuh-alerts-*"],
        "priority": 1
      }
    ]
  }
}
```

---

## Password Requirements
The `wazuh-passwords-tool.sh` enforces strict password rules. If you need to reset the admin password:

```bash
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin -p <newpassword>
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-dashboard
```

Password must be 8–64 characters, contain upper and lowercase letters, a number, and a symbol from: `. * + ? -`

> **Note:** `!` is NOT a valid symbol for this tool. Use `.` or `-` instead.

---

## Agent Enrollment

### Important Rules
- **Never upgrade the Wazuh server to match agents.** Always reinstall the agent at the server version.
- All agents must be version **4.14.3** to match the server.
- LXCs run as root — no `sudo` needed.
- If `MANAGER_IP` is not replaced during install, fix it manually (see below).

### Pre-Enrollment — Remove Old 4.7.5 Agent
Check first:
```bash
systemctl status wazuh-agent
```

If old version present:
```bash
systemctl stop wazuh-agent
apt remove wazuh-agent -y
```

### Agent Install
```bash
# Download and install (replace <agent-name>)
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.3-1_amd64.deb
WAZUH_MANAGER='192.168.1.140' WAZUH_AGENT_NAME='<agent-name>' dpkg -i ./wazuh-agent_4.14.3-1_amd64.deb

# If MANAGER_IP wasn't set correctly, fix it:
sed -i 's/MANAGER_IP/192.168.1.140/' /var/ossec/etc/ossec.conf

# Enable and start
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

### Enrolled Agents
| Agent Name | LXC/VM ID | IP Address | Version | Status |
|------------|-----------|------------|---------|--------|
| pihole-01 | 101 | 192.168.1.225 | 4.14.3 | ✅ Active |
| unbound-01 | 102 | 192.168.1.84 | 4.14.3 | ✅ Active |
| uptime-kuma-01 | 103 | 192.168.1.4 | 4.14.3 | ✅ Active |
| vaultwarden-01 | 104 | 192.168.1.195 | 4.14.3 | ✅ Active |
| caddy-01 | 105 | 192.168.1.107 | 4.14.3 | ✅ Active |
| wireguard-01 | 109 | 192.168.1.110 | 4.14.3 | ✅ Active |
| nextcloud-01 | 112 | 192.168.1.45 | — | ⏳ Pending rebuild |
| collabora-01 | 113 | — | — | ⏳ Pending rebuild |
| jellyfin-01 | 107 | 192.168.1.227 | — | ⏳ Pending |
| opnsense-01 | 111 | — | — | ⏳ Pending |
| immich-01 | 114 | — | — | ⏳ Pending |
| backup-01 | 106 | — | — | 🗑️ To be deleted |
| ssh-01 | 108 | 192.168.1.96 | — | 🗑️ To be deleted |

---

## Uptime Kuma Monitor
- **URL:** https://wazuh.lab
- **Type:** HTTP(s)
- **TLS/SSL errors:** Ignored (self-signed cert)

---

## Backups
Wazuh is backed up via Proxmox VM snapshots and backups. No NFS snapshot repository is configured — this is intentional. Proxmox-level backups are sufficient for a homelab single-node deployment.

---

## Deployment History

| Instance | VM ID | IP | Version | Outcome |
|----------|-------|----|---------|---------|
| wazuh-01 | 110 | 192.168.1.153 | 4.7.5 | ❌ Broken during upgrade attempt to 4.14.3 |
| wazuh-02 | 110 | — | 4.13 | ❌ Install failed — 50GB disk too small |
| wazuh-03 | 115 | 192.168.1.116 | 4.14.3 | ❌ Broken when NFS used for path.data |
| wazuh-04 | 100 | 192.168.1.140 | 4.14.3 | ✅ Current stable deployment |

---

## Lessons Learned
- **100GB disk minimum** — Wazuh installer needs headroom; 50GB is not enough
- **Expand LVM immediately after Ubuntu install** — Ubuntu only mounts ~24GB by default
- **Never upgrade the server** — always reinstall agents at the server version
- **Set index retention policy before enrolling agents** — without it the indexer fills disk
- **Do not use NFS for path.data** — OpenSearch security plugin fails to initialize on NFS; use local storage for path.data
- **Proxmox backups replace NFS snapshots** — no need for a Wazuh snapshot repository in a single-node homelab
- **Password tool symbol restriction** — only `. * + ? -` are valid symbols; `!` will fail
- **l/1/I character confusion** — when saving generated passwords, verify ambiguous characters in Vaultwarden before logging out
