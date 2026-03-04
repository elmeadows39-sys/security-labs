# Immich Setup

Self-hosted photo and video backup (Google Photos alternative) running on Docker in a Proxmox VM, with media stored on a shared NFS drive.

---

## Infrastructure

| Component | Value |
|-----------|-------|
| VM ID | 114 |
| Hostname | immich-01 |
| IP | 192.168.1.219 |
| OS | Ubuntu 22.04 LTS |
| CPU | 2 cores |
| RAM | 4GB |
| Boot Disk | 32GB (local-lvm) |
| Media Storage | /mnt/pve/storage/immich (via NFS from Proxmox) |
| Domain | https://immich.lab |
| Port | 2283 |

---

## VM Creation (Proxmox)

1. Create VM with above specs
2. Check **Start at boot** and **Qemu Agent**
3. Storage: local-lvm, 32GB
4. Install Ubuntu 22.04 LTS

---

## Post-Install (Ubuntu)

### Expand LVM (do this immediately after install)

```bash
sudo apt update && sudo apt upgrade -y
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

### Install Docker

```bash
sudo apt install docker.io ca-certificates curl gnupg -y
```

Add Docker's official repo for docker-compose-plugin:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu jammy stable" | sudo tee /etc/apt/sources.list.d/docker.list

sudo apt update && sudo apt install docker-compose-plugin -y
```

---

## Immich Installation

```bash
mkdir immich && cd immich
wget https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

Edit `.env`:

```bash
nano .env
```

Set the following:

```
UPLOAD_LOCATION=/mnt/storage/immich
DB_PASSWORD=yourchosenpassword
```

---

## NFS Storage Setup

### On Proxmox host

Install NFS server:

```bash
apt install nfs-kernel-server -y
```

Create Immich folder on sdb3:

```bash
mkdir /mnt/pve/storage/immich
```

Add export:

```bash
nano /etc/exports
```

Add line:

```
/mnt/pve/storage 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)
```

Apply:

```bash
exportfs -a && systemctl restart nfs-kernel-server
```

### On Immich VM

Install NFS client and mount:

```bash
sudo apt install nfs-common -y
sudo mkdir -p /mnt/storage
sudo mount 192.168.1.108:/mnt/pve/storage /mnt/storage
```

Make permanent in fstab:

```bash
sudo nano /etc/fstab
```

Add line:

```
192.168.1.108:/mnt/pve/storage /mnt/storage nfs defaults 0 0
```

Create required Immich subdirectories and marker files:

```bash
sudo mkdir -p /mnt/storage/immich/{library,upload,thumbs,profile,backups,encoded-video}
sudo touch /mnt/storage/immich/library/.immich /mnt/storage/immich/upload/.immich /mnt/storage/immich/thumbs/.immich /mnt/storage/immich/profile/.immich /mnt/storage/immich/backups/.immich /mnt/storage/immich/encoded-video/.immich
```

---

## Start Immich

```bash
sudo docker compose up -d
```

Verify all containers are running:

```bash
sudo docker compose ps
```

Expected containers: `immich_server`, `immich_postgres`, `immich_redis`, `immich_machine_learning`

---

## DNS + Caddy

### Pi-hole DNS

Add local DNS record:
- Domain: `immich.lab`
- IP: `192.168.1.107` (Caddy)

### Caddyfile

```
immich.lab {
    tls internal
    reverse_proxy 192.168.1.219:2283
}
```

Reload Caddy:

```bash
systemctl reload caddy
```

---

## Notes

- Media is stored on sdb3 at `/mnt/pve/storage/immich` — same drive as Nextcloud data
- Immich requires `.immich` marker files in each subdirectory or it will fail to start
- Live Photos (iPhone) are supported — Immich stores both the JPEG and the MP4 automatically
- Mobile app sync not yet configured — manual upload via web UI for now
