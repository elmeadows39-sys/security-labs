# WireGuard VPN Setup

**Date:** 2025-02-20  
**LXC ID:** 109  
**Hostname:** wireguard-01  
**IP:** 192.168.1.110  
**OS:** Debian 12

---

## Overview

WireGuard VPN server running in a Proxmox LXC. Allows remote access to the homelab from outside the network using DDNS. DNS is routed through Pi-hole for ad blocking even on mobile data.

---

## Network Details

| Item | Value |
|------|-------|
| LXC IP | 192.168.1.110 |
| VPN Subnet | 10.0.0.0/24 |
| Server VPN IP | 10.0.0.1 |
| Listen Port | 51820 (UDP) |
| DDNS Endpoint | snoopylab23.asuscomm.com:51820 |
| DNS (Pi-hole) | 192.168.1.225 |

---

## Installation

```bash
apt update && apt upgrade -y
apt install -y wireguard wireguard-tools
```

---

## Server Key Generation

```bash
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey
chmod 600 /etc/wireguard/privatekey
```

---

## Server Config `/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <server private key>

[Peer]
# Phone
PublicKey = <phone public key>
AllowedIPs = 10.0.0.2/32
```

---

## Enable and Start

```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

---

## Adding a New Peer (Client)

1. Generate a key pair for the new device:

```bash
wg genkey | tee /etc/wireguard/<device>_privatekey | wg pubkey > /etc/wireguard/<device>_publickey
```

2. Add a `[Peer]` block to `/etc/wireguard/wg0.conf`:

```ini
[Peer]
PublicKey = <device public key>
AllowedIPs = 10.0.0.X/32
```

3. Create a client config `/etc/wireguard/<device>.conf`:

```ini
[Interface]
PrivateKey = <device private key>
Address = 10.0.0.X/24
DNS = 192.168.1.225

[Peer]
PublicKey = <server public key>
Endpoint = snoopylab23.asuscomm.com:51820
AllowedIPs = 0.0.0.0/0
```

4. Generate QR code for mobile:

```bash
apt install -y qrencode
qrencode -t ansiutf8 -s 1 < /etc/wireguard/<device>.conf
```

5. Restart WireGuard to apply changes:

```bash
systemctl restart wg-quick@wg0
```

---

## Peer IP Assignments

| Device | VPN IP |
|--------|--------|
| Server | 10.0.0.1 |
| Phone  | 10.0.0.2 |

---

## Router Port Forwarding (ASUS RT-AC68U)

| Field | Value |
|-------|-------|
| Service Name | WireGuard |
| Protocol | UDP |
| External Port | 51820 |
| Internal Port | 51820 |
| Internal IP | 192.168.1.110 |

---

## Verify Connection

```bash
wg show
```

Look for `latest handshake` under the peer — if it's recent, the tunnel is active and working.

---

## Notes

- Each device gets its own key pair — never share private keys
- Server uses `/24` for the interface, peers use `/32` (one specific IP each)
- `AllowedIPs = 0.0.0.0/0` on the client routes ALL traffic through the VPN (full tunnel)
- DNS set to Pi-hole so ad blocking works even on mobile data
