# 4TB External HDD — Partition Layout
**Device:** `/dev/sdb` (4TB External HDD attached to Proxmox host)  
**Host:** lab-pve (192.168.1.108)  
**Documented:** 2026-02-25  
**Last Updated:** 2026-03-08

---

## Overview
The 4TB drive is partitioned into three sections with dedicated purposes. It replaced the previous backup configuration which was accidentally writing to the Proxmox main system drive (pve-root). Migrating to this drive freed ~29GB on the boot disk.

---

## Partition Layout

| Partition | Size | Mount Point | Purpose |
|-----------|------|-------------|---------|
| sdb1 | 465GB | `/mnt/pve/backup-hdd` | Proxmox VM/LXC backups |
| sdb2 | 465GB | `/mnt/pve/nextcloud-aio` | Reserved for Nextcloud AIO storage |
| sdb3 | 2.7TB | `/mnt/pve/storage` | General storage (Immich, Nextcloud data) |

---

## Backup Configuration (sdb1)
- Proxmox backup job runs daily at 21:00
- Retention policy: keep-last=7
- VMs and LXCs included: 101, 102, 103, 104, 105, 107, 109, 100, 112, 113, 114
- Mount point added to `/etc/fstab` for persistence

## Nextcloud AIO Reserve (sdb2)
- Reserved for future Nextcloud AIO (nc-02) storage
- Previously labeled `wazuh-data` — repurposed 2026-03-08
- Wazuh indexer data migration was abandoned in favor of Proxmox VM snapshots
- Will be used when nc-02 (Nextcloud AIO with built-in Collabora) is deployed

## General Storage (sdb3)
- Immich photo storage at `/mnt/pve/storage/immich`
- Nextcloud data at `/mnt/pve/storage/nextcloud-data`
- Planned: Jellyfin metadata cache

---

## fstab Entries
All three partitions are configured to mount automatically on boot via `/etc/fstab` on the Proxmox host:

```
/dev/sdb1 /mnt/pve/backup-hdd ext4 defaults 0 0
/dev/sdb2 /mnt/pve/nextcloud-aio ext4 defaults 0 0
/dev/sdb3 /mnt/pve/storage ext4 defaults 0 0
```

---

## Notes
- `backup-usb` mount point also exists at `/mnt/pve/backup-usb` — legacy mount point from before the 4TB drive was configured, no longer active
- The Proxmox boot disk (pve-root) was at ~90% usage before this migration, now sits at ~25%
- sdb2 was previously mounted as `wazuh-data` and cleaned/renamed on 2026-03-08
