# Portainer CE Setup

**meadows-lab.com Homelab | March 2026**

---

## Overview

Portainer CE is a Docker management UI deployed on actual-01 (LXC 115). It provides a web interface for managing Docker containers across multiple environments (hosts) via Portainer agents.

| Component | Detail |
|-----------|--------|
| Host LXC | actual-01 (CT 115) |
| Host IP | 192.168.1.240 |
| OS | Debian 12 |
| URL | portainer.meadows-lab.com |
| Ports | 8000 (agent comms), 9443 (HTTPS UI) |
| Version | Portainer CE (latest) |

---

## Installation

### Deploy Portainer CE Container

Portainer CE is deployed alongside Actual Budget on actual-01. Docker must be installed first.

**Known issue:** Portainer behind a reverse proxy throws a `Forbidden - Origin invalid` error. The `--trusted-origins` flag is required. Deploy with:

```bash
docker run -d -p 8000:8000 -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --trusted-origins portainer.meadows-lab.com
```

> Without `--trusted-origins`, login fails with `Forbidden - Origin invalid` when Portainer is behind a reverse proxy.

---

### Caddy Reverse Proxy

Portainer uses HTTPS internally on port 9443, so the Caddy block requires `tls_insecure_skip_verify`.

Add to `/etc/caddy/Caddyfile` on caddy-01:

```
portainer.meadows-lab.com {
  reverse_proxy https://192.168.1.240:9443 {
    transport http {
      tls_insecure_skip_verify
    }
  }
}
```

Add Pi-hole DNS entry: `portainer.meadows-lab.com` → `192.168.1.107` (caddy-01)

> **Note:** Chrome may flag portainer.meadows-lab.com as a dangerous site initially. This is a false positive and clears within a few days as Chrome's Safe Browsing database updates.

---

## Portainer Agent Setup

Portainer agents allow Portainer to manage Docker on remote hosts. Install the agent on each Docker host, then add it as an environment in Portainer.

### Install Agent on a Host

Run on each Docker host (use `sudo` if not root):

```bash
docker run -d -p 9001:9001 \
  --name portainer_agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
```

### Add Environment in Portainer

1. Go to **Environments → Add environment**
2. Select **Docker Standalone → Agent**
3. Name: `<hostname>`
4. Environment URL: `<host-ip>:9001`
5. Click **Connect**

---

## Connected Environments

| Name | IP | Agent Port | Status |
|------|----|------------|--------|
| local (actual-01) | 192.168.1.240 | 9001 | ✅ Up |
| Vaultwarden | 192.168.1.195 | 9001 | ✅ Up |
| Wireguard | 192.168.1.110 | 9001 | ✅ Up |
| Immich | 192.168.1.219 | 9001 | ✅ Up |
| Nextcloud | 192.168.1.56 | 9001 | ✅ Up |

---

## Notes & Troubleshooting

- Portainer is co-located on actual-01 alongside Actual Budget.
- The `--trusted-origins` flag is required when Portainer is behind a reverse proxy — without it the UI throws `Forbidden - Origin invalid` on login.
- Portainer uses port 9443 for its HTTPS UI internally. Caddy must use `tls_insecure_skip_verify` since Portainer uses a self-signed cert internally.
- Port 8000 is the Portainer agent communication port (server-side, not browser UI).
- Agents use port 9001 on each remote host.
- Portainer data is stored in a named Docker volume (`portainer_data`) on actual-01.
