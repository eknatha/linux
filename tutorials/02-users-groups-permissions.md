# 02 — Users, Groups & Permissions (RHEL)

> **Author:** Eknatha Reddy P · [eknathalabs.com](https://eknathalabs.com) · [github.com/eknatha](https://github.com/eknatha)  
> **Applies to:** RHEL 8/9, CentOS Stream, Rocky Linux, AlmaLinux  
> **Level:** Beginner → Intermediate

---

## Table of Contents
1. [User Account Management](#1-user-account-management)
2. [Group Management](#2-group-management)
3. [Understanding Permissions](#3-understanding-permissions)
4. [chmod — Changing Permissions](#4-chmod)
5. [chown & chgrp](#5-chown--chgrp)
6. [Special Bits — SUID, SGID, Sticky](#6-special-bits)
7. [umask](#7-umask)
8. [Access Control Lists (ACLs)](#8-access-control-lists)
9. [sudo Configuration](#9-sudo-configuration)
10. [Do's and Don'ts](#10-dos-and-donts)

---

## 1. User Account Management

### Key Files
```
/etc/passwd     — User account info (username, UID, GID, home, shell)
/etc/shadow     — Encrypted passwords + expiry
/etc/group      — Group definitions
/etc/gshadow    — Secure group info
/etc/skel/      — Template files copied to new home directories
```

### useradd — Creating Users
```bash
# Basic user creation
useradd eknatha

# Full options
useradd \
  -m \                          # Create home directory
  -d /home/eknatha \            # Home directory path
  -s /bin/bash \                # Login shell
  -c "Eknatha Reddy, Platform Engineer" \  # GECOS comment
  -u 1500 \                     # Specific UID
  -g engineers \                # Primary group
  -G wheel,docker,sysops \      # Supplementary groups
  eknatha

# Verify
id eknatha                      # uid=1500(eknatha) gid=1001(engineers) groups=...
grep eknatha /etc/passwd
```

### passwd — Setting Passwords
```bash
passwd eknatha                  # Set password for user (as root)
passwd                          # Change your own password
passwd -l eknatha               # Lock account
passwd -u eknatha               # Unlock account
passwd -e eknatha               # Force password change on next login
passwd -n 7 -x 90 -w 14 eknatha # Min 7 days, max 90 days, warn 14 days
```

### usermod — Modifying Users
```bash
usermod -aG wheel eknatha       # Add to group (-a = append, crucial!)
usermod -s /bin/zsh eknatha     # Change shell
usermod -l newname oldname      # Rename user
usermod -d /data/home/eknatha -m eknatha  # Move home directory
usermod -L eknatha              # Lock (prepends ! to password hash)
usermod -U eknatha              # Unlock
usermod -e 2025-12-31 eknatha   # Set account expiry
```

### userdel — Deleting Users
```bash
userdel eknatha                 # Remove user but keep home directory
userdel -r eknatha              # Remove user AND home directory + mail spool

# Safer: lock first, delete after review
usermod -L eknatha
# ... confirm nothing is running as this user ...
userdel -r eknatha
```

### chage — Password Aging
```bash
chage -l eknatha                # List password aging info
chage -M 90 eknatha             # Max 90 days before change required
chage -m 7 eknatha              # Min 7 days between changes
chage -W 14 eknatha             # Warn 14 days before expiry
chage -E 2025-12-31 eknatha     # Account expiry date
chage -d 0 eknatha              # Force change at next login
```

---

## 2. Group Management

```bash
# Create group
groupadd engineers
groupadd -g 2000 devops         # Specific GID

# Modify group
groupmod -n sre-team engineers  # Rename group
groupmod -g 2001 devops         # Change GID

# Delete group (cannot delete primary group of existing user)
groupdel oldgroup

# Add user to group (use usermod -aG, not gpasswd alone)
usermod -aG docker eknatha
gpasswd -a eknatha docker       # Alternative

# Remove user from group
gpasswd -d eknatha oldgroup

# View group membership
id eknatha                      # All groups
groups eknatha                  # Simple list
cat /etc/group | grep eknatha
```

> **⚠️ Warning:** Changes to group membership take effect at **next login**. Use `newgrp groupname` to activate immediately in current session.

---

## 3. Understanding Permissions

### Permission String Breakdown
```
-rwxr-xr--  2  eknatha  engineers  4096  Aug 12  file.sh
│└┬┘└┬┘└┬┘  │  │        │          │               │
│ │  │  │   │  Owner    Group      Size            File
│ │  │  │   └─ Hard link count
│ │  │  └────── Others (r--)
│ │  └────────── Group (r-x)
│ └─────────────── Owner (rwx)
└──────────────────── File type (- = regular, d = dir, l = symlink)
```

### Permission Values
| Symbol | Octal | File | Directory |
|--------|-------|------|-----------|
| `r` | 4 | Read file content | List directory contents |
| `w` | 2 | Modify file content | Create/delete files inside |
| `x` | 1 | Execute file | Enter directory (cd) |
| `-` | 0 | No permission | No permission |

**Example octal calculation:**
```
rwxr-xr-- = 111 101 100 = 7 5 4 = 754
rw-rw-r-- = 110 110 100 = 6 6 4 = 664
rwx------ = 111 000 000 = 7 0 0 = 700
```

---

## 4. chmod

### Symbolic Mode
```bash
chmod u+x script.sh             # Add execute for owner
chmod g-w file.txt              # Remove write from group
chmod o= file.txt               # Remove all permissions from others
chmod a+r file.txt              # Add read for all (a = ugo)
chmod u=rwx,g=rx,o= script.sh  # Set precisely

# Recursive
chmod -R 755 /var/www/html      # Apply to directory and all contents
```

### Octal Mode (Recommended for Scripts)
```bash
chmod 755 script.sh             # rwxr-xr-x (typical for scripts)
chmod 644 config.conf           # rw-r--r-- (typical for config files)
chmod 600 ~/.ssh/id_rsa         # rw------- (private keys)
chmod 700 ~/.ssh/               # rwx------ (SSH directory)
chmod 664 shared.txt            # rw-rw-r-- (group writable)
chmod 750 /opt/app/             # rwxr-x--- (app directory)
```

### Common Permission Patterns
| Octal | Symbolic | Use Case |
|-------|----------|----------|
| 755 | rwxr-xr-x | Executables, web root dirs |
| 644 | rw-r--r-- | Config files, web content |
| 600 | rw------- | Private keys, sensitive config |
| 700 | rwx------ | User home directory scripts |
| 750 | rwxr-x--- | Application directories |
| 664 | rw-rw-r-- | Shared team files |
| 777 | rwxrwxrwx | Avoid! Only /tmp-like dirs |

---

## 5. chown & chgrp

```bash
# Change owner
chown eknatha file.txt
chown eknatha:engineers file.txt    # Change both owner and group
chown :engineers file.txt           # Change group only (same as chgrp)

# Recursive
chown -R eknatha:engineers /opt/app/

# Change only if current owner matches
chown --from=root:root eknatha:engineers sensitive.conf

# chgrp
chgrp engineers shared/
chgrp -R docker /opt/docker-data/
```

---

## 6. Special Bits

### SUID (Set User ID) — Octal 4000
When set on an **executable**, it runs as the **file owner** (not the executing user).

```bash
# Classic SUID example
ls -l /usr/bin/passwd
# -rwsr-xr-x  root root  (s in owner execute position)

# Set SUID
chmod u+s /usr/local/bin/myapp
chmod 4755 /usr/local/bin/myapp

# Find all SUID files (security audit)
find / -perm /4000 -type f 2>/dev/null
```

> ⚠️ **Security Risk:** SUID on scripts is ignored by Linux kernel. SUID on arbitrary binaries can be a privilege escalation vector. Audit regularly.

### SGID (Set Group ID) — Octal 2000
- On **executables**: runs as the **file's group**
- On **directories**: new files inherit the **directory's group** (team sharing)

```bash
# SGID on directory (team share)
mkdir /shared/project
chgrp engineers /shared/project
chmod g+s /shared/project        # or: chmod 2770 /shared/project

# Files created inside inherit group 'engineers'
ls -l /shared/project            # drwxrws---

# Find SGID
find / -perm /2000 -type f 2>/dev/null
```

### Sticky Bit — Octal 1000
On directories: users can only **delete files they own** (even if they have write permission on the dir).

```bash
ls -ld /tmp
# drwxrwxrwt  (t in others execute position)

# Set sticky bit
chmod +t /shared/uploads
chmod 1777 /shared/uploads

# Result: everyone can write, but only file owner (or root) can delete
```

### Combined Special Bits
```bash
chmod 4755 file    # SUID + rwxr-xr-x
chmod 2770 dir     # SGID + rwxrwx---
chmod 1777 dir     # Sticky + rwxrwxrwx
chmod 6755 file    # SUID + SGID + rwxr-xr-x
```

---

## 7. umask

`umask` defines **default permissions subtracted** from maximum (666 for files, 777 for dirs).

```bash
umask                           # View current umask (e.g., 0022)
umask 027                       # Set umask for current session

# Calculation:
# Files:  666 - 022 = 644 (rw-r--r--)
# Dirs:   777 - 022 = 755 (rwxr-xr-x)
# Secure: 777 - 027 = 750 for dirs, 666-027=640 for files

# Persist umask in ~/.bashrc or /etc/profile
echo "umask 027" >> ~/.bashrc
```

### Common umask Values
| umask | Files | Dirs | Use Case |
|-------|-------|------|----------|
| 022 | 644 | 755 | Default (read for all) |
| 027 | 640 | 750 | Secure (no others access) |
| 077 | 600 | 700 | Private (user only) |
| 002 | 664 | 775 | Team collaboration |

---

## 8. Access Control Lists

ACLs provide **fine-grained permissions** beyond owner/group/others.

```bash
# Check if ACL support is enabled
mount | grep acl
# For XFS (default on RHEL): ACLs are enabled by default
# For ext4: may need 'acl' in /etc/fstab mount options

# View ACLs
getfacl /var/www/html/config.php

# Set ACL — give user specific access
setfacl -m u:eknatha:rx /var/www/html/
setfacl -m u:deploy:rwx /opt/app/releases/
setfacl -m g:developers:rx /etc/nginx/sites-available/

# Remove specific ACL entry
setfacl -x u:eknatha /var/www/html/

# Remove all ACLs
setfacl -b /path/to/file

# Default ACLs — inherited by new files in directory
setfacl -d -m g:engineers:rwx /shared/project/

# Recursive
setfacl -R -m u:eknatha:rx /var/log/app/

# Copy ACL from one file to another
getfacl source.file | setfacl --set-file=- target.file
```

### Reading getfacl Output
```bash
getfacl /shared/project/
# file: shared/project/
# owner: root
# group: engineers
# flags: -s-              ← SGID set
user::rwx                 ← Owner permissions
user:eknatha:r-x          ← Specific user ACL
group::rwx                ← Group permissions
group:docker:r-x          ← Specific group ACL
mask::rwx                 ← Maximum effective permissions
other::---                ← Others permissions
default:user::rwx         ← Default for new files
default:group::rwx
default:other::---
```

---

## 9. sudo Configuration

### /etc/sudoers — Edit with visudo ONLY
```bash
sudo visudo                     # Always use visudo — validates syntax before saving
sudo visudo -f /etc/sudoers.d/eknatha  # Edit a drop-in file
```

### sudoers Syntax
```
# Format: who  where = (as-whom) what
root          ALL=(ALL)           ALL
eknatha       ALL=(ALL)           ALL            # Full sudo
eknatha       ALL=(ALL)           NOPASSWD: ALL  # No password (avoid in prod)
%wheel        ALL=(ALL)           ALL            # Group wheel
%docker       ALL=(ALL)           /usr/bin/docker  # Specific command only
eknatha       ALL=(root)          /usr/bin/systemctl restart nginx
```

### Drop-in Files (Preferred over editing sudoers directly)
```bash
# Create /etc/sudoers.d/eknatha
cat > /etc/sudoers.d/eknatha << 'EOF'
# Allow eknatha to manage nginx and check logs
eknatha ALL=(root) NOPASSWD: /usr/bin/systemctl start nginx, \
                              /usr/bin/systemctl stop nginx, \
                              /usr/bin/systemctl restart nginx, \
                              /usr/bin/journalctl -u nginx
EOF

chmod 440 /etc/sudoers.d/eknatha   # Must be 440 or 400
```

### Useful sudo Options
```bash
sudo -l                         # List your sudo permissions
sudo -u eknatha command         # Run as another user
sudo -i                         # Interactive root shell with root's env
sudo -s                         # Shell as root but keep your env
sudo !!                         # Re-run last command with sudo
```

---

## 10. Do's and Don'ts

### ✅ Do's

**Do use `usermod -aG` (not just `-G`) to add to groups:**
```bash
usermod -aG docker eknatha     # -a = append: keeps existing groups
# Without -a, all current supplementary groups are REPLACED
```

**Do use `visudo` for all sudoers edits:**
```bash
sudo visudo                    # Syntax validation before save
```

**Do set restrictive permissions on sensitive files:**
```bash
chmod 600 ~/.ssh/id_rsa
chmod 600 /etc/ssl/private/server.key
chmod 640 /etc/shadow           # Default on RHEL
```

**Do use drop-in files for sudoers:**
```bash
# /etc/sudoers.d/appteam instead of editing /etc/sudoers directly
```

**Do set default ACLs on shared directories:**
```bash
setfacl -d -m g:team:rwX /shared/project/
# New files will inherit group permissions automatically
```

### ❌ Don'ts

**Don't use 777 permissions:**
```bash
chmod 777 /var/www/html    # ❌ World-writable — security nightmare
chmod 755 /var/www/html    # ✅ Correct for web root
```

**Don't run production apps as root:**
```bash
# ❌ Bad
ExecStart=/usr/bin/node /opt/app/server.js   # in a unit running as root

# ✅ Good — use dedicated user
useradd -r -s /sbin/nologin nodeapp
ExecStart=/usr/bin/node /opt/app/server.js
User=nodeapp
Group=nodeapp
```

**Don't delete the primary group of a user:**
```bash
groupdel engineers        # ❌ If eknatha's primary GID is engineers
# Remove user first, then delete group
```

**Don't give NOPASSWD sudo broadly:**
```bash
# ❌ Too broad
eknatha ALL=(ALL) NOPASSWD: ALL

# ✅ Narrow it to specific commands
eknatha ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp
```

**Don't ignore SUID on unexpected binaries:**
```bash
# Audit regularly
find / -perm /4000 -type f 2>/dev/null | sort
# Any binary you didn't put there and is SUID is a security risk
```

---

## Quick Reference

```bash
# User management
useradd -m -s /bin/bash -G wheel username
passwd username
usermod -aG docker username
userdel -r username
chage -l username

# Permissions
chmod 755 script.sh
chmod -R 644 /var/www/html/
chown eknatha:engineers file
chown -R eknatha:engineers /opt/app/

# ACLs
getfacl file
setfacl -m u:eknatha:rx file
setfacl -d -m g:team:rwx /shared/
setfacl -b file                  # Remove all ACLs

# Special bits
chmod u+s binary                 # SUID
chmod g+s directory              # SGID
chmod +t /shared/uploads         # Sticky bit
find / -perm /4000 2>/dev/null   # Find SUID

# sudo
sudo visudo
sudo -l
sudo -u otheruser command
```

---

> **Prev:** [01 — File System & Navigation](./01-file-system-navigation.md)  
> **Next:** [03 — Process Management](./03-process-management.md)  
> **Repository:** [github.com/eknatha/linux](https://github.com/eknatha/linux)
