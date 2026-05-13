# 05 — Storage & Partitioning (RHEL)

> **Author:** Eknatha Reddy P · [eknathalabs.com](https://eknathalabs.com) · [github.com/eknatha](https://github.com/eknatha)  
> **Applies to:** RHEL 8/9, CentOS Stream, Rocky Linux, AlmaLinux  
> **Level:** Intermediate

---

## Table of Contents
1. [Block Device Fundamentals](#1-block-device-fundamentals)
2. [Partition Tables — MBR vs GPT](#2-partition-tables)
3. [fdisk — MBR Partitioning](#3-fdisk)
4. [gdisk — GPT Partitioning](#4-gdisk)
5. [parted — Universal Tool](#5-parted)
6. [Filesystems — mkfs](#6-filesystems)
7. [Mounting Filesystems](#7-mounting)
8. [/etc/fstab — Persistent Mounts](#8-fstab)
9. [Swap Space](#9-swap-space)
10. [RAID Basics with mdadm](#10-raid-basics)
11. [Disk Monitoring](#11-disk-monitoring)
12. [Do's and Don'ts](#12-dos-and-donts)

---

## 1. Block Device Fundamentals

```bash
# List all block devices
lsblk                          # Tree view
lsblk -f                       # Include filesystem, UUID, label
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,UUID

# Detailed device info
fdisk -l                       # All disk partition tables (root required)
fdisk -l /dev/sda              # Specific disk
blkid                          # UUIDs and filesystem types
blkid /dev/sda1

# Disk I/O stats
iostat -xz 1 5                 # Extended stats, 5 iterations, 1s interval
hdparm -I /dev/sda             # Drive info (SATA)
nvme list                      # NVMe drives
```

### Device Naming Conventions
```
/dev/sda          — SCSI/SATA/USB disk (a = first, b = second)
/dev/sda1         — First partition on sda
/dev/nvme0n1      — NVMe disk (n1 = namespace 1)
/dev/nvme0n1p1    — First partition on NVMe
/dev/vda          — Virtual disk (KVM/cloud VMs)
/dev/xvda         — Xen virtual disk (AWS older)
/dev/md0          — Software RAID device
/dev/mapper/vg-lv — LVM logical volume
```

---

## 2. Partition Tables

### MBR (Master Boot Record)
- Max disk size: **2 TB**
- Max primary partitions: **4** (or 3 primary + 1 extended with logical partitions)
- Boot sector: first 512 bytes
- Legacy BIOS boot

### GPT (GUID Partition Table)
- Max disk size: **9.4 ZB** (no practical limit)
- Max partitions: **128** (typically)
- Required for disks > 2TB
- Required for UEFI boot
- Stores backup at end of disk
- RHEL 8+ default for new installations

```bash
# Check partition table type
parted /dev/sda print | grep "Partition Table"
fdisk -l /dev/sda | grep "Disklabel type"
```

---

## 3. fdisk — MBR Partitioning

```bash
sudo fdisk /dev/sdb

# fdisk interactive commands:
# p — print partition table
# n — new partition
# d — delete partition
# t — change partition type
# l — list partition types
# w — write and exit
# q — quit without saving
# m — help menu

# Common workflow: create new partition
Command (m for help): n
Partition type: p (primary)
Partition number: 1
First sector: [Enter for default]
Last sector: +20G          # or +20480M or specific sector number

# Change type to LVM (8e) or Linux RAID (fd)
Command: t
Partition number: 1
Hex code: 8e               # Linux LVM

# Write changes
Command: w
```

```bash
# Verify
lsblk /dev/sdb
partprobe /dev/sdb          # Inform kernel of partition table change
```

---

## 4. gdisk — GPT Partitioning

```bash
sudo gdisk /dev/sdb

# Commands (similar to fdisk):
# p — print
# n — new partition
# d — delete
# t — change type
# w — write
# q — quit
# ? — help

# Workflow: new GPT partition
Command: n
Partition number: 1
First sector: [Enter]
Last sector: +50G

# Type codes for GPT
# 8300 = Linux filesystem
# 8e00 = Linux LVM
# fd00 = Linux RAID
# ef00 = EFI System
# 0700 = Microsoft basic data
Command: t
Partition GUID code: 8e00   # Linux LVM

Command: w
```

---

## 5. parted — Universal Tool

`parted` works with both MBR and GPT and is scriptable (non-interactive).

```bash
# Interactive mode
parted /dev/sdb

(parted) print                         # Show partition table
(parted) mklabel gpt                   # Create GPT partition table
(parted) mkpart primary ext4 0% 50%    # Partition using percentages
(parted) mkpart primary xfs 50% 100%
(parted) set 1 lvm on                  # Set LVM flag
(parted) align-check optimal 1         # Check partition alignment
(parted) quit

# Non-interactive (scriptable)
parted -s /dev/sdb mklabel gpt
parted -s /dev/sdb mkpart primary xfs 0% 100%
parted -s /dev/sdb set 1 lvm on
partprobe /dev/sdb
```

---

## 6. Filesystems

### Creating Filesystems (mkfs)

**XFS — Default on RHEL** (recommended for most use cases)
```bash
mkfs.xfs /dev/sdb1
mkfs.xfs -L "data" /dev/sdb1           # With label
mkfs.xfs -f /dev/sdb1                  # Force (overwrite existing)
mkfs.xfs -m reflink=1 /dev/sdb1        # Enable reflinks (RHEL 8+)
```

**ext4 — Widely compatible**
```bash
mkfs.ext4 /dev/sdb2
mkfs.ext4 -L "backup" /dev/sdb2
mkfs.ext4 -m 1 /dev/sdb2              # 1% reserved blocks (default 5%)
tune2fs -l /dev/sdb2                   # View ext4 metadata
tune2fs -m 1 /dev/sdb2                 # Adjust reserved blocks on existing
```

**btrfs — Copy-on-Write, snapshots**
```bash
sudo dnf install btrfs-progs -y
mkfs.btrfs /dev/sdb3
mkfs.btrfs -L "btrfs-data" /dev/sdb3
```

### Filesystem Checks
```bash
# Check filesystem (must be UNMOUNTED)
fsck /dev/sdb1                          # Auto-detect type
fsck.xfs -n /dev/sdb1                  # XFS read-only check
e2fsck -f /dev/sdb2                    # Force ext4 check
xfs_repair /dev/sdb1                   # Repair XFS (unmounted)
```

---

## 7. Mounting Filesystems

```bash
# Basic mount
mount /dev/sdb1 /mnt/data
mount -t xfs /dev/sdb1 /mnt/data       # Specify filesystem type

# Mount options
mount -o noexec,nosuid,nodev /dev/sdb1 /mnt/data    # Security options
mount -o ro /dev/sdb1 /mnt/data                     # Read-only
mount -o remount,rw /mnt/data                       # Remount with new options

# Mount by UUID (preferred in scripts/fstab)
blkid /dev/sdb1                                     # Get UUID
mount UUID="xxxxxxxx-xxxx-xxxx" /mnt/data

# Bind mount (expose directory at second location)
mount --bind /var/www /mnt/webroot
mount --rbind /proc /chroot/proc                    # Recursive bind

# Unmount
umount /mnt/data
umount -l /mnt/data                                 # Lazy unmount (detach when not busy)
umount -f /mnt/nfs-share                            # Force (NFS stale mounts)

# View mounts
mount | grep sdb
findmnt                                             # Tree view of mounts
findmnt /mnt/data                                   # Specific mount
cat /proc/mounts                                    # All current mounts
```

---

## 8. /etc/fstab — Persistent Mounts

`/etc/fstab` defines filesystems to mount at boot.

### Format
```
<device>  <mountpoint>  <fstype>  <options>  <dump>  <pass>
```

- **dump**: 0 = no backup (use 0 always now — dump is obsolete)
- **pass**: 0 = no fsck, 1 = root filesystem (check first), 2 = check after root

### Example Entries
```
# /etc/fstab

# Root filesystem
UUID=abc123  /                xfs     defaults        0 0

# Data disk by UUID (recommended — stable across device renames)
UUID=def456  /data            xfs     defaults        0 0

# Read-only data
UUID=ghi789  /opt/readonly    xfs     ro,noexec       0 0

# NFS mount
nas.local:/exports/data  /mnt/nfs  nfs  rw,hard,timeo=600,retrans=2,_netdev  0 0

# tmpfs (RAM disk)
tmpfs  /tmp  tmpfs  defaults,noexec,nosuid,nodev,size=2G  0 0

# Swap
UUID=swap123  none  swap  sw  0 0

# Bind mount
/var/www  /mnt/webroot  none  bind  0 0
```

### Common Mount Options
| Option | Meaning |
|--------|---------|
| `defaults` | rw, suid, exec, auto, nouser, async |
| `noexec` | Cannot run executables from this filesystem |
| `nosuid` | SUID/SGID bits ignored |
| `nodev` | Device files ignored |
| `ro` | Read-only |
| `rw` | Read-write |
| `_netdev` | Network-dependent (wait for network) |
| `nofail` | Don't fail boot if this mount fails |
| `x-systemd.automount` | Automount on access |
| `discard` | Enable TRIM for SSD |

```bash
# Validate fstab without rebooting
mount -a                        # Mount all non-mounted entries in fstab
findmnt --verify                # Verify fstab consistency (RHEL 8+)
```

---

## 9. Swap Space

```bash
# View current swap
swapon --show
free -h

# Create swap partition
mkswap /dev/sdb3
swapon /dev/sdb3               # Enable immediately
swapoff /dev/sdb3              # Disable

# Create swap FILE (cloud-friendly — no partition needed)
fallocate -l 4G /swapfile      # Fast allocation
# or: dd if=/dev/zero of=/swapfile bs=1M count=4096
chmod 600 /swapfile            # Must be root-only
mkswap /swapfile
swapon /swapfile

# Add to fstab for persistence
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# Verify
swapon --show
free -h

# Swappiness tuning (0=use RAM, 100=swap aggressively)
sysctl vm.swappiness           # View (default 60 on RHEL)
sysctl -w vm.swappiness=10     # Temporary (good for databases)
echo 'vm.swappiness=10' > /etc/sysctl.d/99-swappiness.conf
```

---

## 10. RAID Basics with mdadm

```bash
sudo dnf install mdadm -y

# Create RAID 1 (mirror) from 2 disks
mdadm --create /dev/md0 \
  --level=1 \
  --raid-devices=2 \
  /dev/sdb /dev/sdc

# Create RAID 5 (striping + parity) from 3 disks
mdadm --create /dev/md1 \
  --level=5 \
  --raid-devices=3 \
  /dev/sdb /dev/sdc /dev/sdd

# View RAID status
cat /proc/mdstat
mdadm --detail /dev/md0

# Save RAID configuration
mdadm --detail --scan >> /etc/mdadm.conf

# Format and mount
mkfs.xfs /dev/md0
mount /dev/md0 /mnt/raid

# Add a spare
mdadm --add /dev/md0 /dev/sde

# Simulate disk failure (testing)
mdadm --fail /dev/md0 /dev/sdb
mdadm --remove /dev/md0 /dev/sdb
mdadm --add /dev/md0 /dev/sdf    # Add replacement
```

### RAID Levels
| Level | Min Disks | Fault Tolerance | Use Case |
|-------|-----------|-----------------|----------|
| RAID 0 | 2 | None | Speed, no redundancy |
| RAID 1 | 2 | 1 disk | OS drives, critical data |
| RAID 5 | 3 | 1 disk | General storage |
| RAID 6 | 4 | 2 disks | Larger arrays |
| RAID 10 | 4 | 1 per mirror pair | Performance + redundancy |

---

## 11. Disk Monitoring

```bash
# Disk usage
df -hT                         # Human-readable with filesystem type
df -i                          # Inode usage
du -sh /var/log/*              # Size of each log file
du -sh /* 2>/dev/null | sort -rh | head -10   # Find disk hogs

# I/O monitoring
iostat -x 1                    # Extended I/O stats every 1s
iotop                          # Real-time I/O per process
dstat --disk --cpu             # Combined view

# SMART disk health
sudo dnf install smartmontools -y
smartctl -a /dev/sda           # Full SMART report
smartctl -H /dev/sda           # Health check only
# Schedule SMART test
smartctl -t short /dev/sda     # Short test
smartctl -t long /dev/sda      # Long test (~2 hours)
```

---

## 12. Do's and Don'ts

### ✅ Do's

**Do use UUIDs in /etc/fstab:**
```bash
# Device names (sda1) can change after reboot or hardware change
# UUIDs are stable
blkid /dev/sdb1                # Get UUID
# Use UUID=... in fstab
```

**Do verify fstab before rebooting:**
```bash
mount -a                       # Test all fstab entries
findmnt --verify               # Check for errors
```

**Do add `nofail` for non-critical mounts:**
```bash
# Prevents boot failure if a secondary disk is missing
UUID=xxx /data xfs defaults,nofail 0 0
```

**Do use XFS for RHEL (it's the default and best supported):**
```bash
mkfs.xfs /dev/sdb1             # Default choice for RHEL
# XFS: online resize, efficient large files, RHEL support
```

**Do check alignment:**
```bash
parted /dev/sdb align-check optimal 1    # Should say "1 aligned"
# Misaligned partitions cause performance issues on SSDs
```

### ❌ Don'ts

**Don't run fsck on mounted filesystems:**
```bash
# DESTROYS filesystem
# ❌ Wrong
fsck /dev/sda1    # while / is mounted on it

# Boot from rescue media, then fsck unmounted partition
```

**Don't use device names in fstab (use UUID):**
```bash
# ❌ /dev/sdb1 can become /dev/sdc1 after hardware change
/dev/sdb1  /data  xfs  defaults  0 0

# ✅ UUID is stable
UUID=abc123  /data  xfs  defaults  0 0
```

**Don't skip `partprobe` after partition changes:**
```bash
fdisk /dev/sdb    # Create partition
partprobe /dev/sdb  # Tell kernel about it (or reboot)
# Without this, /dev/sdb1 won't appear
```

**Don't `dd` to the wrong disk:**
```bash
# dd is destructive and has no undo
dd if=/dev/zero of=/dev/sdb    # Wipes /dev/sdb
# Triple-check your target device with: lsblk, fdisk -l
```

---

## Quick Reference

```bash
# Discover disks
lsblk -f
fdisk -l
blkid

# Partition
fdisk /dev/sdb                 # MBR
gdisk /dev/sdb                 # GPT
parted -s /dev/sdb mklabel gpt

# Filesystems
mkfs.xfs /dev/sdb1
mkfs.ext4 /dev/sdb2
mkswap /dev/sdb3

# Mount
mount /dev/sdb1 /mnt/data
mount -a                       # Apply fstab
findmnt --verify               # Verify fstab
umount /mnt/data

# Monitoring
df -hT
du -sh /var/* | sort -rh
iostat -x 1
smartctl -H /dev/sda
```

---

> **Prev:** [04 — systemd & Service Management](./04-systemd-services.md)  
> **Next:** [06 — LVM Deep Dive](./06-lvm-deep-dive.md)  
> **Repository:** [github.com/eknatha/linux](https://github.com/eknatha/linux)
