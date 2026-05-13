# Linux Performance Tuning on RHEL

> **Applies to:** RHEL 8/9 · CentOS Stream 8/9 · Rocky Linux · AlmaLinux  
> **Level:** Advanced  
> **Author:** Eknatha Reddy P · [eknathalabs.com](https://eknathalabs.com) · [github.com/eknatha](https://github.com/eknatha)

---

## Table of Contents

1. [Performance Tuning Methodology](#1-performance-tuning-methodology)
2. [System Monitoring Tools](#2-system-monitoring-tools)
3. [CPU Performance](#3-cpu-performance)
4. [Memory Performance](#4-memory-performance)
5. [Disk I/O Performance](#5-disk-io-performance)
6. [Network Performance](#6-network-performance)
7. [tuned — Profile-Based Tuning](#7-tuned--profile-based-tuning)
8. [Kernel Parameters with sysctl](#8-kernel-parameters-with-sysctl)
9. [Profiling with perf and strace](#9-profiling-with-perf-and-strace)
10. [Real-World Scenarios](#10-real-world-scenarios)
11. [Do's and Don'ts](#11-dos-and-donts)
12. [Quick Reference Cheatsheet](#12-quick-reference-cheatsheet)

---

## 1. Performance Tuning Methodology

Never tune blindly. Follow the **USE Method**:

| Letter | Stands For | What to Measure |
|--------|-----------|-----------------|
| U | Utilization | How busy is the resource? (%) |
| S | Saturation | Is it overloaded / queuing? |
| E | Errors | Are there errors/failures? |

**Tuning Workflow:**

```
Observe → Baseline → Identify Bottleneck → Tune One Thing → Measure → Repeat
```

```bash
# Step 1: Get a baseline snapshot
uptime                         # Load average (1m, 5m, 15m)
free -h                        # Memory pressure
df -h                          # Disk space
iostat -x 1 5                  # Disk I/O utilization
ss -s                          # Socket summary
```

**Interpreting load average:**

```bash
# Load average > number of CPUs = system is saturated
nproc                          # Get CPU count
uptime
# Example: load 3.5 on a 4-core = 87.5% utilization (OK)
# Example: load 8.2 on a 4-core = overloaded
```

---

## 2. System Monitoring Tools

### sar — System Activity Reporter

`sar` collects, reports, and saves system activity data. Provided by the `sysstat` package.

```bash
# Install
dnf install sysstat -y
systemctl enable --now sysstat

# CPU usage every 2 seconds, 5 times
sar -u 2 5

# Memory usage
sar -r 2 5

# Disk I/O
sar -d 2 5

# Network throughput
sar -n DEV 2 5

# Historical data (yesterday's CPU report)
sar -u -f /var/log/sa/sa$(date -d yesterday +%d)

# All-in-one system report for today
sar -A
```

**Understanding sar CPU output:**

```
%user   - CPU used by user processes
%system - CPU used by kernel
%iowait - CPU waiting for I/O (high = disk bottleneck)
%steal  - CPU stolen by hypervisor (VMs — high = noisy neighbor)
%idle   - Free CPU
```

### vmstat — Virtual Memory Statistics

```bash
# General syntax: vmstat [delay] [count]
vmstat 2 10

# Output columns explained:
# r  = runnable processes (> nproc = saturated)
# b  = blocked on I/O
# si = swap in (KB/s)  — non-zero = memory pressure
# so = swap out (KB/s) — non-zero = memory pressure
# bi = blocks read from disk
# bo = blocks written to disk
# wa = % CPU waiting for I/O
# st = % CPU stolen (VMs)

# Wide output with timestamps
vmstat -t -w 2 5

# Disk statistics
vmstat -d 2 3

# Partition statistics
vmstat -p /dev/sda1 2 3
```

### iostat — I/O Statistics

```bash
# Install (part of sysstat)
dnf install sysstat -y

# Basic usage
iostat -x 2 5

# Key columns in extended output (-x):
# %util    - device busy % (near 100% = saturated)
# await    - avg I/O wait time in ms (high = slow disk or overloaded)
# r/s, w/s - reads and writes per second
# rkB/s    - read KB per second
# wkB/s    - write KB per second
# aqu-sz   - avg queue size (> 1 = requests piling up)
# svctm    - deprecated, ignore

# Per-device stats
iostat -xz -d sda 2 5

# JSON output (for scripts)
iostat -x -o JSON 2 3
```

### top and htop

```bash
# top interactive keys:
# 1    - Show per-CPU breakdown
# M    - Sort by memory
# P    - Sort by CPU (default)
# k    - Kill a process (enter PID)
# H    - Show threads
# u    - Filter by user
# q    - Quit

top

# htop — more user-friendly
dnf install htop -y
htop
# F5 = tree view, F6 = sort, F9 = kill, F2 = setup
```

### dstat — All-in-One (Replaced by dool on RHEL 9)

```bash
# RHEL 8
dnf install dstat -y
dstat -cdngym 2 10

# RHEL 9 (dstat replaced by dool)
dnf install dool -y
dool -cdngym 2 10

# Columns: cpu, disk, net, memory, system
```

### pidstat — Per-Process Stats

```bash
# Install (sysstat)
# CPU usage per process, every 2s
pidstat 2 5

# I/O per process
pidstat -d 2 5

# Memory per process
pidstat -r 2 5

# All stats for a specific PID
pidstat -u -d -r -p 1234 2 5

# Threads
pidstat -t -p 1234 2 5
```

---

## 3. CPU Performance

### Identifying CPU Bottlenecks

```bash
# Check CPU count and topology
lscpu
nproc
lscpu | grep -E "Socket|Core|Thread|NUMA"

# CPU frequency scaling (current vs max)
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq | head -5
cat /proc/cpuinfo | grep "cpu MHz"

# CPU governor (affects performance)
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# Options: performance, powersave, ondemand, schedutil

# Check for CPU throttling
dmesg | grep -i throttl
```

### Setting CPU Governor

```bash
# Set performance governor for all CPUs
cpupower frequency-set -g performance

# Install cpupower
dnf install kernel-tools -y

# Check current policy
cpupower frequency-info

# Persistent: use tuned (see section 7)
```

### CPU Affinity — Pinning Processes

```bash
# Run command on specific CPUs (cores 0 and 1)
taskset -c 0,1 ./my-app

# Set affinity on running process
taskset -cp 0,1 12345

# Check current affinity
taskset -cp 12345

# NUMA-aware execution
numactl --cpunodebind=0 --membind=0 ./my-app

# Check NUMA topology
numactl --hardware
```

### Interrupt (IRQ) Affinity

```bash
# View IRQ assignments
cat /proc/interrupts

# View IRQ affinity mask
cat /proc/irq/*/smp_affinity

# Assign IRQ 120 to CPU 2 (bitmask: 4 = binary 100)
echo 4 > /proc/irq/120/smp_affinity

# irqbalance daemon — auto-balances IRQs
systemctl status irqbalance
systemctl enable --now irqbalance
```

---

## 4. Memory Performance

### Memory Analysis

```bash
# Overview
free -h
cat /proc/meminfo

# Key fields in /proc/meminfo:
# MemTotal      - Total RAM
# MemFree       - Unused RAM
# MemAvailable  - Available for new processes (FREE + reclaimable cache)
# Buffers       - Kernel buffer cache
# Cached        - Page cache
# SwapTotal/SwapFree - Swap space
# HugePages_Total/Free - Huge pages
# Dirty         - Data waiting to be written to disk

# Show memory per NUMA node
numastat
numastat -m   # Detailed per-node

# Find memory hogs
ps aux --sort=-%mem | head -10

# smem — accurate RSS/PSS/USS memory per process
dnf install smem -y
smem -r | head -20
```

### Swap Management

```bash
# Check swap usage
swapon --show
cat /proc/swaps

# swappiness — how aggressively kernel uses swap
# 0 = avoid swap entirely  60 = default  100 = swap aggressively
sysctl vm.swappiness
sysctl -w vm.swappiness=10       # Temporary
echo "vm.swappiness=10" >> /etc/sysctl.d/99-perf.conf   # Persistent

# Create additional swap file if needed
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Add to /etc/fstab for persistence
echo "/swapfile none swap sw 0 0" >> /etc/fstab
```

### Huge Pages

Huge pages reduce TLB pressure for memory-intensive apps (databases, JVMs).

```bash
# Check current huge pages
cat /proc/meminfo | grep -i huge
grep -i hugepages /proc/meminfo

# Allocate huge pages (2MB each, allocate 512 = 1GB)
sysctl -w vm.nr_hugepages=512

# Persistent
echo "vm.nr_hugepages=512" >> /etc/sysctl.d/99-hugepages.conf

# Transparent Huge Pages (THP)
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never

# Disable THP (recommended for databases like MySQL, MongoDB)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Persistent via rc.local or tuned (see section 7)
```

### OOM Killer Tuning

```bash
# Check OOM score for a process (higher = more likely to be killed)
cat /proc/$(pgrep mysqld)/oom_score

# Adjust OOM score (-1000 = never kill, +1000 = kill first)
echo -500 > /proc/$(pgrep mysqld)/oom_score_adj

# View all processes OOM scores
for pid in /proc/[0-9]*/oom_score; do
  printf "%s\t%s\n" "$(cat $pid 2>/dev/null)" "${pid%/oom_score}"
done | sort -rn | head -10

# Protect critical process from OOM
echo -1000 > /proc/$(pgrep sshd)/oom_score_adj
```

---

## 5. Disk I/O Performance

### Identifying I/O Bottlenecks

```bash
# Quick I/O check
iostat -xz 2 5
# Look for: %util near 100%, await > 20ms, aqu-sz > 1

# Which processes are doing I/O?
iotop -o        # Only show active processes (-o flag)
# or
pidstat -d 2 5

# Disk latency test
ioping -c 20 /dev/sda

# Sequential read/write benchmarks
# Read throughput
dd if=/dev/sda of=/dev/null bs=1M count=1000 status=progress

# Write throughput (careful! overwrites data)
dd if=/dev/zero of=/tmp/testfile bs=1M count=500 status=progress

# fio — comprehensive I/O benchmarking
dnf install fio -y
fio --name=seq-read --filename=/tmp/testfile --rw=read --bs=1M --size=1G --direct=1
fio --name=rand-write --filename=/tmp/testfile --rw=randwrite --bs=4K --size=1G --direct=1 --iodepth=32
```

### I/O Schedulers

```bash
# Check current scheduler per device
cat /sys/block/sda/queue/scheduler

# Available schedulers
# none (noop) - for NVMe/SSDs (no reordering needed)
# mq-deadline - default for most block devices
# bfq         - fair queue (good for desktop/mixed workloads)
# kyber       - low-latency for fast storage

# Change scheduler temporarily
echo mq-deadline > /sys/block/sda/queue/scheduler
echo none > /sys/block/nvme0n1/queue/scheduler

# Persistent change via udev rule
cat > /etc/udev/rules.d/60-scheduler.rules <<'EOF'
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="mq-deadline"
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"
ACTION=="add|change", KERNEL=="nvme*", ATTR{queue/scheduler}="none"
EOF

udevadm trigger --type=devices --action=change
```

### Read-Ahead and Queue Depth

```bash
# Read-ahead — how much data to prefetch (in 512-byte sectors)
blockdev --getra /dev/sda             # Current value
blockdev --setra 4096 /dev/sda        # Set to 2MB (4096 * 512B)

# For sequential workloads: increase read-ahead
# For random access (databases): decrease or disable

# Queue depth — concurrent I/O requests
cat /sys/block/sda/queue/nr_requests  # Default: 64
echo 256 > /sys/block/sda/queue/nr_requests   # Increase for SSDs
```

### File System Optimization

```bash
# Mount options for performance
# noatime - don't update access time on reads (significant improvement)
# nodiratime - don't update dir access time
# barrier=0 - disable write barriers (ONLY if battery-backed cache)

# Example /etc/fstab entry
# /dev/sda1 /data xfs defaults,noatime,nodiratime 0 0

# Remount with noatime
mount -o remount,noatime /data

# XFS-specific tuning
xfs_info /data                      # View current settings
xfs_admin -l /dev/sda1              # View label
# XFS uses aggressive caching; tune via /sys/block or mount options

# Check file fragmentation (ext4)
e2fsck -fn /dev/sda1 2>&1 | grep "non-contiguous"
e4defrag /data                      # Defragment ext4 online

# XFS fragmentation check
xfs_db -r /dev/sda1 -c frag
xfs_fsr /data                       # Online defrag for XFS
```

---

## 6. Network Performance

### Network Analysis

```bash
# Interface stats
ip -s link show eth0
netstat -s          # Protocol statistics (deprecated but still useful)
ss -s               # Socket summary

# Real-time bandwidth
dnf install iftop -y
iftop -i eth0

# Per-connection network usage
nethogs eth0

# sar network stats
sar -n DEV 2 5      # Bytes in/out per interface
sar -n EDEV 2 5     # Network errors

# Dropped packets (critical!)
ip -s link show eth0 | grep -A2 "RX\|TX"
cat /proc/net/dev
```

### Network Tuning

```bash
# TCP buffer sizes (for high-throughput networks)
sysctl net.core.rmem_max
sysctl net.core.wmem_max
sysctl net.ipv4.tcp_rmem
sysctl net.ipv4.tcp_wmem

# Increase for high-bandwidth/high-latency (BDP tuning)
cat >> /etc/sysctl.d/99-network.conf <<'EOF'
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_congestion_control = bbr
EOF
sysctl -p /etc/sysctl.d/99-network.conf

# Enable BBR congestion control (RHEL 8+)
modprobe tcp_bbr
echo tcp_bbr >> /etc/modules-load.d/bbr.conf

# Check congestion control
sysctl net.ipv4.tcp_congestion_control
sysctl net.ipv4.tcp_available_congestion_control

# NIC interrupt coalescing (reduce CPU overhead)
ethtool -c eth0                      # Show current settings
ethtool -C eth0 rx-usecs 100         # Coalesce RX interrupts every 100µs

# NIC ring buffer size
ethtool -g eth0                      # Show current/max ring sizes
ethtool -G eth0 rx 4096 tx 4096      # Increase ring buffers

# Multi-queue NIC tuning
ls /sys/class/net/eth0/queues/
ethtool -l eth0                      # Combined queues
ethtool -L eth0 combined 8           # Set to 8 queues (match CPU count)
```

---

## 7. tuned — Profile-Based Tuning

`tuned` applies comprehensive system tuning profiles automatically.

```bash
# Install and enable
dnf install tuned -y
systemctl enable --now tuned

# List available profiles
tuned-adm list

# Current active profile
tuned-adm active

# Recommended profile for this system
tuned-adm recommend
```

### Built-in Profiles

| Profile | Use Case |
|---------|----------|
| `throughput-performance` | Max throughput, no power saving (recommended for servers) |
| `latency-performance` | Low-latency apps (trading, real-time) |
| `network-latency` | Network-intensive, low-latency |
| `network-throughput` | Maximum network throughput |
| `virtual-guest` | RHEL running as a VM guest |
| `virtual-host` | RHEL as a KVM hypervisor host |
| `oracle` | Oracle database servers |
| `balanced` | Balance performance and power (default) |
| `powersave` | Maximize power saving |

```bash
# Apply a profile
tuned-adm profile throughput-performance

# Apply multiple profiles (merged)
tuned-adm profile virtual-guest throughput-performance

# Disable tuning
tuned-adm off

# Verify active settings
tuned-adm active
tuned-adm verify
```

### Custom tuned Profile

```bash
# Create custom profile
mkdir -p /etc/tuned/my-db-server

cat > /etc/tuned/my-db-server/tuned.conf <<'EOF'
[main]
summary=Custom DB Server Tuning
include=throughput-performance

[cpu]
governor=performance
energy_perf_bias=performance

[vm]
transparent_hugepages=never

[disk]
readahead=>4096

[sysctl]
vm.swappiness=10
vm.dirty_ratio=15
vm.dirty_background_ratio=5
net.core.somaxconn=65535
EOF

# Apply custom profile
tuned-adm profile my-db-server

# Verify
tuned-adm active
```

---

## 8. Kernel Parameters with sysctl

### How sysctl Works

```bash
# View all parameters
sysctl -a

# View a specific parameter
sysctl vm.swappiness
sysctl net.ipv4.ip_forward

# Set temporarily (lost on reboot)
sysctl -w vm.swappiness=10

# Set persistently (survives reboot)
echo "vm.swappiness=10" > /etc/sysctl.d/99-mytuning.conf
sysctl -p /etc/sysctl.d/99-mytuning.conf   # Reload

# Load all sysctl configs
sysctl --system
```

### Essential Performance Parameters

```bash
cat > /etc/sysctl.d/99-performance.conf <<'EOF'
# ===== MEMORY =====
# How aggressively to use swap (0=avoid, 60=default, 10=prefer RAM)
vm.swappiness = 10

# Dirty page ratios
vm.dirty_ratio = 15             # Max % of RAM with dirty pages before blocking writes
vm.dirty_background_ratio = 5   # % at which background writeback starts

# Overcommit — allow more virtual memory than physical (for Java apps)
vm.overcommit_memory = 1        # Always allow (risky; use with OOM tuning)
# vm.overcommit_memory = 0      # Default: heuristic

# ===== FILESYSTEM =====
# inotify watches (for apps watching many files)
fs.inotify.max_user_watches = 524288
fs.file-max = 2097152           # Max open file descriptors system-wide

# ===== NETWORK =====
# Connection backlog
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# TIME_WAIT sockets
net.ipv4.tcp_tw_reuse = 1       # Reuse TIME_WAIT sockets (safe)
net.ipv4.tcp_fin_timeout = 15   # Reduce FIN_WAIT timeout

# Keep-alive
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15

# IP forwarding (for routers/containers)
net.ipv4.ip_forward = 1

# ===== KERNEL =====
# Panic on OOM (set to 0 to disable auto-reboot)
kernel.panic = 10
kernel.panic_on_oops = 1
EOF

sysctl -p /etc/sysctl.d/99-performance.conf
```

### Limits — /etc/security/limits.conf

```bash
# View current limits for a process
cat /proc/$(pgrep mysqld)/limits

# Global limits file
cat /etc/security/limits.conf

# Add limits for a specific user/service
cat >> /etc/security/limits.conf <<'EOF'
# MySQL user
mysql soft nofile 65536
mysql hard nofile 65536
mysql soft nproc  65536
mysql hard nproc  65536

# All users
* soft core unlimited
EOF

# Limits for systemd services (more reliable)
# Create /etc/systemd/system/mysqld.service.d/limits.conf
mkdir -p /etc/systemd/system/mysqld.service.d/
cat > /etc/systemd/system/mysqld.service.d/limits.conf <<'EOF'
[Service]
LimitNOFILE=65536
LimitNPROC=65536
EOF

systemctl daemon-reload
systemctl restart mysqld
```

---

## 9. Profiling with perf and strace

### perf — CPU & Hardware Profiling

```bash
# Install
dnf install perf -y

# CPU profiling — sample CPU call stacks for 30 seconds
perf record -g -p <PID> -- sleep 30
perf report

# System-wide CPU profiling
perf record -ag -- sleep 10
perf report

# Top functions consuming CPU (like top, but for functions)
perf top

# Count hardware events
perf stat -e cycles,instructions,cache-misses,branch-misses ls /

# Flame graphs with perf
perf record -F 99 -g -p <PID> -- sleep 30
perf script | stackcollapse-perf.pl | flamegraph.pl > flames.svg

# perf stat for a command
perf stat -d dd if=/dev/zero of=/dev/null bs=1M count=1000
```

### strace — System Call Tracer

```bash
# Trace system calls of a running process
strace -p <PID>

# Trace with timestamps
strace -t -p <PID>

# Trace specific syscalls only (e.g., only file operations)
strace -e trace=open,read,write,close -p <PID>

# Trace and show time spent in each syscall
strace -T -p <PID>

# Trace a new command
strace ls /var/log/

# Count syscalls and time (summary)
strace -c ls /var/log/
# Output: % time, seconds, usecs/call, calls, syscall name

# Trace child processes too (-f)
strace -f -p <PID>

# Write output to file
strace -o /tmp/trace.log -p <PID>
```

**strace for diagnosing slowness:**

```bash
# Find what files a slow app is opening
strace -e trace=open,openat -p <PID> 2>&1 | grep -v "ENOENT"

# Find network connections
strace -e trace=connect,accept,send,recv -p <PID>

# Find slow syscalls (> 1ms)
strace -T -p <PID> 2>&1 | awk -F'<' '{print $2, $0}' | sort -rn | head -20
```

### ltrace — Library Call Tracer

```bash
# Trace library calls (user-space libraries, not kernel)
dnf install ltrace -y
ltrace -p <PID>
ltrace ls /tmp/
```

---

## 10. Real-World Scenarios

### Scenario 1: High CPU iowait

**Symptom:** `top` shows high `%wa` (iowait), system feels slow.

```bash
# 1. Confirm iowait
sar -u 1 10
# Look for %iowait > 20%

# 2. Find which disk
iostat -xz 1 5
# Find device with %util near 100% or high await

# 3. Find which process
iotop -o -d 1

# 4. Find which files
lsof -p <PID>
fatrace   # Real-time file access tracer (dnf install fatrace)

# 5. Solutions:
# - Switch to faster storage (SSD/NVMe)
# - Tune I/O scheduler (none for NVMe)
# - Increase I/O queue depth
# - Application-level: add caching, async I/O
```

### Scenario 2: Memory Pressure / OOM Kills

**Symptom:** Processes killed randomly, system slow, swap usage climbing.

```bash
# 1. Check OOM events
dmesg | grep -i "oom\|killed"
journalctl -k | grep -i "oom\|out of memory"

# 2. Check memory consumers
ps aux --sort=-%mem | head -15
smem -r | head -15

# 3. Check swap usage trend
sar -r 1 60 | tail -20
vmstat 1 10 | awk '{print $7, $8}'  # si=swap-in, so=swap-out

# 4. Solutions:
# - Add swap space (temporary relief)
# - Tune vm.swappiness lower
# - Protect critical processes with oom_score_adj
# - Fix memory leak in application
# - Add RAM
```

### Scenario 3: High Network Latency / Packet Loss

**Symptom:** Application timeouts, slow database queries, high latency.

```bash
# 1. Basic connectivity test with latency
ping -c 100 <target> | tail -3
mtr --report <target>

# 2. Check packet drops at NIC level
ip -s link show eth0
netstat -s | grep -i "drop\|error\|retransmit"

# 3. Check socket buffer exhaustion
ss -s
sysctl net.core.rmem_max
sysctl net.core.wmem_max

# 4. Check retransmissions (indicates packet loss)
sar -n TCP 1 10
# Look for: retrans/s

# 5. Check if NIC queue is full
ip -s link show eth0 | grep -A2 RX

# 6. Solutions:
# - Increase NIC ring buffers: ethtool -G eth0 rx 4096
# - Tune TCP buffers via sysctl
# - Check for duplex mismatch: ethtool eth0
# - Enable jumbo frames: ip link set eth0 mtu 9000
```

### Scenario 4: MySQL / PostgreSQL Database Slow

```bash
# 1. Check if it's I/O bound
iostat -xz 1 10
pidstat -d 1 10 -p $(pgrep mysqld)

# 2. Check if it's memory bound (buffer pool pressure)
# MySQL: SHOW ENGINE INNODB STATUS;
free -h  # Is there available RAM?

# 3. Check if CPU is the bottleneck
pidstat -u 1 10 -p $(pgrep mysqld)

# 4. System-level tuning for databases
# Disable THP
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Increase file descriptors
echo "mysql hard nofile 65536" >> /etc/security/limits.conf

# Apply tuned profile
tuned-adm profile throughput-performance

# Reduce swappiness
sysctl -w vm.swappiness=1

# 5. I/O scheduler: none for NVMe, mq-deadline for SAS
echo mq-deadline > /sys/block/sda/queue/scheduler
```

---

## 11. Do's and Don'ts

### ✅ Do's

**Baseline First**
```bash
# Always capture a baseline before tuning
sar -A > /root/baseline-$(date +%Y%m%d).txt
uptime >> /root/baseline-$(date +%Y%m%d).txt
free -h >> /root/baseline-$(date +%Y%m%d).txt
```

**Test One Change at a Time**
```bash
# Change ONE parameter, measure, then change the next
# Document every change with date and reason
echo "2024-01-15: Set vm.swappiness=10 - baseline showed high swap usage" >> /root/tuning.log
```

**Use sysctl.d for Persistent Changes**
```bash
# Good: named file in sysctl.d
echo "vm.swappiness=10" > /etc/sysctl.d/99-swappiness.conf

# Verify it loaded
sysctl vm.swappiness
```

**Use tuned for Profile-Based Tuning**
```bash
# Good: use tuned profiles for comprehensive, consistent tuning
tuned-adm profile throughput-performance
```

**Protect Critical Services from OOM**
```bash
# Always protect sshd so you don't lose access
echo -1000 > /proc/$(pgrep sshd | head -1)/oom_score_adj
```

**Test Under Load**
```bash
# Use stress-ng to simulate load before going to production
dnf install stress-ng -y
stress-ng --cpu 4 --io 2 --vm 1 --vm-bytes 1G --timeout 60s
```

---

### ❌ Don'ts

**Don't Tune Without Measuring**
```bash
# ❌ Wrong: applying random "best practices" without data
echo "vm.dirty_ratio=80" >> /etc/sysctl.conf   # Why? Based on what data?

# ✅ Right: measure dirty pages first, then tune
cat /proc/meminfo | grep -i dirty
sar -r 1 60 | grep -i dirty
```

**Don't Set vm.overcommit_memory=1 Without OOM Protection**
```bash
# ❌ Dangerous: allows unlimited overcommit, can cause catastrophic OOM
sysctl -w vm.overcommit_memory=1    # Without OOM protection = disaster

# ✅ If you need overcommit, pair it with OOM score tuning
sysctl -w vm.overcommit_memory=1
echo -1000 > /proc/$(pgrep mysql)/oom_score_adj
```

**Don't Disable Swap Entirely on Non-SSD Systems**
```bash
# ❌ Wrong: disabling swap can cause immediate OOM kills
swapoff -a    # On a server without abundant RAM

# ✅ Right: keep swap, but reduce swappiness to minimize use
sysctl -w vm.swappiness=10
```

**Don't Use /etc/sysctl.conf Directly**
```bash
# ❌ Messy: editing the main sysctl.conf mixes OS defaults with custom tuning
echo "vm.swappiness=10" >> /etc/sysctl.conf

# ✅ Clean: use drop-in files in sysctl.d
echo "vm.swappiness=10" > /etc/sysctl.d/99-tuning.conf
```

**Don't Disable Barriers Without Battery-Backed Cache**
```bash
# ❌ Data corruption risk: disabling write barriers without BBWC
mount -o remount,barrier=0 /data    # Can corrupt FS on power failure

# ✅ Only disable barriers with confirmed battery-backed write cache (RAID controllers)
```

**Don't Set I/O Scheduler to mq-deadline for NVMe**
```bash
# ❌ Suboptimal: NVMe handles its own queuing
echo mq-deadline > /sys/block/nvme0n1/queue/scheduler

# ✅ Correct: use none (noop) for NVMe
echo none > /sys/block/nvme0n1/queue/scheduler
```

**Don't Ignore %steal in VMs**
```bash
# ❌ Tuning the guest OS when the problem is the hypervisor
# High %steal means the hypervisor is starving your VM — no guest-level fix

# ✅ Escalate to infra team; move to less-loaded host or resize VM
sar -u 1 10 | grep -i steal
```

---

## 12. Quick Reference Cheatsheet

```bash
# ===== MONITORING =====
sar -u 2 5                          # CPU stats
sar -r 2 5                          # Memory stats
sar -d 2 5                          # Disk stats
sar -n DEV 2 5                      # Network stats
vmstat 2 5                          # VM stats (runqueue, swap, I/O)
iostat -xz 2 5                      # Disk I/O extended
iotop -o                            # Real-time per-process I/O
pidstat -u -d -r 2 5                # Per-process CPU/IO/mem
ss -s                               # Socket summary
ip -s link show eth0                # NIC stats

# ===== CPU =====
lscpu                               # CPU topology
cpupower frequency-info             # CPU governor/freq
cpupower frequency-set -g performance   # Set performance governor
taskset -c 0,1 <cmd>                # Pin command to cores 0,1
numactl --hardware                  # NUMA topology
numactl --cpunodebind=0 <cmd>       # NUMA-aware execution

# ===== MEMORY =====
free -h                             # Memory overview
sysctl vm.swappiness                # Check swappiness
sysctl -w vm.swappiness=10          # Set swappiness
cat /proc/meminfo | grep -i huge    # Huge pages
echo never > /sys/kernel/mm/transparent_hugepage/enabled  # Disable THP

# ===== DISK =====
cat /sys/block/sda/queue/scheduler  # Current I/O scheduler
echo mq-deadline > /sys/block/sda/queue/scheduler   # Set scheduler
echo none > /sys/block/nvme0n1/queue/scheduler       # NVMe: set none
blockdev --getra /dev/sda           # Read-ahead sectors
blockdev --setra 4096 /dev/sda      # Set read-ahead

# ===== NETWORK =====
ethtool eth0                        # NIC info (duplex, speed)
ethtool -g eth0                     # Ring buffer sizes
ethtool -G eth0 rx 4096 tx 4096     # Increase ring buffers
ethtool -l eth0                     # Combined queues
ethtool -L eth0 combined 8          # Set 8 queues
sysctl net.ipv4.tcp_congestion_control  # Check congestion algo

# ===== TUNED =====
tuned-adm recommend                 # Recommended profile
tuned-adm profile throughput-performance   # Apply profile
tuned-adm active                    # Active profile
tuned-adm verify                    # Verify settings applied

# ===== SYSCTL =====
sysctl -a | grep vm                 # All vm.* params
sysctl -w vm.swappiness=10          # Temp change
echo "vm.swappiness=10" > /etc/sysctl.d/99-tuning.conf   # Persistent
sysctl -p /etc/sysctl.d/99-tuning.conf   # Reload
sysctl --system                     # Load all configs

# ===== PROFILING =====
perf top                            # Live CPU function profiling
perf record -g -p <PID> -- sleep 30 && perf report   # Profile a process
perf stat -d <cmd>                  # Count hw events for command
strace -c -p <PID>                  # Syscall summary for process
strace -T -e trace=read,write -p <PID>   # Time specific syscalls

# ===== LIMITS =====
cat /proc/<PID>/limits              # Current limits for process
ulimit -a                           # Current shell limits
ulimit -n 65536                     # Set open files for session
```

---

*← Previous: [Shell Scripting](11-shell-scripting.md)*

---

*Eknatha Reddy P · [eknathalabs.com](https://eknathalabs.com) · [github.com/eknatha](https://github.com/eknatha)*
