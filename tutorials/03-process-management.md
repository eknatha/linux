# 03 — Process Management (RHEL)

> **Author:** Eknatha Reddy P · [eknathalabs.com](https://eknathalabs.com) · [github.com/eknatha](https://github.com/eknatha)  
> **Applies to:** RHEL 8/9, CentOS Stream, Rocky Linux, AlmaLinux  
> **Level:** Intermediate

---

## Table of Contents
1. [Process Basics & States](#1-process-basics--states)
2. [Viewing Processes — ps](#2-viewing-processes)
3. [Real-time Monitoring — top & htop](#3-real-time-monitoring)
4. [Signals & kill](#4-signals--kill)
5. [Background Jobs](#5-background-jobs)
6. [Priority — nice & renice](#6-priority--nice--renice)
7. [Resource Limits — ulimit](#7-resource-limits--ulimit)
8. [/proc Filesystem](#8-proc-filesystem)
9. [OOM Killer](#9-oom-killer)
10. [cgroups (Control Groups)](#10-cgroups)
11. [Do's and Don'ts](#11-dos-and-donts)

---

## 1. Process Basics & States

Every running program is a **process** with a unique **PID** (Process ID). Processes form a tree rooted at PID 1 (systemd on RHEL).

### Process States
| State | Code | Description |
|-------|------|-------------|
| Running | R | Executing on CPU or in run queue |
| Sleeping | S | Waiting for an event (interruptible) |
| Disk Sleep | D | Waiting for I/O (uninterruptible — cannot be killed!) |
| Stopped | T | Suspended by signal (Ctrl+Z or SIGSTOP) |
| Zombie | Z | Completed but parent hasn't read exit status |
| Idle | I | Kernel idle thread |

```bash
# D state processes indicate I/O bottleneck or kernel hung
# Z state processes mean the parent process has a bug (not reading wait())
ps aux | grep Z    # Find zombies
ps aux | grep D    # Find I/O-blocked processes
```

### Key Process IDs
```bash
$$          # Current shell PID
$!          # PID of last background command
$PPID       # Parent PID
```

---

## 2. Viewing Processes

### ps — Process Snapshot
```bash
# Most common combinations
ps aux                          # All processes, BSD format
ps -ef                          # All processes, POSIX format
ps aux --sort=-%cpu | head -10  # Top CPU consumers
ps aux --sort=-%mem | head -10  # Top memory consumers
ps -u eknatha                   # Processes for a user
ps -p 1234                      # Specific PID
ps --ppid 1                     # Children of PID 1 (systemd services)

# Process tree
ps -ejH                         # Forest/tree view
ps axjf                         # ASCII tree
pstree                          # Visual tree
pstree -p                       # Include PIDs
pstree -u eknatha               # Only user's processes
```

### Column Meanings (ps aux)
```
USER  PID  %CPU  %MEM  VSZ      RSS     TTY  STAT  START  TIME  COMMAND
root  1     0.0   0.3   171000  10000   ?    Ss    09:00  0:02  /usr/lib/systemd/systemd
│     │      │     │     │        │      │    ││     │      │
│     │      │     │     │        └ RSS: Resident (physical) RAM in KB
│     │      │     │     └─── VSZ: Virtual memory size in KB
│     │      │     └────────── % of physical RAM
│     │      └──────────────── % of CPU time
│     └─────────────────────── Process ID
└───────────────────────────── User running the process
```

### pidof, pgrep, pkill
```bash
pidof nginx                     # Get PID(s) of nginx
pidof -s sshd                   # Single PID (first match)
pgrep nginx                     # Like pidof but more flexible
pgrep -u eknatha                # All PIDs for user
pgrep -l nginx                  # Show name with PID
pgrep -a python                 # Show full command
```

---

## 3. Real-time Monitoring

### top
```bash
top                             # Launch top
top -u eknatha                  # Filter by user
top -p 1234,5678                # Specific PIDs
top -b -n 3 -d 5 > report.txt  # Batch mode (3 iterations, 5s delay)
```

**top Interactive Keys:**
| Key | Action |
|-----|--------|
| `P` | Sort by CPU |
| `M` | Sort by Memory |
| `T` | Sort by Time |
| `k` | Kill a process (enter PID) |
| `r` | Renice a process |
| `1` | Toggle per-CPU stats |
| `H` | Toggle threads |
| `f` | Add/remove columns |
| `q` | Quit |
| `u` | Filter by user |

**top Header Explained:**
```
top - 10:30:01 up 5 days, 3:12,  2 users,  load average: 0.45, 0.52, 0.48
│              │                             │─── 1min  5min  15min load avg
│              └──── System uptime
Tasks: 312 total,   1 running, 311 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.3 us,  0.8 sy,  0.0 ni, 96.5 id,  0.3 wa,  0.0 hi,  0.1 si
          │          │              │            │── wa: I/O wait — if high: disk bottleneck
          │          │              └──────────── id: idle
          │          └─────────────────────────── sy: kernel/system calls
          └────────────────────────────────────── us: user space
MiB Mem:  15887.4 total,   2341.0 free,   8234.5 used,   5311.9 buff/cache
MiB Swap:  2048.0 total,   2047.9 free,      0.1 used.   6952.6 avail Mem
```

### htop (Install First)
```bash
sudo dnf install htop -y
htop                            # More visual than top
# Mouse-clickable, easy process kill with F9
# F2 for setup, F6 to sort, F5 for tree view
```

---

## 4. Signals & kill

### Common Signals
| Signal | Number | Description | Catchable? |
|--------|--------|-------------|------------|
| SIGHUP | 1 | Hangup / reload config | Yes |
| SIGINT | 2 | Interrupt (Ctrl+C) | Yes |
| SIGQUIT | 3 | Quit with core dump | Yes |
| SIGKILL | 9 | Kill immediately | **No** |
| SIGTERM | 15 | Graceful termination | Yes |
| SIGSTOP | 19 | Pause process | **No** |
| SIGCONT | 18 | Continue paused process | Yes |
| SIGUSR1/2 | 10/12 | User-defined | Yes |

```bash
# List all signals
kill -l

# Send signals by PID
kill 1234                       # Default: SIGTERM (15)
kill -9 1234                    # SIGKILL (force)
kill -HUP 1234                  # Reload (common for daemons)
kill -STOP 1234                 # Pause
kill -CONT 1234                 # Resume

# Kill by name
pkill nginx                     # SIGTERM to all nginx processes
pkill -9 zombie_process
pkill -u eknatha                # Kill all processes of user
pkill -HUP rsyslog              # Reload rsyslog

# killall — by exact name
killall nginx
killall -9 stuck-process

# Confirm before killing
pkill -e nginx                  # Echo process name when killing (-e = echo)
```

### Why SIGKILL is a Last Resort
```bash
# SIGTERM gives process time to:
# - Flush buffers / save state
# - Close file descriptors
# - Remove PID files / socket files
# - Notify other processes

# SIGKILL (9):
# - Immediate termination by kernel
# - No cleanup — can corrupt databases, leave locks
# - Always try SIGTERM first, wait 5-10s, then SIGKILL
```

---

## 5. Background Jobs

```bash
# Send to background
long-command &                  # Run immediately in background
Ctrl+Z                          # Suspend current foreground process

# Job control
jobs                            # List jobs in current shell
jobs -l                         # Include PIDs
fg                              # Bring last background job to foreground
fg %2                           # Bring job 2 to foreground
bg                              # Resume suspended job in background
bg %2                           # Resume job 2 in background

# Prevent job from dying when shell exits
nohup long-command &            # Output → nohup.out
nohup long-command > /var/log/myapp.log 2>&1 &

# Disown — release job from shell
long-command &
disown %1                       # Job 1 won't be killed when shell closes
```

### screen / tmux (Persistent Sessions)
```bash
# tmux (recommended)
sudo dnf install tmux -y
tmux new -s mysession           # New named session
tmux attach -t mysession        # Reattach
tmux ls                         # List sessions
# Ctrl+B d to detach

# screen
sudo dnf install screen -y
screen -S mysession
screen -r mysession             # Reattach
```

---

## 6. Priority — nice & renice

### Understanding Priority
```
PR (Priority): 0 to 139 (lower = higher priority on CPU)
NI (Nice):    -20 to 19 (nice value; lower = more CPU aggressive)
PR = 20 + NI   (for user processes)
```

```bash
# Start command with specific nice value
nice -n 10 backup-script.sh     # Less aggressive (+10 nice)
nice -n -5 important-process    # More aggressive (requires root for negative)

# Change running process priority
renice -n 10 -p 1234            # By PID
renice -n 5 -u eknatha          # All processes of user
renice -n -5 -p 1234            # Increase priority (requires root)

# View priority in ps
ps -eo pid,ni,pri,comm --sort=-ni | head -20
```

### When to Use nice
```bash
# CPU-intensive batch jobs (backup, compilation, rsync)
nice -n 15 tar -czf backup.tar.gz /data/

# Low-priority monitoring scripts
nice -n 19 /opt/scripts/health-check.sh

# High-priority real-time tasks (use with caution)
sudo nice -n -10 /opt/trading/engine
```

---

## 7. Resource Limits — ulimit

### Viewing Limits
```bash
ulimit -a                       # All limits for current shell
ulimit -n                       # Open file descriptors (nofile)
ulimit -u                       # Max user processes
ulimit -m                       # Max memory (kB)
ulimit -s                       # Stack size
```

### Setting Limits
```bash
ulimit -n 65535                 # Increase file descriptors (soft limit)
ulimit -Sn 65535                # Soft limit
ulimit -Hn 100000               # Hard limit (requires root to increase)
```

### Persistent Limits — /etc/security/limits.conf
```bash
# /etc/security/limits.conf format:
# <domain>  <type>  <item>  <value>
# domain: username, @group, * (all), or root

cat >> /etc/security/limits.conf << 'EOF'
# Platform Engineering user limits
eknatha   soft   nofile   65535
eknatha   hard   nofile   100000
eknatha   soft   nproc    32768
eknatha   hard   nproc    65536
*         soft   core     0          # Disable core dumps for all users
@docker   soft   nofile   1048576
EOF
```

### Drop-in Files (Preferred on RHEL 8+)
```bash
# Create /etc/security/limits.d/99-app-limits.conf
cat > /etc/security/limits.d/99-app-limits.conf << 'EOF'
nodeapp soft nofile 1000000
nodeapp hard nofile 1000000
nodeapp soft nproc  65535
nodeapp hard nproc  65535
EOF
```

### Verify Effective Limits for a Running Process
```bash
cat /proc/1234/limits            # Limits of PID 1234
```

---

## 8. /proc Filesystem

`/proc` is a **virtual filesystem** exposing kernel and process information in real-time.

```bash
# Per-process info (replace 1234 with actual PID)
ls /proc/1234/
cat /proc/1234/cmdline | tr '\0' ' '   # Full command line
cat /proc/1234/status                   # Process status details
cat /proc/1234/limits                   # Resource limits
ls -la /proc/1234/fd/                   # Open file descriptors
cat /proc/1234/net/tcp                  # Network connections

# System-wide info
cat /proc/cpuinfo                       # CPU details
cat /proc/meminfo                       # Memory info
cat /proc/loadavg                       # Load averages (1m 5m 15m running/total lastpid)
cat /proc/sys/kernel/pid_max            # Max PID value
cat /proc/sys/fs/file-max               # System-wide file descriptor limit
cat /proc/uptime                        # Uptime in seconds

# Useful kernel tunables via /proc/sys (non-persistent — use sysctl for persistent)
echo 1 > /proc/sys/net/ipv4/ip_forward  # Enable IP forwarding
```

### sysctl — Persistent Kernel Parameters
```bash
sysctl -a                               # All kernel parameters
sysctl vm.swappiness                    # View specific
sysctl -w vm.swappiness=10             # Set temporarily

# Persist in /etc/sysctl.d/
cat > /etc/sysctl.d/99-platform.conf << 'EOF'
# Network performance
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_forward = 1

# VM settings
vm.swappiness = 10
vm.dirty_ratio = 15

# File limits
fs.file-max = 2097152
EOF

sysctl --system                         # Apply all /etc/sysctl.d/*.conf files
```

---

## 9. OOM Killer

When memory is exhausted, the kernel's **Out-of-Memory (OOM) killer** terminates processes to free memory.

```bash
# Check if OOM killer fired
dmesg | grep -i "oom"
journalctl -k | grep -i "oom\|killed"

# View OOM score (0-1000, higher = more likely to be killed)
cat /proc/1234/oom_score
cat /proc/1234/oom_score_adj            # Adjustment (-1000 to 1000)

# Protect critical processes from OOM killer
echo -1000 > /proc/$(pidof critical-daemon)/oom_score_adj

# Make a process MORE likely to be killed (sacrifice it to protect others)
echo 900 > /proc/$(pidof unimportant-job)/oom_score_adj

# Persistent via systemd unit
# In /etc/systemd/system/myapp.service:
# OOMScoreAdjust=-500   (protect)
# OOMScoreAdjust=500    (sacrifice)
```

---

## 10. cgroups

**Control Groups (cgroups)** limit and monitor resource usage per process group.

On RHEL 8, systemd uses **cgroups v1** by default. RHEL 9 uses **cgroups v2**.

```bash
# View cgroup hierarchy
systemctl status nginx              # Shows cgroup info
systemd-cgls                        # Full cgroup tree
systemd-cgtop                       # Real-time cgroup resource usage

# Resource limits via systemd (easiest on RHEL)
# In /etc/systemd/system/myapp.service or override:
systemctl edit nginx
```

```ini
[Service]
# CPU limit
CPUQuota=50%              # Max 50% of one CPU
CPUWeight=200             # Relative weight (default 100)

# Memory limit
MemoryMax=512M            # Hard memory cap
MemoryHigh=400M           # Soft limit (starts throttling)
MemorySwapMax=0           # Disable swap for this service

# I/O limit
IOWeight=50               # Relative I/O weight
IOReadBandwidthMax=/dev/sda 100M
IOWriteBandwidthMax=/dev/sda 50M
```

```bash
# Apply changes
systemctl daemon-reload
systemctl restart nginx

# Verify
systemctl show nginx | grep -E "Memory|CPU|IO"
cat /sys/fs/cgroup/system.slice/nginx.service/memory.current
```

---

## 11. Do's and Don'ts

### ✅ Do's

**Do SIGTERM before SIGKILL:**
```bash
kill 1234           # Try graceful first
sleep 10
# If still running:
kill -9 1234
```

**Do monitor D-state processes:**
```bash
# D state = process stuck waiting for I/O — indicates disk/NFS issue
watch -n 1 'ps aux | grep " D "'
```

**Do use ulimits for long-running services:**
```bash
# Before running any high-concurrency app, verify:
ulimit -n          # Should be >= 65535
```

**Do check OOM killer logs after memory incidents:**
```bash
journalctl -k --since "1 hour ago" | grep -i oom
```

**Do use `pgrep` with patterns instead of `ps | grep`:**
```bash
pgrep -a nginx      # Cleaner than: ps aux | grep nginx | grep -v grep
```

### ❌ Don'ts

**Don't SIGKILL a database without exhausting other options:**
```bash
# mysql, postgres, redis — SIGKILL can corrupt data
# Always try: systemctl stop mysql (allows graceful shutdown)
```

**Don't leave zombies unaddressed:**
```bash
# Zombies indicate a bug in the parent process
# Find parent: ps -o ppid= -p <zombie_pid>
# Solution: fix parent process (or kill it to let init adopt)
```

**Don't set ulimits only in shell (they won't apply to services):**
```bash
# Shell ulimits don't affect systemd-launched services
# Use systemd LimitNOFILE= in unit files for services
```

**Don't use nice for I/O-intensive tasks without ionice:**
```bash
# nice only affects CPU priority
# For I/O, use ionice:
sudo ionice -c 3 -p 1234        # Idle class (only gets I/O when nothing else needs it)
ionice -c 2 -n 7 rsync -avz /src /dst   # Best-effort, lowest priority
```

---

## Quick Reference

```bash
# View processes
ps aux --sort=-%cpu | head -10
ps -ejH | head -30              # Tree view
pgrep -la nginx

# Kill processes
kill -15 1234                   # SIGTERM (graceful)
kill -9 1234                    # SIGKILL (force)
pkill -HUP nginx                # Reload all nginx

# Background jobs
command &                       # Run in background
Ctrl+Z                          # Suspend
jobs && fg                      # List and foreground
nohup command > /var/log/cmd.log 2>&1 &

# Priority
nice -n 15 backup.sh            # Low priority
renice -n -5 -p 1234            # Boost running process

# Limits
ulimit -n 65535                 # File descriptors
cat /proc/1234/limits           # Process limits

# OOM
dmesg | grep oom
echo -1000 > /proc/1234/oom_score_adj   # Protect from OOM
```

---

> **Prev:** [02 — Users, Groups & Permissions](./02-users-groups-permissions.md)  
> **Next:** [04 — systemd & Service Management](./04-systemd-services.md)  
> **Repository:** [github.com/eknatha/linux](https://github.com/eknatha/linux)
