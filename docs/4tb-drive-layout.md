# 4TB External HDD — Partition Layout

**Device:** `/dev/sdb` (4TB External HDD attached to Proxmox host)  
**Host:** lab-pve (192.168.1.108)  
**Documented:** 2026-02-25

---

## Overview

The 4TB drive is partitioned into three sections with dedicated purposes. It replaced the previous backup configuration which was accidentally writing to the Proxmox main system drive (pve-root). Migrating to this drive freed ~29GB on the boot disk.

---

## Partition Layout

| Partition | Size | Mount Point | Purpose |
|-----------|------|-------------|---------|
| sdb1 | 465GB | `/mnt/pve/backup-hdd` | Proxmox VM/LXC backups |
| sdb2 | 465GB | `/mnt/pve/wazuh-data` | Reserved for Wazuh indexer data migration |
| sdb3 | 2.7TB | `/mnt/pve/storage` | General storage / future use |

---

## Backup Configuration (sdb1)

- Proxmox backup job runs daily
- Retention policy: last 7 backups
- All LXCs and VMs included in backup job
- Mount point added to `/etc/fstab` for persistence

## Wazuh Reserve (sdb2)

- Reserved for future migration of Wazuh indexer data directory
- Wazuh indexer currently stores data on the wazuh-01 VM boot disk (50GB)
- Migration to sdb2 planned as log volume grows
- See `docs/wazuh-setup.md` for Wazuh configuration details

## General Storage (sdb3)

- Currently available for future use
- Planned uses: Jellyfin metadata cache, overflow storage

---

## fstab Entries

All three partitions are configured to mount automatically on boot via `/etc/fstab` on the Proxmox host.

---

## Notes

- `backup-usb` mount point also exists at `/mnt/pve/backup-usb` — this is a legacy mount point from before the 4TB drive was configured and is no longer active
- The Proxmox boot disk (pve-root) was at ~90% usage before this migration, now sits at ~17%
