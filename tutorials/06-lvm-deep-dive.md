# 06 — LVM Deep Dive (RHEL)

> **Author:** Eknatha Reddy P · [eknathalabs.com](https://eknathalabs.com) · [github.com/eknatha](https://github.com/eknatha)  
> **Applies to:** RHEL 8/9, CentOS Stream, Rocky Linux, AlmaLinux  
> **Level:** Intermediate

---

## Table of Contents
1. [LVM Architecture](#1-lvm-architecture)
2. [Physical Volumes (PV)](#2-physical-volumes)
3. [Volume Groups (VG)](#3-volume-groups)
4. [Logical Volumes (LV)](#4-logical-volumes)
5. [Resizing — Grow & Shrink](#5-resizing)
6. [LVM Snapshots](#6-lvm-snapshots)
7. [Thin Provisioning](#7-thin-provisioning)
8. [LVM Migration & pvmove](#8-lvm-migration)
9. [LVM Cache (lvmcache)](#9-lvm-cache)
10. [Troubleshooting](#10-troubleshooting)
11. [Do's and Don'ts](#11-dos-and-donts)

---

## 1. LVM Architecture

```
Physical Storage Layer:
  /dev/sdb  /dev/sdc  /dev/sdd   ← Physical disks / partitions

Physical Volumes (PV):
  [/dev/sdb]  [/dev/sdc]  [/dev/sdd]   ← Initialized with pvcreate

Volume Group (VG):
  [         vg_data          ]          ← Pool of storage from PVs
  PE PE PE PE PE PE PE PE PE PE         ← Physical Extents (default 4MB each)

Logical Volumes (LV):
  [  lv_app  ][  lv_db  ][ lv_logs ]   ← Flexible, resizable volumes

Filesystems:
  [   xfs    ][  ext4   ][   xfs   ]   ← Created on top of LVs
```

**Key concepts:**
- **PE (Physical Extent)**: Minimum allocation unit (default 4MB)
- **LE (Logical Extent)**: Maps to PEs; LV size = number of LEs × PE size
- **Metadata**: LVM stores metadata at start and end of each PV

---

## 2. Physical Volumes

```bash
# Initialize disk/partition as PV
pvcreate /dev/sdb
pvcreate /dev/sdb1 /dev/sdb2        # Multiple partitions
pvcreate --dataalignment 4M /dev/sdb  # Align to 4MB (SSD optimization)

# View PVs
pvs                              # Summary
pvdisplay                        # Detailed
pvdisplay /dev/sdb               # Specific PV
pvscan                           # Scan for all PVs

# Remove PV (must move data off first)
pvremove /dev/sdb                # Remove LVM metadata from disk
```

**pvs output explained:**
```
  PV         VG        Fmt  Attr PSize   PFree
  /dev/sdb   vg_data   lvm2 a--  100.00g  50.00g
  /dev/sdc   vg_data   lvm2 a--   50.00g   0.00g
│             │              │    │        └─ Free space
│             │              │    └────────── PSize: Total size
│             │              └─────────────── Attributes (a=allocatable)
│             └────────────────────────────── Volume Group name
└──────────────────────────────────────────── Device path
```

---

## 3. Volume Groups

```bash
# Create VG from one or more PVs
vgcreate vg_data /dev/sdb
vgcreate vg_data /dev/sdb /dev/sdc    # Multiple PVs
vgcreate -s 8M vg_data /dev/sdb      # Custom PE size (8MB instead of 4MB)

# View VGs
vgs                              # Summary
vgdisplay                        # Detailed
vgdisplay vg_data                # Specific VG

# Extend VG with new disk
vgextend vg_data /dev/sdd

# Reduce VG (remove a PV — must be empty)
pvmove /dev/sdb                  # Move data off /dev/sdb first
vgreduce vg_data /dev/sdb        # Then remove from VG

# Rename VG
vgrename vg_data vg_production

# Remove VG (must remove all LVs first)
vgremove vg_data

# Export/Import VG (for disk migration between systems)
vgexport vg_data                 # Prepare for migration
vgimport vg_data                 # Import on new system
vgchange -ay vg_data             # Activate
```

---

## 4. Logical Volumes

### Creating Logical Volumes
```bash
# By size
lvcreate -L 20G -n lv_app vg_data
lvcreate -L 50G -n lv_db vg_data
lvcreate -L 500M -n lv_swap vg_data

# By percentage
lvcreate -l 50%VG -n lv_app vg_data       # 50% of VG
lvcreate -l 100%FREE -n lv_data vg_data   # All remaining space
lvcreate -l 80%PVS -n lv_data vg_data     # 80% of PV space

# With stripe (performance — similar to RAID 0)
lvcreate -L 100G -n lv_db -i 2 -I 64 vg_data   # 2 stripes, 64KB chunk

# Mirror (similar to RAID 1)
lvcreate -L 50G -n lv_mirror -m 1 vg_data   # 1 mirror copy
```

### Viewing Logical Volumes
```bash
lvs                              # Summary
lvdisplay                        # Detailed
lvdisplay /dev/vg_data/lv_app    # Specific LV
lvscan                           # Scan for all LVs
```

### Full Workflow: New Disk to Mounted Volume
```bash
# 1. Initialize PV
pvcreate /dev/sdb

# 2. Create or extend VG
vgcreate vg_data /dev/sdb
# or extend existing:
# vgextend vg_data /dev/sdb

# 3. Create LV
lvcreate -L 80G -n lv_app vg_data

# 4. Create filesystem
mkfs.xfs /dev/vg_data/lv_app

# 5. Create mount point and mount
mkdir -p /opt/app
mount /dev/vg_data/lv_app /opt/app

# 6. Add to fstab (use device path or UUID)
echo '/dev/vg_data/lv_app /opt/app xfs defaults 0 0' >> /etc/fstab

# 7. Verify
df -hT /opt/app
```

### Activating Deactivated LVs
```bash
lvchange -ay /dev/vg_data/lv_app     # Activate
lvchange -an /dev/vg_data/lv_app     # Deactivate
vgchange -ay vg_data                  # Activate all LVs in VG
```

---

## 5. Resizing

### Growing an LV + Filesystem (Online — No Downtime)

```bash
# XFS — Can only grow, not shrink
# Grow LV by 20G
lvextend -L +20G /dev/vg_data/lv_app

# Grow filesystem to fill LV
xfs_growfs /opt/app                  # Uses mountpoint (not device)
# or
xfs_growfs /dev/vg_data/lv_app

# Do both in one command
lvextend -L +20G -r /dev/vg_data/lv_app   # -r = resize filesystem

# Grow to specific total size
lvextend -L 100G /dev/vg_data/lv_app
xfs_growfs /opt/app

# Use all free space in VG
lvextend -l +100%FREE /dev/vg_data/lv_app
xfs_growfs /opt/app
```

### Growing ext4 Filesystem
```bash
lvextend -L +20G /dev/vg_data/lv_db
resize2fs /dev/vg_data/lv_db         # Online resize supported
# or:
lvextend -L +20G -r /dev/vg_data/lv_db  # -r does both
```

### Shrinking LV — ext4 Only (XFS CANNOT SHRINK)
```bash
# ⚠️ Must UNMOUNT first — cannot shrink online

umount /dev/vg_data/lv_db
e2fsck -f /dev/vg_data/lv_db        # Required before shrink
resize2fs /dev/vg_data/lv_db 30G    # Shrink filesystem to 30G
lvreduce -L 30G /dev/vg_data/lv_db  # Shrink LV to match
mount /dev/vg_data/lv_db /var/lib/mysql
```

---

## 6. LVM Snapshots

Snapshots capture the state of an LV at a point in time using **Copy-on-Write (CoW)**.

```bash
# Create snapshot (requires free space in VG for CoW data)
lvcreate -L 5G -s -n lv_app-snap /dev/vg_data/lv_app
#          │    │   └─ Snapshot name
#          │    └───── -s = snapshot
#          └────────── Size for CoW data (not full copy — just changed blocks)

# View snapshots
lvs -a                               # -a shows hidden volumes
lvdisplay /dev/vg_data/lv_app-snap

# Mount snapshot (read-only by default)
mkdir /mnt/snapshot
mount -o ro /dev/vg_data/lv_app-snap /mnt/snapshot

# Mount writable snapshot
mount -o rw /dev/vg_data/lv_app-snap /mnt/snapshot

# Remove snapshot
umount /mnt/snapshot
lvremove /dev/vg_data/lv_app-snap

# Revert (merge) — DESTROYS current state and restores snapshot
umount /opt/app
lvconvert --merge /dev/vg_data/lv_app-snap
mount /opt/app
```

### Backup Workflow with Snapshots
```bash
# 1. Create snapshot
lvcreate -L 10G -s -n lv_db-backup /dev/vg_data/lv_db

# 2. Mount and backup from snapshot (original DB keeps running)
mount -o ro /dev/vg_data/lv_db-backup /mnt/db-snap
rsync -avz /mnt/db-snap/ /backup/db-$(date +%Y%m%d)/

# 3. Cleanup
umount /mnt/db-snap
lvremove -f /dev/vg_data/lv_db-backup
```

---

## 7. Thin Provisioning

Thin provisioning allows creating LVs larger than available space — allocate on write.

```bash
# 1. Create thin pool
lvcreate -L 100G --thinpool tp_pool vg_data

# 2. Create thin LVs (can be larger than pool!)
lvcreate -V 50G --thin -n lv_app vg_data/tp_pool
lvcreate -V 50G --thin -n lv_db vg_data/tp_pool
lvcreate -V 50G --thin -n lv_logs vg_data/tp_pool
# Total provisioned: 150G on 100G pool — overcommit!

# 3. Monitor thin pool usage
lvs -a vg_data
# Watch Data% column — if it hits 100%, thin pool is full = data corruption!

# 4. Extend thin pool when needed
lvextend -L +50G vg_data/tp_pool

# 5. Thin snapshot (very efficient — shares extents)
lvcreate -s --name lv_app-snap vg_data/lv_app
```

> ⚠️ **Warning:** Monitor thin pool Data% continuously. If pool fills up without warning, filesystems go read-only. Set up alerts at 80%.

---

## 8. LVM Migration — pvmove

Move data between physical volumes without downtime.

```bash
# Move all data off /dev/sdb (e.g., to replace failing disk)
pvmove /dev/sdb

# Move specific LV's data
pvmove -n lv_app /dev/sdb /dev/sdc

# pvmove runs in background; monitor progress
lvs -a                             # Check copy% column
watch -n 5 'pvs && lvs'

# If pvmove is interrupted (reboot), resume it
pvmove               # Will resume from where it stopped

# After pvmove completes
vgreduce vg_data /dev/sdb          # Remove old PV from VG
pvremove /dev/sdb                   # Remove LVM metadata
```

---

## 9. LVM Cache (lvmcache)

Use an SSD to cache frequently accessed data from a slow HDD LV.

```bash
# 1. Initialize SSD as PV
pvcreate /dev/nvme0n1

# 2. Add SSD to existing VG
vgextend vg_data /dev/nvme0n1

# 3. Create cache pool on SSD
lvcreate -L 20G --name cache_pool vg_data /dev/nvme0n1

# 4. Convert cache pool
lvconvert --type cache-pool vg_data/cache_pool

# 5. Attach cache pool to slow LV
lvconvert --type cache --cachepool vg_data/cache_pool vg_data/lv_db

# 6. Monitor cache hit rate
lvdisplay --maps vg_data/lv_db | grep -i cache

# 7. Detach cache (if needed)
lvconvert --uncache vg_data/lv_db
```

---

## 10. Troubleshooting

```bash
# VG not visible after reboot
pvscan --cache                         # Update LVM device cache
vgscan                                 # Scan for VGs
vgchange -ay                           # Activate all VGs

# LV not visible
lvscan
lvchange -ay /dev/vg_data/lv_app

# Check for LVM errors
dmesg | grep -i lvm
journalctl | grep -i lvm
pvck /dev/sdb                          # Check PV metadata
vgck vg_data                           # Check VG metadata

# Recover from duplicate VG/PV after disk clone
vgimportclone -n vg_clone /dev/sdc    # Import as new VG with different name

# LVM metadata backup/restore
ls /etc/lvm/backup/                    # Automatic backups of VG metadata
vgcfgrestore -f /etc/lvm/backup/vg_data vg_data  # Restore metadata

# Thin pool data recovery
lvconvert --repair vg_data/tp_pool     # Attempt repair
```

---

## 11. Do's and Don'ts

### ✅ Do's

**Do monitor thin pool usage with alerting:**
```bash
# In a monitoring script:
DATA_PCT=$(lvs --noheadings -o data_percent vg_data/tp_pool | tr -d ' ')
if (( $(echo "$DATA_PCT > 80" | bc -l) )); then
    echo "ALERT: Thin pool at ${DATA_PCT}%" | mail -s "LVM Alert" admin@company.com
fi
```

**Do use `-r` flag with lvextend for atomic resize:**
```bash
lvextend -L +20G -r /dev/vg_data/lv_app   # Resize LV + filesystem together
```

**Do keep 10-20% VG free for snapshots:**
```bash
vgs vg_data    # Check VFree column
```

**Do backup LVM metadata:**
```bash
vgcfgbackup -f /backup/lvm-$(date +%Y%m%d).backup    # Manual backup
ls /etc/lvm/backup/        # Auto backups are here
```

### ❌ Don'ts

**Don't try to shrink XFS:**
```bash
# XFS does not support shrinking — plan storage allocation upfront
# ❌ lvreduce without resize2fs equivalent for XFS
```

**Don't let thin pool hit 100%:**
```bash
# 100% thin pool = all filesystems on that pool go read-only instantly
# Monitor Data% in lvs output continuously
```

**Don't remove a PV without pvmove:**
```bash
# ❌ Data corruption if you vgreduce without moving data off first
vgreduce vg_data /dev/sdb    # Only safe if PV shows 0 used extents (pvs)

# ✅ Safe path:
pvmove /dev/sdb              # Move data off first
vgreduce vg_data /dev/sdb   # Then remove
```

---

## Quick Reference

```bash
# Full workflow
pvcreate /dev/sdb
vgcreate vg_data /dev/sdb
lvcreate -L 50G -n lv_app vg_data
mkfs.xfs /dev/vg_data/lv_app
mount /dev/vg_data/lv_app /opt/app

# Display
pvs && vgs && lvs
pvdisplay /dev/sdb
vgdisplay vg_data
lvdisplay /dev/vg_data/lv_app

# Extend (online)
vgextend vg_data /dev/sdc
lvextend -L +20G -r /dev/vg_data/lv_app   # XFS
lvextend -L +20G -r /dev/vg_data/lv_db    # ext4

# Snapshots
lvcreate -L 5G -s -n lv_app-snap /dev/vg_data/lv_app
lvremove -f /dev/vg_data/lv_app-snap

# Migrate
pvmove /dev/sdb
vgreduce vg_data /dev/sdb
```

---

> **Prev:** [05 — Storage & Partitioning](./05-storage-partitioning.md)  
> **Next:** [07 — Networking on RHEL](./07-networking-rhel.md)  
> **Repository:** [github.com/eknatha/linux](https://github.com/eknatha/linux)
