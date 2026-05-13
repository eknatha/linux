# 10 — Package Management on RHEL

> **Author:** Eknatha Reddy P · [eknathalabs.com](https://eknathalabs.com) · [github.com/eknatha](https://github.com/eknatha)  
> **Applies to:** RHEL 8/9, CentOS Stream, Rocky Linux, AlmaLinux  
> **Level:** Beginner → Intermediate

---

## Table of Contents
1. [dnf — Modern Package Manager](#1-dnf)
2. [rpm — Low-Level Package Tool](#2-rpm)
3. [Repository Management](#3-repository-management)
4. [Module Streams (RHEL 8+)](#4-module-streams)
5. [GPG Keys & Security](#5-gpg-keys)
6. [Local Repositories](#6-local-repositories)
7. [dnf History & Rollback](#7-dnf-history)
8. [Do's and Don'ts](#8-dos-and-donts)

---

## 1. dnf

`dnf` (Dandified YUM) is the default package manager for RHEL 8/9. It replaced `yum`.

### Install, Update, Remove
```bash
# Install
dnf install nginx
dnf install nginx httpd postgresql-server     # Multiple packages
dnf install /path/to/package.rpm              # Local RPM file
dnf install https://example.com/pkg.rpm       # Remote URL

# Update
dnf update                                    # Update all packages
dnf update nginx                              # Update specific package
dnf upgrade                                   # Like update, also handles obsoletes
dnf check-update                              # List available updates (no install)
dnf update --security                         # Security updates only
dnf update --bugfix                           # Bugfix updates only
dnf update --cve CVE-2024-1234               # Update fixing specific CVE

# Remove
dnf remove nginx                              # Remove package (keeps dependencies)
dnf autoremove                                # Remove unused/orphan dependencies
dnf remove nginx --noautoremove               # Remove without touching deps

# Reinstall
dnf reinstall nginx                           # Reinstall (useful if files corrupted)
```

### Search & Info
```bash
# Search
dnf search nginx                              # Search in name + summary
dnf search all "web server"                  # Search all fields

# Info
dnf info nginx                                # Package info
dnf info --installed                          # All installed packages
dnf info --available                          # All available packages
dnf info --updates                            # Packages with updates

# List
dnf list installed                            # All installed
dnf list installed | grep nginx               # Filter
dnf list available                            # Available packages
dnf list updates                              # Packages with updates

# What provides a file?
dnf provides /usr/sbin/nginx
dnf provides "*/libssl.so.1.1"
dnf whatprovides /etc/nginx/nginx.conf
```

### Groups
```bash
# Package groups (install related packages together)
dnf group list                                # List all groups
dnf group list --installed
dnf group info "Development Tools"
dnf group install "Development Tools"
dnf group remove "Development Tools"
dnf group upgrade "Development Tools"
```

### Clean Cache
```bash
dnf clean all                                 # Remove all cached data
dnf clean packages                            # Remove cached packages only
dnf clean metadata                            # Remove repo metadata
dnf makecache                                 # Download fresh metadata
```

---

## 2. rpm

`rpm` is the low-level package tool — operates directly on RPM packages without dependency resolution.

### Query Installed Packages
```bash
rpm -qa                                       # Query all installed packages
rpm -qa | sort | grep nginx
rpm -q nginx                                  # Is this package installed?
rpm -qi nginx                                 # Package info
rpm -ql nginx                                 # List all files in package
rpm -qd nginx                                 # List documentation files
rpm -qc nginx                                 # List config files
rpm -qR nginx                                 # List dependencies (Requires)
rpm -qf /usr/sbin/nginx                      # Which package owns this file?
rpm -q --changelog nginx | head -30          # Recent changelog
rpm -q --scripts nginx                        # Pre/post install scripts
```

### Query RPM File (Before Installing)
```bash
rpm -qpi package.rpm                          # Info about RPM file
rpm -qpl package.rpm                          # Files in RPM file
rpm -qpR package.rpm                          # Dependencies of RPM file
```

### Install / Remove with rpm
```bash
# Only use rpm directly when dnf isn't appropriate
rpm -ivh package.rpm                          # Install (verbose + progress)
rpm -Uvh package.rpm                          # Upgrade (or install if not present)
rpm -Fvh package.rpm                          # Freshen (only if already installed)
rpm -evv package.rpm                          # Erase (verbose)
rpm -evv --nodeps package.rpm                 # Remove ignoring deps (dangerous!)
```

### RPM Database
```bash
rpm --rebuilddb                               # Rebuild RPM database (fix corruption)
rpm -Va                                       # Verify all installed packages
rpm -V nginx                                  # Verify specific package
# Output: S=size, M=mode, 5=MD5, L=symlink, D=device, U=user, G=group, T=time
```

---

## 3. Repository Management

### Viewing Repositories
```bash
dnf repolist                                  # Enabled repos
dnf repolist all                              # All repos (enabled + disabled)
dnf repolist --verbose
dnf repo-pkgs baseos list                     # Packages from specific repo
```

### Adding Repositories

**Method 1: DNF config-manager plugin**
```bash
dnf install dnf-plugins-core -y

# Add repo by URL
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Enable/Disable repos
dnf config-manager --enable powertools
dnf config-manager --disable powertools
```

**Method 2: Manual .repo file**
```bash
cat > /etc/yum.repos.d/nginx.repo << 'EOF'
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF
```

### Repo File Format
```ini
[repo-id]                           # Unique identifier — used in dnf commands
name=Repository Name                 # Human-readable name
baseurl=http://example.com/repo/    # Base URL (or mirrorlist=)
enabled=1                           # 1=enabled, 0=disabled
gpgcheck=1                          # 1=verify GPG signatures
gpgkey=https://example.com/key.gpg # GPG key URL
exclude=package1 package2           # Exclude specific packages
includepkgs=nginx*                  # Only include matching packages
priority=1                          # Lower = higher priority (needs dnf-plugin-priorities)
skip_if_unavailable=1               # Don't fail if repo is unreachable
sslverify=1                         # Verify SSL certificate
```

### EPEL (Extra Packages for Enterprise Linux)
```bash
# RHEL 8
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

# RHEL 9
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm

# Enable CodeReady Linux Builder (needed for EPEL deps)
subscription-manager repos --enable codeready-builder-for-rhel-9-x86_64-rpms
# For Rocky/AlmaLinux:
dnf config-manager --enable crb
```

---

## 4. Module Streams

RHEL 8+ Application Streams allow multiple versions of a software to coexist.

```bash
# List available modules
dnf module list
dnf module list nginx
dnf module list postgresql

# View module info
dnf module info nodejs

# Enable a specific stream
dnf module enable nodejs:18
dnf module enable postgresql:14

# Install from module stream
dnf module install nodejs:18
dnf module install nodejs:18/development   # Specific profile

# Switch to different stream version
dnf module reset nodejs               # Reset to default stream
dnf module switch-to nodejs:20        # Switch to specific stream

# Disable module (prevent packages from being installed)
dnf module disable nodejs:16

# View enabled streams
dnf module list --enabled
```

### Module Profiles
Profiles define which packages to install within a module:
```bash
# Common profiles: default, minimal, development, server, client
dnf module install nginx:1.22/default
dnf module install nodejs:18/development    # Includes npm, tools
dnf module install postgresql:14/server     # Server only
```

---

## 5. GPG Keys

```bash
# Import a GPG key
rpm --import https://nginx.org/keys/nginx_signing.key
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

# List imported keys
rpm -qa gpg-pubkey*
rpm -qi gpg-pubkey-<version>           # View key info

# Verify package signature
rpm --checksig nginx-1.24.0-1.rpm

# Configure GPG check per repo (in .repo file)
gpgcheck=1                              # Enforce (default, required)
gpgkey=https://example.com/key.gpg

# Verify all installed package signatures
rpm -Va | grep ^..5        # 5 = MD5 checksum mismatch
```

---

## 6. Local Repositories

Useful for air-gapped systems or caching packages.

```bash
# Install createrepo tool
dnf install createrepo_c -y

# Create a local repo from RPM files
mkdir -p /opt/localrepo
cp *.rpm /opt/localrepo/
createrepo_c /opt/localrepo/

# Update after adding more packages
createrepo_c --update /opt/localrepo/

# Create .repo file pointing to local repo
cat > /etc/yum.repos.d/local.repo << 'EOF'
[local]
name=Local Repository
baseurl=file:///opt/localrepo/
enabled=1
gpgcheck=0
EOF

dnf clean all
dnf repolist

# Sync a remote repo locally (for offline/air-gap)
dnf install dnf-utils -y
reposync --repo=epel --download-metadata -p /opt/repos/
createrepo_c /opt/repos/epel/
```

### HTTP-Served Local Repo
```bash
# Serve repo over HTTP
dnf install httpd -y
cp -r /opt/localrepo /var/www/html/repo
systemctl start httpd

# Point clients to HTTP repo
cat > /etc/yum.repos.d/http-local.repo << 'EOF'
[http-local]
name=HTTP Local Repo
baseurl=http://repo-server.local/repo/
enabled=1
gpgcheck=0
EOF
```

---

## 7. dnf History

dnf keeps a transaction history allowing rollback.

```bash
# View history
dnf history                             # List all transactions
dnf history list                        # Same
dnf history info 5                      # Details of transaction 5
dnf history info last                   # Last transaction

# Undo a transaction
dnf history undo 5                      # Undo transaction 5 (removes installed, reinstalls removed)
dnf history undo last                   # Undo last transaction

# Redo a transaction
dnf history redo 5

# Rollback to before transaction N
dnf history rollback 5                  # Restore state before transaction 5

# List packages changed in a transaction
dnf history info 5 | grep -E "Install|Upgrade|Erase"
```

---

## 8. Do's and Don'ts

### ✅ Do's

**Do always use `dnf` (not `rpm -i`) for installing packages with dependencies:**
```bash
dnf install nginx        # Resolves and installs all dependencies automatically
```

**Do verify package integrity before installing:**
```bash
rpm --checksig package.rpm       # Check GPG signature
```

**Do enable GPG checking for all repos:**
```bash
gpgcheck=1              # Always in your .repo files
gpgkey=...              # Always specify the key
```

**Do use `dnf history undo` instead of manually removing:**
```bash
# After a failed upgrade or accidental install:
dnf history undo last   # Cleaner than manual rpm -e
```

**Do use `dnf provides` to find the package for a missing file:**
```bash
# "command not found: netstat"
dnf provides netstat            # → net-tools
dnf install net-tools
```

**Do clean up regularly:**
```bash
dnf autoremove           # Remove unused dependencies
dnf clean all            # Free up cache space
```

### ❌ Don'ts

**Don't use `rpm -e` on packages with dependents:**
```bash
# ❌ Breaks packages that depend on this one
rpm -evv --nodeps openssl

# ✅ Let dnf handle it
dnf remove openssl    # dnf warns if other packages depend on it
```

**Don't skip GPG checks:**
```bash
# ❌ Security risk — installs unsigned packages
dnf install package --nogpgcheck
rpm -ivh --nosignature package.rpm

# ✅ Only bypass GPG for testing, and only from trusted sources
```

**Don't use `dnf update *` without checking what will change:**
```bash
dnf check-update        # Review what will be updated first
# Then:
dnf update              # Apply after review
```

**Don't install EPEL packages on RHEL without checking support implications:**
```bash
# EPEL packages are not supported by Red Hat
# For production RHEL with support contract, check vendor support policy first
```

---

## Quick Reference

```bash
# Install/Remove
dnf install nginx
dnf remove nginx
dnf autoremove

# Search/Info
dnf search nginx
dnf info nginx
dnf provides /usr/sbin/nginx

# Update
dnf check-update
dnf update
dnf update --security

# Repos
dnf repolist
dnf config-manager --add-repo URL
dnf config-manager --enable/--disable REPO

# Module streams
dnf module list
dnf module enable nodejs:18
dnf module install nodejs:18

# RPM queries
rpm -qa | grep nginx
rpm -ql nginx
rpm -qf /usr/bin/python3
rpm -V nginx

# History
dnf history
dnf history undo last
```

---

> **Prev:** [09 — SELinux Mastery](./09-selinux.md)  
> **Next:** [11 — Shell Scripting for SysAdmins](./11-shell-scripting.md)  
> **Repository:** [github.com/eknatha/linux](https://github.com/eknatha/linux)
