# 07 — Networking on RHEL

> **Author:** Eknatha Reddy P · [eknathalabs.com](https://eknathalabs.com) · [github.com/eknatha](https://github.com/eknatha)  
> **Applies to:** RHEL 8/9, CentOS Stream, Rocky Linux, AlmaLinux  
> **Level:** Intermediate

---

## Table of Contents
1. [Network Stack Overview](#1-network-stack-overview)
2. [NetworkManager — nmcli & nmtui](#2-networkmanager)
3. [ip Commands](#3-ip-commands)
4. [DNS Configuration](#4-dns-configuration)
5. [Routing](#5-routing)
6. [Network Bonding & Teaming](#6-bonding--teaming)
7. [VLANs](#7-vlans)
8. [Troubleshooting Toolkit](#8-troubleshooting-toolkit)
9. [Do's and Don'ts](#9-dos-and-donts)

---

## 1. Network Stack Overview

On RHEL 8/9, **NetworkManager** is the preferred method for all network configuration. Direct edits to ifcfg files or ip commands are not persistent across reboots.

```
Application Layer
      ↕
 NetworkManager  ← Central configuration manager
      ↕
  Kernel Networking (ip, ss, netstat)
      ↕
   Network Interface (ens3, eth0, bond0)
      ↕
   Physical / Virtual NIC
```

### Interface Naming
RHEL 8+ uses **predictable network interface names** (no more eth0/eth1):
```
ens3    — Ethernet, slot 3 (embedded/onboard)
enp2s0  — Ethernet, PCI bus 2, slot 0
eno1    — Ethernet, onboard, index 1
wlp3s0  — Wireless, PCI bus 3, slot 0
bond0   — Bonding interface
team0   — Teaming interface
```

---

## 2. NetworkManager

### nmcli — Command Line Interface

```bash
# Status overview
nmcli general status
nmcli connection show               # All connections
nmcli connection show --active      # Active connections
nmcli device status                 # All devices
nmcli device show ens3              # Detailed device info

# Create static IP connection
nmcli connection add \
  type ethernet \
  con-name "ens3-static" \
  ifname ens3 \
  ipv4.method manual \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8 8.8.4.4" \
  ipv4.dns-search "example.com" \
  autoconnect yes

# Activate the connection
nmcli connection up ens3-static

# Modify existing connection
nmcli connection modify ens3-static \
  ipv4.addresses "192.168.1.101/24" \
  ipv4.dns "1.1.1.1 8.8.8.8"
nmcli connection up ens3-static     # Apply changes

# DHCP connection
nmcli connection add \
  type ethernet \
  con-name "ens4-dhcp" \
  ifname ens4 \
  ipv4.method auto

# Add secondary IP (multi-homing)
nmcli connection modify ens3-static \
  +ipv4.addresses "10.0.0.50/24"

# Delete connection
nmcli connection delete ens3-static

# Bring up/down
nmcli connection up ens3-static
nmcli connection down ens3-static
nmcli device disconnect ens3
nmcli device connect ens3

# Reload NetworkManager configs
nmcli connection reload
```

### Key nmcli Properties
```bash
# View all properties of a connection
nmcli connection show ens3-static

# Common properties:
# ipv4.method: auto (DHCP) | manual (static) | disabled
# ipv4.addresses: IP/prefix (e.g. 192.168.1.100/24)
# ipv4.gateway: default gateway
# ipv4.dns: DNS servers (space-separated)
# ipv4.dns-search: search domains
# ipv4.routes: static routes (e.g. "10.0.0.0/8 192.168.1.254")
# connection.autoconnect: yes|no
# 802-3-ethernet.mtu: MTU (default 1500)
```

### nmtui — Text UI (Interactive)
```bash
nmtui                               # Interactive TUI — good for quick edits
nmtui edit ens3-static              # Edit specific connection
nmtui connect ens3-static           # Connect
```

### Connection Files Location
```bash
ls /etc/NetworkManager/system-connections/    # RHEL 8+ (keyfile format)
# Or legacy:
ls /etc/sysconfig/network-scripts/            # ifcfg-* format (deprecated in RHEL 9)
```

---

## 3. ip Commands

The `ip` command replaces the deprecated `ifconfig`, `route`, and `arp`.

### ip addr — IP Addresses
```bash
ip addr show                        # All interfaces
ip addr show ens3                   # Specific interface
ip a                                # Short form

# Add/remove IP (NOT persistent — use nmcli for persistence)
ip addr add 192.168.1.200/24 dev ens3
ip addr del 192.168.1.200/24 dev ens3

# Bring interface up/down
ip link set ens3 up
ip link set ens3 down
```

### ip link — Link Layer
```bash
ip link show                        # All interfaces
ip link show ens3
ip link set ens3 mtu 9000           # Set jumbo frames
ip link set ens3 promisc on         # Promiscuous mode
ip -s link show ens3                # With statistics (RX/TX errors)
```

### ip route — Routing Table
```bash
ip route show                       # Main routing table
ip route show table all             # All tables

# Add/delete routes (NOT persistent)
ip route add 10.0.0.0/8 via 192.168.1.254 dev ens3
ip route del 10.0.0.0/8
ip route add default via 192.168.1.1    # Default gateway

# Persistent routes via nmcli
nmcli connection modify ens3-static \
  +ipv4.routes "10.0.0.0/8 192.168.1.254"
nmcli connection up ens3-static
```

### ss — Socket Statistics (Replaces netstat)
```bash
ss -tlnp                            # TCP listening, numeric, with process
ss -ulnp                            # UDP listening
ss -tlnp | grep :80                 # Who's listening on port 80
ss -s                               # Summary statistics
ss -o state established             # Established connections
ss -tnp dst 10.0.0.100             # Connections to specific host
ss -tnp sport = :22                 # SSH connections
```

---

## 4. DNS Configuration

### Resolver Configuration
```bash
# Modern: /etc/resolv.conf is managed by NetworkManager + systemd-resolved
cat /etc/resolv.conf

# View DNS via nmcli
nmcli connection show ens3-static | grep dns

# Set DNS
nmcli connection modify ens3-static \
  ipv4.dns "1.1.1.1 8.8.8.8" \
  ipv4.dns-search "internal.example.com example.com"
nmcli connection up ens3-static

# systemd-resolved (RHEL 8+)
systemctl status systemd-resolved
resolvectl status               # Current DNS configuration
resolvectl query google.com    # DNS query via systemd-resolved
resolvectl flush-caches        # Flush DNS cache

# Manual /etc/resolv.conf (if not using systemd-resolved)
cat > /etc/resolv.conf << 'EOF'
nameserver 1.1.1.1
nameserver 8.8.8.8
search internal.example.com example.com
options ndots:5 timeout:2 attempts:3
EOF
```

### /etc/hosts — Local Override
```bash
# /etc/hosts takes precedence over DNS (by default in /etc/nsswitch.conf)
cat /etc/hosts
echo "10.0.0.50 db.internal db" >> /etc/hosts

# NSSwitch configuration
cat /etc/nsswitch.conf | grep hosts
# hosts: files dns    ← files first, then DNS
```

### DNS Testing
```bash
dig google.com                      # Full DNS lookup
dig @8.8.8.8 google.com            # Use specific DNS server
dig google.com MX                   # MX records
dig +short google.com              # IP only
nslookup google.com
host google.com
getent hosts google.com            # Uses nsswitch.conf (includes /etc/hosts)
```

---

## 5. Routing

```bash
# View routing table
ip route show
ip route get 10.0.0.50             # Which route would be used for this IP?

# Persistent static routes via nmcli
nmcli connection modify ens3-static \
  ipv4.routes "10.0.0.0/8 192.168.1.254, 172.16.0.0/12 192.168.1.1"
nmcli connection up ens3-static

# Enable IP forwarding (router function)
sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/99-ip-forward.conf
sysctl --system

# Policy routing (multiple routing tables)
ip route show table 100            # Custom routing table
ip rule show                       # Policy routing rules
```

---

## 6. Bonding & Teaming

### Network Bonding (Kernel Driver)

```bash
# Create a bond interface (LACP — 802.3ad)
nmcli connection add \
  type bond \
  con-name bond0 \
  ifname bond0 \
  bond.options "mode=802.3ad,miimon=100,lacp_rate=fast,xmit_hash_policy=layer3+4"

# Add slave interfaces to bond
nmcli connection add \
  type ethernet \
  con-name bond0-slave-ens3 \
  ifname ens3 \
  master bond0

nmcli connection add \
  type ethernet \
  con-name bond0-slave-ens4 \
  ifname ens4 \
  master bond0

# Configure IP on the bond
nmcli connection modify bond0 \
  ipv4.method manual \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1

# Activate
nmcli connection up bond0-slave-ens3
nmcli connection up bond0-slave-ens4
nmcli connection up bond0

# Monitor bond
cat /proc/net/bonding/bond0
```

### Bonding Modes
| Mode | Name | Description | Switch Needed |
|------|------|-------------|---------------|
| 0 | balance-rr | Round-robin | No |
| 1 | active-backup | One active, one standby | No |
| 2 | balance-xor | Hash-based | No |
| 3 | broadcast | All interfaces transmit | No |
| 4 | 802.3ad (LACP) | Dynamic link aggregation | Yes (LACP) |
| 5 | balance-tlb | Adaptive transmit | No |
| 6 | balance-alb | Adaptive load balancing | No |

---

## 7. VLANs

```bash
# Create VLAN 100 on ens3
nmcli connection add \
  type vlan \
  con-name vlan100 \
  ifname ens3.100 \
  dev ens3 \
  id 100 \
  ipv4.method manual \
  ipv4.addresses 10.100.0.10/24

nmcli connection up vlan100

# Verify
ip link show ens3.100
ip addr show ens3.100
```

---

## 8. Troubleshooting Toolkit

### Systematic Troubleshooting Approach

```bash
# Step 1: Is the interface up and has an IP?
ip addr show ens3
nmcli device status

# Step 2: Can we reach the gateway?
ip route show | grep default          # Find default gateway
ping -c 3 192.168.1.1                 # Ping gateway

# Step 3: Can we reach DNS?
ping -c 3 8.8.8.8                     # Ping by IP
dig google.com @8.8.8.8               # DNS test

# Step 4: Can we resolve names?
ping -c 3 google.com                  # Test DNS resolution

# Step 5: Is a specific port reachable?
nc -zv 10.0.0.50 3306                 # Test TCP port
telnet 10.0.0.50 22                   # Test SSH port
```

### Specific Tools

```bash
# traceroute / tracepath
traceroute 8.8.8.8
tracepath 8.8.8.8                     # No root needed, UDP-based

# mtr — continuous traceroute
sudo dnf install mtr -y
mtr 8.8.8.8                           # Interactive
mtr --report 8.8.8.8                  # Report mode

# tcpdump — packet capture
tcpdump -i ens3                        # Capture on interface
tcpdump -i ens3 port 80               # Filter by port
tcpdump -i ens3 host 10.0.0.50        # Filter by host
tcpdump -i ens3 -w /tmp/capture.pcap  # Save to file
tcpdump -r /tmp/capture.pcap          # Read from file

# nmap
sudo dnf install nmap -y
nmap -sV 192.168.1.1                  # Service version scan
nmap -p 22,80,443 192.168.1.0/24     # Scan subnet for ports
nmap -sn 192.168.1.0/24              # Ping sweep (no port scan)

# curl for HTTP testing
curl -v http://10.0.0.50
curl -I https://api.example.com       # Headers only
curl -o /dev/null -s -w "%{http_code}\n" http://app.local   # Just status code
curl --connect-timeout 5 http://app.local

# ss for connection analysis
ss -tlnp                              # Listening services
ss -s                                 # Summary
ss -o state TIME-WAIT | wc -l        # Count TIME_WAIT connections

# arp — neighbor table
arp -n                                # ARP table (deprecated)
ip neigh show                         # Modern equivalent
ip neigh flush dev ens3               # Flush ARP cache
```

### Common Network Issues

```bash
# "No route to host"
ip route show                         # Check routing table
ip route get <destination-ip>         # What route would be used?

# "Connection refused"
ss -tlnp | grep <port>               # Is service listening?
systemctl status <service>            # Is service running?

# DNS not resolving
resolvectl status                     # Check current DNS config
dig @127.0.0.53 google.com           # Test systemd-resolved
cat /etc/nsswitch.conf                # Check resolution order

# High network latency
ping -f -c 1000 <host>               # Flood ping (root needed) — check packet loss
mtr <host>                            # Find where latency occurs

# Interface showing errors
ip -s link show ens3                  # Check RX/TX error counters
ethtool ens3                          # Driver and NIC info
ethtool -S ens3                       # NIC statistics
```

---

## 9. Do's and Don'ts

### ✅ Do's

**Do use nmcli for all persistent configuration:**
```bash
# Temporary (ip commands) are lost on reboot
# Permanent: use nmcli
nmcli connection modify ens3-static ipv4.dns "1.1.1.1 8.8.8.8"
nmcli connection up ens3-static
```

**Do use `ss` instead of `netstat`:**
```bash
ss -tlnp           # netstat is deprecated and slower
```

**Do verify with `ip route get` before troubleshooting:**
```bash
ip route get 10.100.0.50    # Shows exact route + src IP that would be used
```

**Do name connections meaningfully:**
```bash
nmcli connection add con-name "prod-ens3-static" ...   # Better than default names
```

**Do test DNS resolution separately from network connectivity:**
```bash
ping 8.8.8.8        # Tests network reachability
ping google.com      # Tests DNS + network
```

### ❌ Don'ts

**Don't edit ifcfg files directly on RHEL 9:**
```bash
# ❌ Deprecated in RHEL 9, may not work
vi /etc/sysconfig/network-scripts/ifcfg-ens3

# ✅ Use nmcli or write keyfiles in /etc/NetworkManager/system-connections/
```

**Don't use `ifconfig` — it's deprecated:**
```bash
# ❌ Outdated, may not be installed
ifconfig ens3

# ✅ Modern equivalent
ip addr show ens3
```

**Don't disable NetworkManager without an alternative:**
```bash
# ❌ Leaves you with no persistent networking
systemctl disable --now NetworkManager
```

**Don't set DNS in /etc/resolv.conf manually on systems using NetworkManager:**
```bash
# NetworkManager overwrites /etc/resolv.conf
# Set DNS via: nmcli connection modify <con> ipv4.dns "..."
```

---

## Quick Reference

```bash
# Status
nmcli device status
nmcli connection show
ip addr show

# Static IP
nmcli con mod ens3-static \
  ipv4.method manual \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "1.1.1.1 8.8.8.8"
nmcli con up ens3-static

# Routes
ip route show
ip route get 10.0.0.1
nmcli con mod ens3-static +ipv4.routes "10.0.0.0/8 192.168.1.254"

# DNS
resolvectl status
dig @1.1.1.1 google.com

# Troubleshoot
ping 192.168.1.1        # Gateway
ping 8.8.8.8            # Internet
dig google.com           # DNS
ss -tlnp | grep :80     # Port in use
tcpdump -i ens3 -n port 443   # Packet capture
```

---

> **Prev:** [06 — LVM Deep Dive](./06-lvm-deep-dive.md)  
> **Next:** [08 — firewalld & iptables](./08-firewalld-iptables.md)  
> **Repository:** [github.com/eknatha/linux](https://github.com/eknatha/linux)
