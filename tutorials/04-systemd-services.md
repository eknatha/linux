# 04 — systemd & Service Management (RHEL)

> **Author:** Eknatha Reddy P · [eknathalabs.com](https://eknathalabs.com) · [github.com/eknatha](https://github.com/eknatha)  
> **Applies to:** RHEL 8/9, CentOS Stream, Rocky Linux, AlmaLinux  
> **Level:** Intermediate

---

## Table of Contents
1. [systemd Overview](#1-systemd-overview)
2. [systemctl — Service Control](#2-systemctl)
3. [Unit Files — Anatomy](#3-unit-files)
4. [Writing a Service Unit](#4-writing-a-service-unit)
5. [Targets (Runlevels)](#5-targets)
6. [journalctl — Log Management](#6-journalctl)
7. [Timer Units](#7-timer-units)
8. [Drop-in Overrides](#8-drop-in-overrides)
9. [Boot Analysis](#9-boot-analysis)
10. [Do's and Don'ts](#10-dos-and-donts)

---

## 1. systemd Overview

**systemd** is the init system and service manager for RHEL 8/9. PID 1 is always `systemd`.

```
PID 1 → systemd
  ├── Units: .service, .socket, .timer, .mount, .target, .path, .slice ...
  ├── Manages: startup order, dependencies, logging (journald), cgroups
  └── Replaces: SysVinit, Upstart, cron (partially), /etc/rc.d
```

### Unit File Locations (Priority Order)
```
/etc/systemd/system/        ← Admin-created / overrides (highest priority)
/run/systemd/system/        ← Runtime (not persistent)
/lib/systemd/system/        ← Vendor defaults from packages (don't edit)
/usr/lib/systemd/system/    ← Same as above (RHEL 8+)
```

---

## 2. systemctl

### Service Lifecycle
```bash
# State control
systemctl start nginx
systemctl stop nginx
systemctl restart nginx        # stop + start
systemctl reload nginx         # Send SIGHUP (reload config without restart)
systemctl reload-or-restart nginx  # Try reload, fall back to restart

# Enable/Disable (survive reboots)
systemctl enable nginx         # Creates symlinks in /etc/systemd/system/*.wants/
systemctl disable nginx        # Removes symlinks
systemctl enable --now nginx   # Enable + start immediately
systemctl disable --now nginx  # Disable + stop immediately

# Status
systemctl status nginx
systemctl is-active nginx      # Returns: active / inactive / failed
systemctl is-enabled nginx     # Returns: enabled / disabled / masked
systemctl is-failed nginx      # Returns: failed / active / inactive

# Masking (stronger than disable)
systemctl mask postfix         # Symlinks to /dev/null — cannot be started
systemctl unmask postfix
```

### Listing Units
```bash
systemctl list-units                          # All active units
systemctl list-units --failed                 # Only failed units
systemctl list-units --type=service           # Only services
systemctl list-units --type=service --state=running
systemctl list-unit-files                     # All unit files + state
systemctl list-unit-files --type=service
systemctl list-dependencies nginx             # What nginx depends on
systemctl list-dependencies --reverse nginx   # What depends on nginx
```

### System Control
```bash
systemctl daemon-reload        # Reload unit files after any edit
systemctl daemon-reexec        # Re-execute systemd itself (rare)
systemctl poweroff
systemctl reboot
systemctl halt
systemctl rescue               # Emergency single-user mode
systemctl emergency            # Minimal emergency mode
```

---

## 3. Unit Files

### Unit Types
| Extension | Purpose |
|-----------|---------|
| `.service` | System service (most common) |
| `.socket` | Socket-based activation |
| `.timer` | Scheduled tasks (cron replacement) |
| `.mount` | Filesystem mount points |
| `.automount` | Auto-mount on access |
| `.target` | Grouping units / runlevel equivalent |
| `.path` | Path-based activation |
| `.slice` | cgroup hierarchy management |
| `.scope` | Externally created processes |
| `.device` | Kernel device |

### Anatomy of a .service File
```ini
[Unit]
Description=My Web Application          # Short description
Documentation=https://docs.myapp.com    # Docs URL
After=network.target postgresql.service # Start AFTER these
Requires=postgresql.service             # Hard dependency (fails if postgres fails)
Wants=redis.service                     # Soft dependency (starts redis, continues if fails)
ConditionPathExists=/opt/myapp/app.py   # Only start if path exists

[Service]
Type=simple                     # Process type (see below)
User=myapp                      # Run as this user
Group=myapp
WorkingDirectory=/opt/myapp
Environment=NODE_ENV=production
EnvironmentFile=/etc/myapp/env  # Load env vars from file
ExecStartPre=/opt/myapp/pre-start.sh    # Run before ExecStart
ExecStart=/usr/bin/python3 /opt/myapp/app.py
ExecStartPost=/opt/myapp/post-start.sh
ExecReload=/bin/kill -HUP $MAINPID     # Reload command
ExecStop=/opt/myapp/stop.sh
Restart=on-failure              # Restart policy
RestartSec=5s                   # Wait 5s before restart
StartLimitIntervalSec=60        # Reset failure counter after 60s
StartLimitBurst=5               # Allow up to 5 restarts in that window
KillMode=process                # Kill only main process (not child processes)
TimeoutStartSec=30              # Startup timeout
TimeoutStopSec=30               # Shutdown timeout

# Security hardening
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/var/lib/myapp /var/log/myapp

# Resource limits
LimitNOFILE=65535
MemoryMax=512M
CPUQuota=80%

[Install]
WantedBy=multi-user.target      # Enable at which target
```

### Service Types
| Type | Description | Use Case |
|------|-------------|----------|
| `simple` | ExecStart is the main process | Most services |
| `exec` | Like simple, but waits for exec to succeed | RHEL 9+ |
| `forking` | Process forks and parent exits | Traditional daemons |
| `oneshot` | Runs once, exits; systemd waits | Scripts, migrations |
| `notify` | Sends ready notification via sd_notify() | Advanced daemons |
| `dbus` | Ready when D-Bus name acquired | Desktop services |
| `idle` | Starts after all jobs complete | Background tasks |

### Restart Policies
| Value | Restarts on |
|-------|-------------|
| `no` | Never (default) |
| `on-success` | Clean exit only |
| `on-failure` | Non-zero exit, signal |
| `on-abnormal` | Signal or watchdog timeout |
| `always` | Always restarts |
| `on-abort` | Only on unclean signal |

---

## 4. Writing a Service Unit

### Example: Node.js API Service
```bash
# 1. Create the user
useradd -r -s /sbin/nologin nodeapi

# 2. Create the unit file
sudo tee /etc/systemd/system/nodeapi.service << 'EOF'
[Unit]
Description=Node.js API Service
Documentation=https://github.com/eknatha/nodeapi
After=network.target
Wants=postgresql.service

[Service]
Type=simple
User=nodeapi
Group=nodeapi
WorkingDirectory=/opt/nodeapi
Environment=NODE_ENV=production
Environment=PORT=3000
ExecStart=/usr/bin/node /opt/nodeapi/server.js
Restart=on-failure
RestartSec=5s
StartLimitIntervalSec=120
StartLimitBurst=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier=nodeapi
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ReadWritePaths=/var/lib/nodeapi /var/log/nodeapi
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

# 3. Reload daemon and start
systemctl daemon-reload
systemctl enable --now nodeapi

# 4. Verify
systemctl status nodeapi
journalctl -u nodeapi -f
```

### Example: One-Shot Migration Script
```ini
[Unit]
Description=Database migration for v2.0
After=postgresql.service
ConditionPathExists=!/var/lib/myapp/.migrated

[Service]
Type=oneshot
User=myapp
ExecStart=/opt/myapp/migrate.sh
ExecStartPost=/usr/bin/touch /var/lib/myapp/.migrated
RemainAfterExit=yes             # Consider service "active" after exit

[Install]
WantedBy=multi-user.target
```

---

## 5. Targets

Targets are **groups of units** — equivalent to SysVinit runlevels.

```bash
# View current target
systemctl get-default

# Set default target
systemctl set-default multi-user.target    # No GUI (server)
systemctl set-default graphical.target     # GUI

# Switch target at runtime
systemctl isolate multi-user.target        # Switch immediately

# Target equivalents
# SysVinit 0 = poweroff.target
# SysVinit 1 = rescue.target
# SysVinit 3 = multi-user.target (server default)
# SysVinit 5 = graphical.target
# SysVinit 6 = reboot.target
```

### Common Targets
| Target | Description |
|--------|-------------|
| `poweroff.target` | Shutdown |
| `rescue.target` | Single user mode (root password required) |
| `emergency.target` | Minimal — only root filesystem |
| `multi-user.target` | Full multi-user, no GUI |
| `graphical.target` | Full multi-user + GUI |
| `reboot.target` | Reboot |
| `network.target` | Network configured |
| `network-online.target` | Network truly online (check connectivity) |

---

## 6. journalctl

systemd's **journald** collects logs from all services, kernel, and systemd itself.

```bash
# Basic usage
journalctl                              # All logs (oldest first)
journalctl -r                           # Reverse (newest first)
journalctl -f                           # Follow (like tail -f)
journalctl -n 100                       # Last 100 lines

# Filter by service
journalctl -u nginx
journalctl -u nginx -u php-fpm          # Multiple services
journalctl -u "myapp*"                  # Wildcard

# Filter by time
journalctl --since "2024-08-12 10:00:00"
journalctl --until "2024-08-12 11:00:00"
journalctl --since "1 hour ago"
journalctl --since yesterday
journalctl --since today

# Filter by priority (emergency, alert, crit, err, warning, notice, info, debug)
journalctl -p err                       # Errors and above
journalctl -p warning..err              # Warning to error range
journalctl -p crit -u nginx            # Critical logs from nginx

# Kernel messages
journalctl -k                           # Kernel (dmesg equivalent)
journalctl -k --since "30 min ago"

# Boot sessions
journalctl -b                           # Current boot
journalctl -b -1                        # Previous boot
journalctl --list-boots                 # All recorded boots

# Format options
journalctl -u nginx --no-pager         # No paging (for scripts)
journalctl -u nginx -o json            # JSON format
journalctl -u nginx -o json-pretty
journalctl -u nginx -o cat             # Message only (no metadata)
journalctl -u nginx -o short-precise   # Full timestamps

# Disk usage
journalctl --disk-usage
journalctl --vacuum-size=500M          # Keep max 500MB of logs
journalctl --vacuum-time=30d           # Keep logs for 30 days

# Search
journalctl -u nginx | grep "error"
journalctl _COMM=python3               # By command name
journalctl _PID=1234                   # By PID
journalctl _UID=1000                   # By user ID
```

### Configure journald Retention
```bash
# /etc/systemd/journald.conf
sudo tee -a /etc/systemd/journald.conf << 'EOF'
[Journal]
SystemMaxUse=2G
SystemKeepFree=500M
MaxRetentionSec=90day
MaxFileSec=1week
Compress=yes
EOF

systemctl restart systemd-journald
```

---

## 7. Timer Units

systemd timers replace cron for better integration with systemd.

### Monotonic Timer (Relative to System Events)
```bash
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily Backup Timer

[Timer]
OnBootSec=5min           # First run: 5 min after boot
OnUnitActiveSec=24h      # Then every 24 hours
Persistent=true          # Run missed jobs on next start

[Install]
WantedBy=timers.target
```

### Calendar Timer (Cron-Like)
```bash
# /etc/systemd/system/cleanup.timer
[Unit]
Description=Weekly Cleanup Timer

[Timer]
OnCalendar=weekly                      # Every Sunday 00:00
OnCalendar=Mon..Fri 09:00:00           # Weekdays at 9am
OnCalendar=*-*-* 02:30:00             # Daily at 02:30
OnCalendar=2024-08-12 10:00:00        # One-time
RandomizedDelaySec=30min              # Spread load (avoid thundering herd)
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
# The .service file that does the actual work
# /etc/systemd/system/backup.service
[Unit]
Description=Daily Backup Job

[Service]
Type=oneshot
User=backup
ExecStart=/opt/scripts/backup.sh
```

```bash
# Manage timers
systemctl enable --now backup.timer
systemctl list-timers                    # Show all timers + next trigger
systemctl list-timers --all              # Include inactive
journalctl -u backup.service             # Check last run logs

# Test calendar expressions
systemd-analyze calendar "Mon..Fri 09:00"
systemd-analyze calendar weekly
```

---

## 8. Drop-in Overrides

Never edit `/usr/lib/systemd/system/` directly — use overrides.

```bash
# Method 1: systemctl edit (RECOMMENDED)
systemctl edit nginx
# Creates /etc/systemd/system/nginx.service.d/override.conf
# Opens editor with current unit context

# Method 2: Full replacement
systemctl edit --full nginx
# Creates /etc/systemd/system/nginx.service (full copy)

# Example override — increase file limit and add env var
# /etc/systemd/system/nginx.service.d/override.conf
[Service]
LimitNOFILE=100000
Environment=NGINX_WORKERS=4
Restart=always
RestartSec=3s

# Apply
systemctl daemon-reload
systemctl restart nginx

# View effective configuration (merged)
systemctl cat nginx

# Remove override
rm -rf /etc/systemd/system/nginx.service.d/
systemctl daemon-reload
```

---

## 9. Boot Analysis

```bash
# Total boot time breakdown
systemd-analyze
# Startup finished in 2.341s (kernel) + 3.891s (initrd) + 8.203s (userspace) = 14.435s

# Per-unit timing
systemd-analyze blame | head -20

# Critical path (what's slowing boot)
systemd-analyze critical-chain

# Full boot timeline SVG
systemd-analyze plot > /tmp/boot-timeline.svg

# Check for unit dependency problems
systemd-analyze verify /etc/systemd/system/myapp.service
```

---

## 10. Do's and Don'ts

### ✅ Do's

**Do always run daemon-reload after editing unit files:**
```bash
systemctl daemon-reload        # Must do this after any /etc/systemd/system/ change
```

**Do use `systemctl edit` for overrides, not direct edits to /usr/lib:**
```bash
systemctl edit nginx           # Creates override.conf automatically
```

**Do specify proper Restart= for production services:**
```bash
Restart=on-failure             # Minimum for production
RestartSec=5s
StartLimitIntervalSec=120
StartLimitBurst=5
```

**Do use `After=network-online.target` for services needing real network:**
```bash
# network.target = networking configured
# network-online.target = networking truly connected (DNS works)
After=network-online.target
Wants=network-online.target
```

**Do use `StandardOutput=journal` and `SyslogIdentifier`:**
```bash
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp          # Filter with: journalctl -t myapp
```

### ❌ Don'ts

**Don't edit files in /usr/lib/systemd/system/ directly:**
```bash
# ❌ Gets overwritten on package update
vi /usr/lib/systemd/system/nginx.service

# ✅ Use override
systemctl edit nginx
```

**Don't use `Type=forking` unless the daemon actually forks:**
```bash
# Wrong type causes systemd to track wrong PID
# Most modern apps use simple/notify
```

**Don't forget `RemainAfterExit=yes` for oneshot services:**
```bash
# Without it, systemctl status shows "inactive" even on success
Type=oneshot
RemainAfterExit=yes
```

**Don't use `Restart=always` for oneshot units:**
```bash
# Type=oneshot + Restart=always = infinite restart loop
# Use on-failure for oneshot
```

**Don't mask critical units without a plan:**
```bash
systemctl mask firewalld       # ❌ Now you can't start it at all, even manually
# Use disable if you just don't want it at boot
```

---

## Quick Reference

```bash
# Service control
systemctl start|stop|restart|reload|status nginx
systemctl enable --now nginx
systemctl disable --now nginx
systemctl mask|unmask postfix

# Unit files
systemctl daemon-reload
systemctl edit nginx              # Create override
systemctl cat nginx               # View merged config
systemctl list-units --failed

# journalctl
journalctl -u nginx -f            # Follow nginx logs
journalctl -u nginx --since "1 hour ago" -p err
journalctl -b                     # This boot
journalctl -k                     # Kernel messages

# Timers
systemctl list-timers
systemd-analyze calendar weekly

# Boot
systemd-analyze blame | head -20
systemd-analyze critical-chain
```

---

> **Prev:** [03 — Process Management](./03-process-management.md)  
> **Next:** [05 — Storage & Partitioning](./05-storage-partitioning.md)  
> **Repository:** [github.com/eknatha/linux](https://github.com/eknatha/linux)
