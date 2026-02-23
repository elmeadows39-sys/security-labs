# WireGuard VPN Setup (wg-easy)
**Date:** 2026-02-21  
**LXC ID:** 109  
**Hostname:** wireguard-01  
**IP:** 192.168.1.110  
**OS:** Debian 12

---

## Overview
WireGuard VPN running via wg-easy in Docker. Provides remote access to the homelab from outside 
the network using DDNS. DNS is routed through Pi-hole for ad blocking even on mobile data.

> **Note:** We started with a manual WireGuard install but replaced it with wg-easy for easier 
> peer management via web UI. We also hit a Docker networking issue — see Troubleshooting section.

---

## Network Details
| Item | Value |
|------|-------|
| LXC IP | 192.168.1.110 |
| VPN Subnet | 10.8.0.0/24 |
| Server VPN IP | 10.8.0.1 |
| Listen Port | 51820 (UDP) |
| Web UI Port | 51821 (TCP) |
| DDNS Endpoint | snoopylab23.asuscomm.com:51820 |
| DNS (Pi-hole) | 192.168.1.225 |
| Web UI | http://192.168.1.110:51821 |

---

## Peer IP Assignments
| Device | VPN IP |
|--------|--------|
| Server | 10.8.0.1 |
| Phone  | 10.8.0.2 |
| Laptop | 10.8.0.3

---

## Prerequisites
Docker must be installed on the LXC. The LXC also needs nesting enabled in Proxmox 
(Options > Features > Nesting) for Docker to run properly.

---

## Installation

### 1. Install Docker
```bash
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | sh
```

### 2. Run wg-easy
```bash
docker run -d \
  --name=wg-easy \
  --network host \
  --cap-add=NET_ADMIN \
  --cap-add=SYS_MODULE \
  -e WG_HOST=snoopylab23.asuscomm.com \
  -e PASSWORD_HASH='<your-bcrypt-hash>' \
  -e WG_DEFAULT_DNS=192.168.1.225 \
  -v ~/.wg-easy:/etc/wireguard \
  --restart unless-stopped \
  ghcr.io/wg-easy/wg-easy
```

> **Important:** `--network host` is required. Without it, the wg0 interface gets created inside 
> the Docker network namespace instead of the LXC, breaking all routing.

---

## Adding a New Peer (Client)
1. Go to `http://192.168.1.110:51821`
2. Log in with the admin password
3. Click **+ New Client** and give it a name
4. Scan the QR code with the WireGuard app on your device
5. Enable the tunnel and test

---

## Split Tunnel Configuration (Recommended for Laptop)
By default, wg-easy assigns Allowed IPs: 0.0.0.0/0 to new clients, which routes 
all 
traffic through the tunnel. This will kill internet access if you're already on 
the home LAN.

For the laptop peer, open the WireGuard app on Windows, click Edit on the tunnel, and change Allowed IPs to:

10.8.0.0/24, 192.168.1.0/24

This routes only homelab traffic through the tunnel and lets normal internet go out directly. Use this when connected at home. When away (e.g. at school on a different network), the default 0.0.0.0/0 is fine if you want Pi-hole DNS coverage on all traffic.

To disconnect when not needed: leave the peer enabled in the wg-easy UI but simply deactivate the tunnel in the WireGuard Windows app. It will be ready to reconnect with one click.

---

## Router Port Forwarding (ASUS RT-AC68U)
| Field | Value |
|-------|-------|
| Protocol | UDP |
| External Port | 51820 |
| Internal Port | 51820 |
| Internal IP | 192.168.1.110 |

---

## Verify Connection
```bash
wg show
```
Look for `latest handshake` under the peer — if it's recent, the tunnel is active. 
Also check transfer bytes are increasing while browsing.

---

## Persistence
nftables rules are saved to `/etc/nftables.conf` and load on boot:
```bash
systemctl enable nftables
```
The Docker container restarts automatically via `--restart unless-stopped`.

---

## Troubleshooting

### Problem: Nothing loads through the tunnel
**Root cause:** Two issues combined:
1. Running wg-easy without `--network host` puts wg0 inside Docker's network namespace — 
   the LXC can't route traffic through it
2. Docker sets a DROP policy on the nftables FORWARD chain, blocking all WireGuard traffic

**Fix:**
Recreate the container with `--network host` (no -p port mappings needed), then add nft rules:
```bash
nft insert rule ip filter FORWARD iifname "wg0" accept
nft insert rule ip filter FORWARD oifname "wg0" accept
```
Save rules:
```bash
nft list ruleset > /etc/nftables.conf
systemctl enable nftables
```

### Problem: iptables rules not taking effect
This system uses both `iptables` (nftables backend) and `iptables-legacy`. 
Rules added with `iptables` may not be seen by the kernel. Use `nft` commands directly instead.

### Confirm traffic is using the tunnel
```bash
wg show
```
Check the endpoint IP — it should be your mobile carrier's IP, not your home IP, 
confirming traffic is coming in over mobile data through the tunnel.
