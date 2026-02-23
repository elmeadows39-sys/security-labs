# Uptime Kuma Setup
**Date:** 2026-02-22
**LXC ID:** 103
**Hostname:** uptime-kuma-01
**IP:** 192.168.1.4:3001
**OS:** Debian 12

---

## Overview
Uptime Kuma is a self-hosted monitoring tool that watches homelab services and sends 
alerts via Discord when something goes down or comes back up.

---

## Access
| Item | Value |
|------|-------|
| Web UI | http://(your IP):3001 |

---

## Active Monitors
| Service | Type | Address | Interval |
|---------|------|---------|----------|
| Pi-hole | HTTP | http://192.168.1.225/admin | 60s |
| Unbound | TCP | 192.168.1.x:5335 | 60s |
| Vaultwarden | HTTP | http://192.168.1.x | 60s |
| WireGuard (wg-easy) | TCP | 192.168.1.110:51821 | 60s |

---

## Notifications
Discord webhook configured. Alerts sent to #general in Homelab Discord server.
All monitors have Discord notification enabled by default.

To add a new notification channel:
1. Go to **Settings > Notifications > Add Notification**
2. Select type (Discord, Email, etc.)
3. Fill in details and click **Test**
4. Enable **Apply on all existing monitors** if desired

---

## Adding a New Monitor
1. Click **+ Add New Monitor**
2. Select type — use **HTTP(s)** for web UIs, **TCP Port** for services without HTTP
3. Fill in name, URL/hostname, port, and interval
4. Assign the Discord notification
5. Click **Save**

> **Note:** Use TCP Port instead of HTTP for services that require login — 
> HTTP monitors may show red if they get a non-200 response from a login page.

---

## Password Reset
Uptime Kuma has no password reset in the UI. To reset via command line:
```bash
cd /opt/uptime-kuma
node extra/reset-password.js
```

---

## Notes
- Each monitor can have notifications enabled/disabled individually
- Discord webhook URL is stored in Settings > Notifications
- Store the Discord webhook URL in Vaultwarden for safekeeping
