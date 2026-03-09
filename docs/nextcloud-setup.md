# Nextcloud — Setup History & Current State

## Current Setup (as of March 8, 2026)

**VM:** nextcloud-aio-02 (ID: 106)
**OS:** Ubuntu 22.04
**IP:** 192.168.1.56
**Access:** https://nextcloud.lab (via Caddy reverse proxy)

### Stack
- **Nextcloud AIO v12.8.0** — Docker-based all-in-one deployment
- **Nextcloud Office (Collabora)** — built into AIO, no separate LXC needed
- **PostgreSQL, Redis, Apache** — all managed by AIO internally
- **Caddy (caddy-01 at 192.168.1.107)** — handles TLS and reverse proxy
- **DNS** — nextcloud.lab → 192.168.1.107 in Pi-hole

### Storage
- **Boot disk:** 32GB on local-lvm (OS + Docker)
- **Data directory:** `/mnt/nextcloud-aio` — NFS mount from Proxmox host
  - Proxmox host exports `/mnt/pve/nextcloud-aio` (sdb2, 434GB free)
  - Mounted in VM via `/etc/fstab`: `192.168.1.108:/mnt/pve/nextcloud-aio /mnt/nextcloud-aio nfs defaults 0 0`
  - AIO configured with `NEXTCLOUD_DATADIR="/mnt/nextcloud-aio"`

### Docker Run Command
```
sudo docker run -d --name nextcloud-aio-mastercontainer --restart always \
  -p 8080:8080 \
  --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  --volume /var/run/docker.sock:/var/run/docker.sock \
  -e APACHE_PORT=11000 \
  -e APACHE_IP_BINDING=0.0.0.0 \
  -e SKIP_DOMAIN_VALIDATION=true \
  -e NEXTCLOUD_DATADIR="/mnt/nextcloud-aio" \
  nextcloud/all-in-one:latest
```

### Caddy Config (nextcloud.lab block)
```
nextcloud.lab {
    tls internal
    reverse_proxy 192.168.1.56:11000 {
        header_up Host {host}
        header_up X-Forwarded-Proto {scheme}
        header_up X-Forwarded-For {remote}
    }
}
```

### AIO Admin Panel
- Accessible at: https://192.168.1.56:8080
- Always use the local IP for AIO admin, not the domain

---

## Retired: nextcloud-01 (LXC 112) + collabora-01 (LXC 113)

### What they were
- **nextcloud-01:** Debian LXC, manual Nextcloud install on NC33, data on sdb3
- **collabora-01:** Debian LXC, manually installed coolwsd + Collabora CODE

### Why they were retired
After multiple sessions (March 2–5, 2026), we were never able to get Nextcloud Office (document editing in the browser) working reliably. Issues encountered:

- richdocuments 8.7.3 incompatible with NC33 — had to manually edit `appinfo/info.xml` to force-enable it
- Appdata directory permission errors blocking app store and theming
- Collabora WOPI callback errors — coolwsd couldn't reach Nextcloud via domain due to SSL verification failures on internal self-signed certs
- WebSocket connection failures when Collabora tried to connect back to the browser
- Caddy TLS transport config causing SSL handshake errors between Caddy and Collabora's plain HTTP port

After exhausting multiple approaches (coolwsd.xml edits, Caddy header configs, trusted domain fixes, WOPI host whitelisting), the decision was made to scrap both LXCs and migrate to **Nextcloud AIO**, which bundles Collabora internally and avoids all the manual integration complexity.

### What worked on nc-01
- File uploads and storage on sdb3 worked correctly
- nextcloud.lab domain + Caddy reverse proxy worked
- Pi-hole DNS resolution worked
- Basic Nextcloud functionality (non-Office) worked fine

---

## Todo / Next Steps
- [ ] Migrate files from OneDrive → Nextcloud
- [ ] Point Bitwarden app at Vaultwarden (after domain purchase)
- [ ] Add to Wazuh monitoring
- [ ] Add to Uptime Kuma
- [ ] Set static IP for nextcloud-aio-02
