# Domain & Wildcard Cert Migration Guide
## meadows-lab.com via Let's Encrypt + Cloudflare DNS-01 Challenge

This document covers the full migration from internal `.lab` domains with self-signed certs to trusted Let's Encrypt wildcard certs using Cloudflare DNS-01 challenge, served via Caddy reverse proxy.

---

## Overview

- **Domain:** meadows-lab.com (registered via Cloudflare Registrar)
- **Cert:** Wildcard `*.meadows-lab.com` via Let's Encrypt DNS-01 challenge
- **Reverse Proxy:** Caddy (caddy-01, LXC 105, 192.168.1.107)
- **DNS:** Pi-hole (LXC 101, 192.168.1.225) for internal resolution
- **No ports exposed to internet** — Cloudflare DNS challenge only proves domain ownership, all traffic stays internal

---

## Step 1 — Domain & Cloudflare Setup

1. Register domain via [Cloudflare Registrar](https://www.cloudflare.com/products/registrar/) — at-cost pricing, free DNS.
2. In Cloudflare dashboard, create a **DNS Edit API token** scoped to your domain only:
   - Go to **My Profile → API Tokens → Create Token**
   - Use "Edit zone DNS" template
   - Scope to your specific zone (meadows-lab.com)
3. Save your **Zone ID** and **API Token** securely (Vaultwarden recommended).

---

## Step 2 — Rebuild Caddy with Cloudflare DNS Plugin

The stock apt version of Caddy does not include DNS plugins. You need to rebuild using `xcaddy`.

```bash
# Install xcaddy
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/xcaddy/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-xcaddy-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/xcaddy/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-xcaddy.list
sudo apt update && sudo apt install xcaddy -y

# Install Go (apt version too old — install manually from go.dev)
# Download latest Go tarball from https://go.dev/dl/ and install to /usr/local
# Remove any leftover apt Go binary: rm /usr/bin/go

# Build Caddy with Cloudflare DNS plugin
xcaddy build --with github.com/caddy-dns/cloudflare

# Verify Cloudflare module is present
caddy list-modules | grep cloudflare
# Should return: dns.providers.cloudflare

# Backup old binary and swap
sudo cp /usr/bin/caddy /usr/bin/caddy.bak
sudo systemctl stop caddy
sudo mv caddy /usr/bin/caddy
sudo systemctl start caddy
sudo systemctl status caddy
```

> **Note:** Build takes ~2 hours on a low-resource LXC (512MB RAM, 1 CPU). Plan accordingly.

---

## Step 3 — Cloudflare API Token in Caddy

```bash
# Create env file
sudo nano /etc/caddy/cloudflare.env
# Add: CLOUDFLARE_API_TOKEN=your_token_here

# Secure the file
sudo chmod 600 /etc/caddy/cloudflare.env
sudo chown caddy:caddy /etc/caddy/cloudflare.env

# Add EnvironmentFile to Caddy systemd service
sudo nano /lib/systemd/system/caddy.service
# Under [Service] add:
# EnvironmentFile=/etc/caddy/cloudflare.env

sudo systemctl daemon-reload
sudo systemctl restart caddy
```

> **Tip:** If you have a character ambiguity issue with the token (l vs 1 vs |), delete and recreate the token in Cloudflare dashboard.

---

## Step 4 — Caddyfile Configuration

Add the global `acme_dns` block at the top of `/etc/caddy/Caddyfile`. This tells Caddy to use Cloudflare DNS challenge globally for all cert issuance — no `tls internal` needed on new blocks.

```
{
    acme_dns cloudflare {env.CLOUDFLARE_API_TOKEN}
}
```

---

## Step 5 — Migrating Each Service

For each service, you need to:
1. Add a block to the Caddyfile on caddy-01
2. Add a DNS record in Pi-hole pointing the new domain to caddy-01 (192.168.1.107)
3. Restart Caddy and verify the cert issues

### Standard Services (HTTP backend)

```
service.meadows-lab.com {
    reverse_proxy 192.168.1.XXX:PORT
}
```

### Services with HTTPS backends (Wazuh, Proxmox)

For services that run their own HTTPS with self-signed certs internally:

```
service.meadows-lab.com {
    reverse_proxy https://192.168.1.XXX:PORT {
        transport http {
            tls_insecure_skip_verify
        }
    }
}
```

### Service Reference

| Service | New URL | Backend |
|---------|---------|---------|
| Vaultwarden | vault.meadows-lab.com | 192.168.1.195:8080 |
| Pi-hole | pihole.meadows-lab.com | 192.168.1.225:80 |
| Nextcloud | nextcloud.meadows-lab.com | 192.168.1.56:11000 |
| Uptime Kuma | kuma.meadows-lab.com | 192.168.1.4:3001 |
| WireGuard | wireguard.meadows-lab.com | 192.168.1.110:51821 |
| Jellyfin | jellyfin.meadows-lab.com | 192.168.1.227:8096 |
| Wazuh | wazuh.meadows-lab.com | https://192.168.1.140:443 |
| Immich | immich.meadows-lab.com | 192.168.1.219:2283 |
| Proxmox | proxmox.meadows-lab.com | https://192.168.1.108:8006 |

---

## Service-Specific Notes

### Nextcloud AIO
Nextcloud AIO manages domain config via `mastercontainer configuration.json` — manual `config.php` edits are overwritten on container restart.

1. Edit domain in `configuration.json` on the Nextcloud VM
2. Stop all containers via AIO panel (`http://[NC-IP]:8080`)
3. Restart — all containers come up with new domain, Collabora updates automatically

### Pi-hole
Pi-hole redirects root `/` to `/admin/login` — this returns a 403 on the root URL which is normal. Bookmark `/admin/login` directly.

### Proxmox
- Works correctly via new URL
- Chrome may block click input on TOTP field — use **Tab** key to focus the field instead

### Backward Compatibility
Keep old `.lab` blocks in Caddyfile alongside new `.meadows-lab.com` blocks until migration is confirmed complete, then remove old blocks and Pi-hole DNS entries.

---

## How It Works

```
Browser → Pi-hole DNS (resolves *.meadows-lab.com to 192.168.1.107)
       → caddy-01 (terminates TLS, reverse proxies to backend)
       → Backend service
```

Cloudflare's only role is proving domain ownership during cert issuance via DNS-01 challenge. It is not involved in day-to-day traffic. All traffic stays on the internal network.

Let's Encrypt issues the actual certificate. Caddy handles renewal automatically.
