# 09 — SELinux Mastery (RHEL)

> **Author:** Eknatha Reddy P · [eknathalabs.com](https://eknathalabs.com) · [github.com/eknatha](https://github.com/eknatha)  
> **Applies to:** RHEL 8/9, CentOS Stream, Rocky Linux, AlmaLinux  
> **Level:** Advanced

---

## Table of Contents
1. [SELinux Concepts](#1-selinux-concepts)
2. [Modes & Status](#2-modes--status)
3. [SELinux Contexts](#3-selinux-contexts)
4. [Managing Contexts](#4-managing-contexts)
5. [Booleans](#5-booleans)
6. [Troubleshooting Denials](#6-troubleshooting-denials)
7. [Custom Policies with audit2allow](#7-custom-policies)
8. [Port Labels](#8-port-labels)
9. [Common Scenarios](#9-common-scenarios)
10. [Do's and Don'ts](#10-dos-and-donts)

---

## 1. SELinux Concepts

**SELinux (Security-Enhanced Linux)** implements **Mandatory Access Control (MAC)** in the kernel. Unlike DAC (file permissions), MAC decisions are made by a policy that even root cannot override.

### How SELinux Works
Every process and file has a **security context** (label). The kernel checks the **policy** before every operation:
```
Process (context) → attempts operation → on Object (context)
                          ↓
                   Policy check: ALLOW or DENY
                          ↓
               Logged in /var/log/audit/audit.log
```

### SELinux Architecture
```
Policy (rules)
  ├── Type Enforcement (TE) — main rules: what processes can do to objects
  ├── Role-Based Access Control (RBAC) — roles (staff_r, sysadm_r, unconfined_r)
  └── Multi-Level Security (MLS) — sensitivity levels (optional)

Context format: user:role:type:level
Example: system_u:system_r:httpd_t:s0
         │         │        │       └── Level (MLS)
         │         │        └────────── Type (most important part)
         │         └─────────────────── Role
         └───────────────────────────── SELinux user
```

---

## 2. Modes & Status

### SELinux Modes
| Mode | Description |
|------|-------------|
| `Enforcing` | Policy enforced, denials logged |
| `Permissive` | Policy NOT enforced, denials still logged (debugging) |
| `Disabled` | SELinux completely off (requires reboot to enable) |

```bash
# Check status
sestatus
getenforce                          # Quick: Enforcing / Permissive / Disabled

# Temporary mode change (survives until reboot)
setenforce 0                        # → Permissive (for debugging)
setenforce 1                        # → Enforcing (back to production)
setenforce Permissive               # By name

# Permanent mode change
# Edit /etc/selinux/config:
SELINUX=enforcing    # enforcing | permissive | disabled
# Changes require REBOOT

# Enable SELinux after disabled (requires relabeling)
# 1. Set SELINUX=permissive in /etc/selinux/config
# 2. touch /.autorelabel
# 3. Reboot (relabeling happens — takes minutes)
# 4. After boot, set SELINUX=enforcing
# 5. Reboot again
```

### sestatus Output
```
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux mount point:            /sys/fs/selinux
Loaded policy name:             targeted
Current mode:                   enforcing          ← Operational mode
Mode from config file:          enforcing          ← Config file mode
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```

### Policy Types
| Policy | Description |
|--------|-------------|
| `targeted` | Only specific daemons confined (default on RHEL) |
| `mls` | Multi-level security (government use) |
| `minimum` | Subset of targeted |

---

## 3. SELinux Contexts

### Viewing Contexts
```bash
# Files/directories
ls -Z /var/www/html/
ls -laZ /etc/nginx/

# Processes
ps auxZ | grep nginx
ps -eZ | grep httpd_t

# Current user
id -Z

# Sockets
ss -Z -tlnp
```

### Context Format
```
system_u:object_r:httpd_sys_content_t:s0
│         │        │                   └── Level
│         │        └───────────────────── Type (most important!)
│         └────────────────────────────── Role (object_r for files)
└──────────────────────────────────────── SELinux user
```

### Common Types
| Type | Used For |
|------|---------|
| `httpd_t` | Apache/nginx process |
| `httpd_sys_content_t` | Web content files |
| `httpd_sys_rw_content_t` | Web content that needs write |
| `sshd_t` | SSH daemon process |
| `mysqld_t` | MySQL/MariaDB process |
| `container_t` | Container processes |
| `unconfined_t` | Unrestricted (most user processes) |

---

## 4. Managing Contexts

### chcon — Temporary Context Change
```bash
# Change type context (NOT persistent — overwritten by restorecon or relabeling)
chcon -t httpd_sys_content_t /opt/mywebsite/
chcon -R -t httpd_sys_content_t /opt/mywebsite/    # Recursive

# Copy context from reference file
chcon --reference=/var/www/html /opt/mywebsite/
```

### semanage fcontext — Persistent Context Rules
```bash
# Add a persistent file context rule (survives restorecon and relabeling)
semanage fcontext -a -t httpd_sys_content_t "/opt/mywebsite(/.*)?"
semanage fcontext -a -t httpd_sys_rw_content_t "/opt/mywebsite/uploads(/.*)?"

# Apply the rule to existing files
restorecon -Rv /opt/mywebsite/

# View all file context rules
semanage fcontext -l
semanage fcontext -l | grep httpd

# Delete a rule
semanage fcontext -d "/opt/mywebsite(/.*)?"

# Modify existing rule
semanage fcontext -m -t httpd_sys_rw_content_t "/opt/mywebsite/uploads(/.*)?"
```

### restorecon — Restore to Policy Default
```bash
restorecon /var/www/html/index.html          # Single file
restorecon -Rv /var/www/html/               # Recursive + verbose
restorecon -Rn /var/www/html/               # Dry run (what would change)

# Full system relabel (after enabling SELinux or fixing major issues)
touch /.autorelabel && reboot
# or: restorecon -R /   (slow, may miss some files)
```

---

## 5. Booleans

Booleans are **named on/off switches** in the SELinux policy that allow common access patterns without custom policies.

```bash
# List all booleans
getsebool -a
getsebool -a | grep httpd
semanage boolean -l                         # With description

# Get specific boolean
getsebool httpd_can_network_connect

# Set boolean
setsebool httpd_can_network_connect on      # Temporary (until reboot)
setsebool -P httpd_can_network_connect on   # Persistent (-P = permanent)
setsebool -P httpd_can_network_connect=1    # Alternative syntax
```

### Common Booleans
```bash
# Web server
httpd_can_network_connect       # Allow httpd to make network connections
httpd_can_network_connect_db    # Allow httpd to connect to databases
httpd_can_sendmail              # Allow httpd to send email
httpd_use_nfs                   # Allow httpd to access NFS
httpd_use_cifs                  # Allow httpd to access CIFS/Samba
httpd_execmem                   # Allow httpd to execute memory

# SSH
sshd_enable_x11_forwarding      # Allow X11 forwarding in SSH

# FTP
ftp_home_dir                    # Allow FTP to access home dirs
ftpd_anon_write                 # Allow anonymous FTP write

# Networking
nis_enabled                     # Allow NIS use
use_nfs_home_dirs               # Allow users with NFS home dirs to log in
use_samba_home_dirs             # Allow users with Samba home dirs to log in

# DNS
named_write_master_zones        # Allow named to write to master zones
```

---

## 6. Troubleshooting Denials

### Finding Denials
```bash
# ausearch — query audit log
ausearch -m avc                              # All AVC denials
ausearch -m avc -ts recent                  # Recent denials (last 10 min)
ausearch -m avc -ts today                   # Today's denials
ausearch -m avc -c nginx                    # Denials for nginx command
ausearch -m avc -p 1234                     # Denials for PID

# audit2why — explain why a denial happened
ausearch -m avc -ts recent | audit2why

# journalctl for SELinux messages
journalctl | grep -i selinux
journalctl | grep avc

# sealert (best tool for human-readable explanations)
sudo dnf install setroubleshoot-server -y
sealert -a /var/log/audit/audit.log         # Analyze all denials
sealert -l "*"                              # List all alerts
```

### Reading an AVC Denial
```
type=AVC msg=audit(1691840000.123:456): avc:  denied
{ name_connect }
for pid=1234 comm="nginx" name="3306" scontext=system_u:system_r:httpd_t:s0
    tcontext=system_u:object_r:mysqld_port_t:s0 tclass=tcp_socket permissive=0

Breaking it down:
  denied { name_connect }     — What was denied: connecting by name
  pid=1234 comm="nginx"       — The process: nginx PID 1234
  scontext=...httpd_t         — Source context: nginx (httpd_t type)
  tcontext=...mysqld_port_t   — Target context: MySQL port (3306)
  tclass=tcp_socket           — Object class: TCP socket
  permissive=0                — Enforcing mode (1 = permissive)
```

### Using audit2allow for Custom Policies
```bash
# Generate a policy module from denials
ausearch -m avc -ts recent | audit2allow -M mypolicy

# This creates:
# mypolicy.te   — Text policy file (human-readable)
# mypolicy.pp   — Compiled binary policy

# Review the .te file before loading!
cat mypolicy.te

# Load the policy
semodule -i mypolicy.pp

# Verify it's loaded
semodule -l | grep mypolicy

# Remove a policy module
semodule -r mypolicy
```

---

## 7. Custom Policies

### Writing a Policy from Scratch

```bash
# Example: nginx needs to connect to port 3306 (MySQL)
# Step 1: Check if a boolean exists first
getsebool -a | grep httpd | grep db   # httpd_can_network_connect_db

# Step 2: Try the boolean first
setsebool -P httpd_can_network_connect_db on

# Step 3: If boolean doesn't cover it, generate custom policy
ausearch -m avc -ts recent -c nginx | audit2allow -M nginx-mysql

# Step 4: Review
cat nginx-mysql.te
# module nginx-mysql 1.0;
# require { type httpd_t; type mysqld_port_t; class tcp_socket name_connect; }
# allow httpd_t mysqld_port_t:tcp_socket name_connect;

# Step 5: Load
semodule -i nginx-mysql.pp

# Step 6: Test and verify
systemctl restart nginx
# Try the denied operation
ausearch -m avc -ts recent   # Should be clean now
```

### Type Enforcement File Structure
```
module <name> <version>;

require {
    type <domain>;
    type <type>;
    class <class> <permission>;
}

allow <domain> <type>:<class> { <permissions> };
```

---

## 8. Port Labels

```bash
# View port labels
semanage port -l
semanage port -l | grep http
semanage port -l | grep 8080

# Add a port label (let nginx use port 8443)
semanage port -a -t http_port_t -p tcp 8443
semanage port -a -t http_port_t -p tcp 8080

# Modify existing
semanage port -m -t http_port_t -p tcp 8080

# Delete
semanage port -d -t http_port_t -p tcp 8443

# Common port types
# http_port_t: 80, 443, 8008, 8009, 8443
# ssh_port_t: 22
# mysqld_port_t: 3306
# redis_port_t: 6379
# postgresql_port_t: 5432
```

---

## 9. Common Scenarios

### Nginx Serving Files from /opt
```bash
# Problem: nginx can't read /opt/mywebsite (wrong SELinux context)

# Fix:
semanage fcontext -a -t httpd_sys_content_t "/opt/mywebsite(/.*)?"
restorecon -Rv /opt/mywebsite/

# For directories that nginx needs to write to:
semanage fcontext -a -t httpd_sys_rw_content_t "/opt/mywebsite/uploads(/.*)?"
restorecon -Rv /opt/mywebsite/uploads/
```

### Nginx/Apache Connecting to Backend Service
```bash
# Problem: nginx can't proxy to backend app
setsebool -P httpd_can_network_connect on
# or for database:
setsebool -P httpd_can_network_connect_db on
```

### Custom SSH Port
```bash
# Problem: sshd won't bind to port 2222
semanage port -a -t ssh_port_t -p tcp 2222
# Or if already in list:
semanage port -m -t ssh_port_t -p tcp 2222
```

### Application Writing to /tmp or Custom Dir
```bash
# Allow app to write to /opt/myapp/run/
semanage fcontext -a -t tmp_t "/opt/myapp/run(/.*)?"
restorecon -Rv /opt/myapp/run/
# or more specific to the app type:
semanage fcontext -a -t myapp_var_run_t "/opt/myapp/run(/.*)?"
```

---

## 10. Do's and Don'ts

### ✅ Do's

**Do keep SELinux in Enforcing mode in production:**
```bash
sestatus | grep "Current mode"    # Should show: enforcing
```

**Do use Permissive mode for debugging, not Disabled:**
```bash
setenforce 0          # Permissive: denials still logged!
# Debug, find denials in audit.log
# Fix with boolean, fcontext, or custom policy
setenforce 1          # Back to enforcing
```

**Do check booleans before writing custom policies:**
```bash
getsebool -a | grep httpd
semanage boolean -l | grep -i network    # Search by description
# A boolean is always simpler than a custom policy
```

**Do use `semanage fcontext` + `restorecon` (not just chcon):**
```bash
# chcon: temporary — survives until restorecon or relabeling
# semanage fcontext + restorecon: PERMANENT — survives everything
semanage fcontext -a -t httpd_sys_content_t "/myapp(/.*)?"
restorecon -Rv /myapp/
```

**Do review auto-generated policies before loading:**
```bash
cat mypolicy.te    # Read what audit2allow generated — could be too permissive
```

### ❌ Don'ts

**NEVER run `setenforce 0` permanently in production:**
```bash
# ❌ Disables all MAC protection
setenforce 0    # Only for debugging, switch back immediately after

# ❌ Worst: permanent disable in config
SELINUX=disabled    # /etc/selinux/config — never for production
```

**Don't set SELINUX=disabled as a "quick fix":**
```bash
# Every denial has a correct fix:
# 1. Wrong file context → semanage fcontext + restorecon
# 2. Needs network → boolean
# 3. Custom port → semanage port
# 4. Unique need → audit2allow + semodule
# There is ALWAYS a proper solution
```

**Don't blindly apply audit2allow output:**
```bash
# audit2allow may generate OVERLY permissive policies
# Always review mypolicy.te before semodule -i
# Narrow scope: specific type, specific permission
```

---

## Quick Reference

```bash
# Status
sestatus
getenforce
setenforce 0|1              # Temporary permissive/enforcing

# Contexts
ls -Z /var/www/html/
ps auxZ | grep httpd
chcon -t httpd_sys_content_t /opt/web/   # Temporary
semanage fcontext -a -t httpd_sys_content_t "/opt/web(/.*)?"   # Permanent
restorecon -Rv /opt/web/

# Booleans
getsebool -a | grep httpd
setsebool -P httpd_can_network_connect on

# Ports
semanage port -l | grep http
semanage port -a -t http_port_t -p tcp 8080

# Troubleshoot
ausearch -m avc -ts recent | audit2why
ausearch -m avc -ts recent | audit2allow -M mypol
semodule -i mypol.pp
semodule -l | grep mypol
```

---

> **Prev:** [08 — firewalld & iptables](./08-firewalld-iptables.md)  
> **Next:** [10 — Package Management (RHEL)](./10-package-management-rhel.md)  
> **Repository:** [github.com/eknatha/linux](https://github.com/eknatha/linux)
