# 01 — File System & Navigation (RHEL)

> **Author:** Eknatha Reddy P · [eknathalabs.com](https://eknathalabs.com) · [github.com/eknatha](https://github.com/eknatha)  
> **Applies to:** RHEL 8/9, CentOS Stream, Rocky Linux, AlmaLinux  
> **Level:** Beginner → Intermediate

---

## Table of Contents
1. [Linux Filesystem Hierarchy (FHS)](#1-linux-filesystem-hierarchy)
2. [Essential Navigation Commands](#2-essential-navigation-commands)
3. [Listing Files — ls in Depth](#3-listing-files)
4. [Finding Files — find & locate](#4-finding-files)
5. [Inodes, Hard Links & Soft Links](#5-inodes-hard-links--soft-links)
6. [File Types & Metadata](#6-file-types--metadata)
7. [Globbing & Wildcards](#7-globbing--wildcards)
8. [Do's and Don'ts](#8-dos-and-donts)
9. [Quick Reference Cheatsheet](#9-quick-reference-cheatsheet)

---

## 1. Linux Filesystem Hierarchy

```
/                   Root of the entire filesystem
├── /bin            Essential user binaries (ls, cp, mv) — symlink to /usr/bin on RHEL 8+
├── /sbin           System admin binaries — symlink to /usr/sbin on RHEL 8+
├── /etc            System-wide configuration files
├── /home           User home directories
├── /root           Root user's home
├── /var            Variable data (logs, spool, databases)
│   ├── /var/log    System and application logs
│   └── /var/lib    Application state data
├── /tmp            Temporary files (cleared on reboot or by systemd-tmpfiles)
├── /usr            User programs and data
│   ├── /usr/bin    Non-essential user binaries
│   ├── /usr/lib    Libraries
│   └── /usr/share  Architecture-independent data
├── /proc           Virtual filesystem — kernel + process info
├── /sys            Virtual filesystem — hardware and kernel parameters
├── /dev            Device files (disks, terminals, random)
├── /boot           Bootloader, kernel images, initramfs
├── /opt            Optional third-party software
├── /run            Runtime data (PIDs, sockets) — tmpfs, cleared on reboot
└── /mnt, /media    Mount points for temporary filesystems
```

### Key RHEL 8+ Change: UsrMerge
On RHEL 8 and later, `/bin`, `/sbin`, `/lib`, `/lib64` are **symlinks** to their `/usr` counterparts. This prevents split-package issues and is called "UsrMerge."

```bash
ls -la /bin       # → /bin -> usr/bin
ls -la /lib       # → /lib -> usr/lib
```

---

## 2. Essential Navigation Commands

### pwd — Print Working Directory
```bash
pwd                     # /home/eknatha
pwd -P                  # Resolve symlinks to real path
```

### cd — Change Directory
```bash
cd /var/log             # Absolute path
cd ..                   # One level up
cd -                    # Toggle between last two directories
cd ~                    # Go to your home directory
cd ~username            # Go to another user's home
```

### mkdir — Make Directories
```bash
mkdir mydir
mkdir -p /opt/app/config/v2    # Create all parents at once (-p is idempotent)
mkdir -m 750 /secure/dir       # Create with specific permissions
```

### rmdir vs rm
```bash
rmdir emptydir             # Only works on empty directories
rm -rf /path/to/dir        # Recursive + force — use carefully!
```

---

## 3. Listing Files

### Basic ls Options
```bash
ls -l             # Long format: permissions, links, owner, group, size, date
ls -la            # Include hidden files (starting with .)
ls -lh            # Human-readable sizes (KB, MB, GB)
ls -lt            # Sort by modification time, newest first
ls -ltr           # Sort by time, oldest first (great for log dirs)
ls -lS            # Sort by size, largest first
ls -ld /etc       # Show directory itself, not its contents
ls --color=auto   # Colorize output
```

### Long Format Decoded
```
-rw-r--r-- 1 root root 4096 Aug 12 10:30 /etc/hosts
│           │ │    │    │    └── Last modification timestamp
│           │ │    │    └─────── Size in bytes
│           │ │    └──────────── Group owner
│           │ └───────────────── User owner
│           └───────────────── Hard link count
└─────────────────────────────── Permissions (type + rwx*3)
```

### File Type Indicators in ls -l
| First char | Type |
|------------|------|
| `-` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `c` | Character device |
| `b` | Block device |
| `p` | Named pipe (FIFO) |
| `s` | Socket |

---

## 4. Finding Files

### find — The Workhorse
```bash
# By name
find /etc -name "*.conf"
find /var -name "*.log" -type f

# By type
find /dev -type b               # Block devices
find /tmp -type l               # Symlinks

# By size
find /var -size +100M           # Larger than 100 MB
find /home -size -1k            # Smaller than 1 KB
find /var/log -size +50M -size -1G

# By time (in days)
find /var/log -mtime -1         # Modified in last 24 hours
find /tmp -atime +7             # Accessed more than 7 days ago
find /etc -newer /etc/passwd    # Modified after /etc/passwd

# By permissions
find / -perm /4000              # Any SUID file
find / -perm /2000              # Any SGID file
find / -perm -o+w               # World-writable files (security risk)

# By owner
find /home -user eknatha
find /var -group apache

# Combining conditions
find /var/log -name "*.log" -mtime +30 -size +10M

# Execute action on results
find /tmp -name "*.tmp" -delete
find /etc -name "*.bak" -exec ls -lh {} \;
find /var/log -name "*.log" -exec gzip {} \;

# Find and process safely with xargs
find /home -name "*.py" | xargs grep -l "import os"
find /tmp -name "*.tmp" -print0 | xargs -0 rm -f
```

### locate — Fast Indexed Search
```bash
# Install on RHEL
sudo dnf install mlocate -y

# Update the database
sudo updatedb

# Search
locate nginx.conf
locate -i nginx.conf            # Case-insensitive
locate --limit 10 "*.conf"
locate -r '\.conf$'             # Regex pattern

# Check when database was last updated
stat /var/lib/mlocate/mlocate.db
```

> **Note:** `locate` uses a database updated daily via cron. Files created since last `updatedb` won't appear. Use `find` for real-time results.

### which, whereis, type
```bash
which python3                   # /usr/bin/python3
whereis nginx                   # nginx: /usr/sbin/nginx /etc/nginx /usr/share/man/man8/nginx.8.gz
type ls                         # ls is aliased to 'ls --color=auto'
type -a python                  # Show all locations
```

---

## 5. Inodes, Hard Links & Soft Links

### What is an Inode?
Every file is represented by an **inode** — a data structure storing file metadata (permissions, owner, timestamps, block pointers). The filename is stored in the directory entry and points to an inode number.

```bash
ls -li /etc/hosts           # Shows inode number in first column
stat /etc/hosts             # Full inode metadata
```

**stat output explained:**
```
  File: /etc/hosts
  Size: 174             Blocks: 8          IO Block: 4096
Device: fd00h/64768d    Inode: 8391907     Links: 1
Access: (0644/-rw-r--r--)  Uid: (0/root)  Gid: (0/root)
Modify: 2024-08-01 10:30:00
Change: 2024-08-01 10:30:00
```

### Hard Links
A hard link is a **directory entry** pointing to the same inode.

```bash
# Create a hard link
ln /etc/hosts /tmp/hosts-copy

# Both files share the same inode number
ls -li /etc/hosts /tmp/hosts-copy   # Same inode!

# Deleting one doesn't delete the data — only when link count hits 0
rm /tmp/hosts-copy                  # /etc/hosts still works fine
```

**Hard link rules:**
- Cannot cross filesystem boundaries
- Cannot link to directories (except root with bind mounts)
- Link count in `ls -l` shows how many directory entries point to that inode

### Soft (Symbolic) Links
A symlink is a special file containing a **path string** to the target.

```bash
# Create a symlink
ln -s /etc/nginx/nginx.conf /root/nginx.conf
ln -s /usr/bin/python3 /usr/local/bin/python

# View symlinks
ls -la /root/nginx.conf             # Shows → target
readlink /root/nginx.conf           # /etc/nginx/nginx.conf
readlink -f /root/nginx.conf        # Resolve all symlinks to absolute path

# Remove a symlink (never add trailing slash!)
rm /root/nginx.conf                 # Correct
rm /root/nginx.conf/                # WRONG — removes target contents!
```

**Symlink vs Hard Link:**
| Feature | Hard Link | Soft Link |
|---------|-----------|-----------|
| Cross filesystem | ❌ | ✅ |
| Link directories | ❌ (usually) | ✅ |
| Survives target deletion | ✅ | ❌ (dangling) |
| Different inode | ❌ (same) | ✅ |
| `ls -l` indicator | Count increases | `l` prefix + `→` |

---

## 6. File Types & Metadata

### file — Determine File Type
```bash
file /etc/hosts             # /etc/hosts: ASCII text
file /bin/ls                # /bin/ls: ELF 64-bit LSB pie executable
file /dev/sda               # /dev/sda: block special
file /tmp/script.sh         # /tmp/script.sh: Bourne-Again shell script
```

### Timestamps — 3 Types
```bash
stat /etc/passwd
# Access time (atime): last time file was read
# Modify time (mtime): last time file CONTENT was changed
# Change time (ctime): last time file METADATA changed (permissions, owner)
```

```bash
# Update timestamps
touch file.txt              # Create or update atime+mtime to now
touch -m file.txt           # Only update mtime
touch -t 202408121030 file  # Set specific timestamp
```

---

## 7. Globbing & Wildcards

```bash
*           # Any string (including empty)
?           # Any single character
[abc]       # Any character in set
[a-z]       # Any character in range
[!abc]      # Any character NOT in set

# Examples
ls /etc/*.conf
ls /var/log/nginx/access.log.*
ls /home/eknatha/project-?/
ls /etc/[a-m]*.conf
find /tmp -name "[Tt]emp*"
```

### Brace Expansion
```bash
mkdir -p /opt/app/{config,logs,data,tmp}
cp /etc/nginx/nginx.conf{,.bak}         # Backup: nginx.conf.bak
echo {1..5}                              # 1 2 3 4 5
touch file{01..10}.log                  # file01.log ... file10.log
```

---

## 8. Do's and Don'ts

### ✅ Do's

**Do verify before bulk deletes:**
```bash
# First preview what find would delete
find /tmp -name "*.tmp" -mtime +7
# Then add -delete only after confirming output
find /tmp -name "*.tmp" -mtime +7 -delete
```

**Do use absolute paths in scripts:**
```bash
# In scripts, always use absolute paths
/usr/bin/find /var/log -name "*.log"   # Not just: find /var/log
```

**Do use `ls -ltr` for logs:**
```bash
ls -ltr /var/log/nginx/         # Most recent log last — easy to tail
```

**Do use `readlink -f` for symlink resolution:**
```bash
readlink -f /etc/alternatives/python3   # Get final resolved path
```

**Do quote variables with spaces:**
```bash
find /home -name "my file.txt"
FILE="my file.txt"
ls -l "$FILE"           # Always quote
```

### ❌ Don'ts

**Don't run rm -rf without extreme caution:**
```bash
# NEVER run this without triple-checking the variable
rm -rf $DIRECTORY/      # If $DIRECTORY is empty → rm -rf /

# Safer: Use -i for interactive, or move to /tmp first
mv /path/to/delete /tmp/pending-delete-$(date +%s)
```

**Don't use `ls` output in scripts:**
```bash
# WRONG — breaks on filenames with spaces
for f in $(ls /tmp/*.log); do ...

# RIGHT — use glob or find with -print0
for f in /tmp/*.log; do echo "$f"; done
find /tmp -name "*.log" -print0 | xargs -0 process
```

**Don't ignore inode exhaustion:**
```bash
df -i                   # Check inode usage — even with free disk space,
                        # you can't create files if inodes are full
```

**Don't confuse `rm symlink` with `rm symlink/`:**
```bash
ln -s /data/important /tmp/link
rm /tmp/link            # Removes the symlink — target safe ✅
rm /tmp/link/           # Deletes CONTENTS of /data/important ⚠️
```

**Don't modify /proc or /sys directly without understanding the effect:**
```bash
# /proc and /sys are kernel interfaces — changes are immediate and can crash the system
echo 1 > /proc/sys/net/ipv4/ip_forward   # OK — enables IP forwarding
# But random writes to unknown /proc paths can panic the kernel
```

---

## 9. Quick Reference Cheatsheet

```bash
# Navigation
pwd                          # Current directory
cd /var/log                  # Go to absolute path
cd -                         # Toggle last two dirs
ls -lhtr                     # Sorted by time, human-readable

# Finding files
find /etc -name "*.conf"     # By name
find / -perm /4000           # SUID files
find /var -size +100M        # Large files
locate nginx.conf            # Fast indexed search
which python3                # Binary location

# Links
ln file1 file2               # Hard link
ln -s /target /link          # Soft link
readlink -f /link            # Resolve symlink
ls -li file                  # Show inode number

# Metadata
stat file                    # Full metadata
file /bin/ls                 # Determine file type
touch -m file                # Update mtime only

# Globbing
ls /etc/*.conf               # Wildcard
mkdir -p /opt/{a,b,c}        # Brace expansion
cp file.conf{,.bak}          # Quick backup
```

---

> **Next:** [02 — Users, Groups & Permissions](./02-users-groups-permissions.md)  
> **Repository:** [github.com/eknatha/linux](https://github.com/eknatha/linux)
