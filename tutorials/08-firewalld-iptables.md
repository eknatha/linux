# 08 — firewalld & iptables (RHEL)

> **Author:** Eknatha Reddy P · [eknathalabs.com](https://eknathalabs.com) · [github.com/eknatha](https://github.com/eknatha)  
> **Applies to:** RHEL 8/9, CentOS Stream, Rocky Linux, AlmaLinux  
> **Level:** Intermediate

---

## Table of Contents
1. [Firewall Overview on RHEL](#1-firewall-overview)
2. [firewalld — Zones & Services](#2-firewalld)
3. [firewalld Rich Rules](#3-rich-rules)
4. [firewalld Masquerade & Port Forwarding](#4-masquerade--port-forwarding)
5. [iptables — Deep Control](#5-iptables)
6. [nftables — The Future](#6-nftables)
7. [Do's and Don'ts](#7-dos-and-donts)

---

## 1. Firewall Overview

On RHEL 8/9:
- **firewalld** = High-level frontend (recommended, default)
- **iptables** = Low-level kernel framework (firewalld uses nftables as backend on RHEL 9)
- **nftables** = Replacement for iptables (backend on RHEL 9)

```
Application (firewalld, iptables-legacy, nft)
         ↓
    nftables kernel subsystem
         ↓
   Netfilter hooks in kernel
         ↓
   Network packets
```

---

## 2. firewalld

### Zones — The Core Concept

Every interface and source IP is assigned a **zone** with a predefined trust level.

```bash
# View zones
firewall-cmd --get-zones
firewall-cmd --get-default-zone
firewall-cmd --get-active-zones          # Zones with assigned interfaces

# Change default zone
firewall-cmd --set-default-zone=public

# Assign interface to zone
firewall-cmd --zone=internal --add-interface=ens4 --permanent
firewall-cmd --reload
```

### Built-in Zones (Trust Level)
| Zone | Trust | Default Policy |
|------|-------|---------------|
| `drop` | Untrusted | All incoming dropped, no ICMP |
| `block` | Untrusted | All incoming rejected (ICMP refused) |
| `public` | Public networks | Only selected incoming allowed |
| `external` | External with masquerade | Selected incoming, masquerading |
| `dmz` | DMZ servers | Limited incoming |
| `work` | Work networks | More trusted |
| `home` | Home networks | More trusted, some discovery |
| `internal` | Internal networks | Trusted with services |
| `trusted` | Fully trusted | All incoming accepted |

### Services — Predefined Rules
```bash
# List all predefined services
firewall-cmd --get-services
ls /usr/lib/firewalld/services/        # Service XML definitions

# View details of a service
firewall-cmd --info-service=http

# Add/remove services (--permanent makes it survive reboots)
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --zone=public --add-service=https --permanent
firewall-cmd --zone=public --remove-service=http --permanent

# Apply permanent changes
firewall-cmd --reload

# Temporary (no --permanent, takes effect immediately, lost on reload)
firewall-cmd --zone=public --add-service=http     # Immediate, no reload needed
```

### Port Rules
```bash
# Open specific port
firewall-cmd --zone=public --add-port=8080/tcp --permanent
firewall-cmd --zone=public --add-port=4000-5000/tcp --permanent
firewall-cmd --zone=public --add-port=53/udp --permanent

# Remove port
firewall-cmd --zone=public --remove-port=8080/tcp --permanent

firewall-cmd --reload
```

### View Current Configuration
```bash
firewall-cmd --list-all                 # Current zone config
firewall-cmd --zone=public --list-all   # Specific zone
firewall-cmd --list-all-zones           # All zones
firewall-cmd --list-services            # Active services
firewall-cmd --list-ports               # Active ports
firewall-cmd --list-rich-rules          # Rich rules
```

### Custom Services
```bash
# Create a custom service definition
cat > /etc/firewalld/services/myapp.xml << 'EOF'
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>MyApp</short>
  <description>My Application API</description>
  <port protocol="tcp" port="3000"/>
  <port protocol="tcp" port="3001"/>
</service>
EOF

firewall-cmd --reload
firewall-cmd --zone=public --add-service=myapp --permanent
firewall-cmd --reload
```

---

## 3. Rich Rules

Rich rules provide fine-grained control beyond simple service/port rules.

### Syntax
```
rule
  [family="ipv4|ipv6"]
  [source address="IP/prefix" [invert="True"]]
  [destination address="IP/prefix"]
  [service|port|protocol ...]
  [log [prefix="..."] [level="..."] [limit value="N/s|m|h|d"]]
  [audit]
  [accept|reject|drop]
```

### Examples
```bash
# Allow SSH only from specific IP
firewall-cmd --zone=public \
  --add-rich-rule='rule family="ipv4" source address="10.0.0.0/8" service name="ssh" accept' \
  --permanent

# Block SSH from specific IP
firewall-cmd --zone=public \
  --add-rich-rule='rule family="ipv4" source address="192.168.50.100" service name="ssh" reject' \
  --permanent

# Allow HTTP from subnet with logging
firewall-cmd --zone=public \
  --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" service name="http" log prefix="HTTP-ALLOW " level="info" limit value="1/m" accept' \
  --permanent

# Rate limit: allow SMTP but limit to 10 connections per minute
firewall-cmd --zone=public \
  --add-rich-rule='rule service name="smtp" limit value="10/m" accept' \
  --permanent

# Allow port from specific source
firewall-cmd --zone=public \
  --add-rich-rule='rule family="ipv4" source address="172.16.0.0/12" port port="5432" protocol="tcp" accept' \
  --permanent

# Drop all from specific country/IP range
firewall-cmd --zone=public \
  --add-rich-rule='rule family="ipv4" source address="203.0.113.0/24" drop' \
  --permanent

# Apply and check
firewall-cmd --reload
firewall-cmd --list-rich-rules
```

---

## 4. Masquerade & Port Forwarding

### IP Masquerade (NAT — Source NAT)
Allows internal hosts to reach internet via this machine's IP.

```bash
# Enable masquerade on external zone
firewall-cmd --zone=external --add-masquerade --permanent

# Or on specific zone
firewall-cmd --zone=public --add-masquerade --permanent

# Enable IP forwarding (required for masquerade)
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-ip-forward.conf
sysctl --system

firewall-cmd --reload
firewall-cmd --zone=external --query-masquerade   # Verify
```

### Port Forwarding (DNAT)
```bash
# Forward incoming :80 to internal server 10.0.0.50:8080
firewall-cmd --zone=public \
  --add-forward-port=port=80:proto=tcp:toport=8080:toaddr=10.0.0.50 \
  --permanent

# Forward :443 to internal HTTPS
firewall-cmd --zone=public \
  --add-forward-port=port=443:proto=tcp:toport=443:toaddr=10.0.0.50 \
  --permanent

# Forward to different port, same machine
firewall-cmd --zone=public \
  --add-forward-port=port=2222:proto=tcp:toport=22 \
  --permanent

firewall-cmd --reload
firewall-cmd --list-forward-ports
```

---

## 5. iptables

### Understanding Chains and Tables
```
Tables:
  filter   — Default. INPUT, OUTPUT, FORWARD chains
  nat      — NAT. PREROUTING, POSTROUTING, OUTPUT chains
  mangle   — Packet modification
  raw      — Connection tracking bypass

Chains (in filter table):
  INPUT    — Packets destined for this host
  OUTPUT   — Packets originating from this host
  FORWARD  — Packets routing through this host

Packet flow:
  Incoming:  NIC → PREROUTING → [routing decision] → INPUT → Application
  Forwarded: NIC → PREROUTING → [routing decision] → FORWARD → POSTROUTING → NIC
  Outgoing:  Application → OUTPUT → POSTROUTING → NIC
```

### Basic iptables Operations
```bash
# View rules
iptables -L -n -v --line-numbers        # filter table
iptables -t nat -L -n -v                # NAT table
iptables -t mangle -L -n -v

# Default policies
iptables -P INPUT DROP              # Drop all input by default
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT           # Allow all output

# Allow established connections (stateful — essential!)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow ICMP (ping)
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Allow SSH
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
# Or from specific source:
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT

# Allow HTTP/HTTPS
iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT

# Rate limiting (protection against SSH brute force)
iptables -A INPUT -p tcp --dport 22 -m limit --limit 3/min --limit-burst 5 -j ACCEPT

# Block specific IP
iptables -A INPUT -s 203.0.113.100 -j DROP

# Log before dropping (last rules)
iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "IPTables-Dropped: " --log-level 4
iptables -A INPUT -j DROP
```

### NAT with iptables
```bash
# SNAT (masquerade) — outgoing traffic from 192.168.1.0/24
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o ens3 -j MASQUERADE

# DNAT (port forward) — incoming port 80 to 10.0.0.50:8080
iptables -t nat -A PREROUTING -i ens3 -p tcp --dport 80 \
  -j DNAT --to-destination 10.0.0.50:8080

# Enable IP forwarding
sysctl -w net.ipv4.ip_forward=1
```

### Custom Chains (for organization)
```bash
# Create custom chain
iptables -N SSH_RULES

# Add rules to custom chain
iptables -A SSH_RULES -s 10.0.0.0/8 -j ACCEPT
iptables -A SSH_RULES -s 192.168.0.0/16 -j ACCEPT
iptables -A SSH_RULES -j DROP

# Jump to custom chain from INPUT
iptables -A INPUT -p tcp --dport 22 -j SSH_RULES
```

### Saving and Restoring Rules
```bash
# Install persistence package
sudo dnf install iptables-services -y
systemctl enable iptables

# Save current rules
iptables-save > /etc/sysconfig/iptables

# Restore
iptables-restore < /etc/sysconfig/iptables

# Delete all rules
iptables -F                         # Flush all rules
iptables -X                         # Delete custom chains
iptables -Z                         # Zero counters
iptables -t nat -F
iptables -t nat -X
iptables -P INPUT ACCEPT            # Reset default policy!
```

---

## 6. nftables

nftables is the modern replacement for iptables (backend for firewalld on RHEL 9).

```bash
# Install
sudo dnf install nftables -y
systemctl enable --now nftables

# View current ruleset
nft list ruleset

# Basic nftables configuration
cat > /etc/nftables.conf << 'EOF'
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        # Loopback
        iif lo accept

        # Established/related
        ct state established,related accept
        ct state invalid drop

        # ICMP
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept

        # SSH (rate limited)
        tcp dport 22 ct state new limit rate 3/minute burst 5 packets accept

        # HTTP/HTTPS
        tcp dport { 80, 443 } accept

        # Log and drop
        log prefix "nft-dropped: " drop
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority 100;
        ip saddr 192.168.1.0/24 oif ens3 masquerade
    }
}
EOF

nft -f /etc/nftables.conf
nft list ruleset                    # Verify
```

---

## 7. Do's and Don'ts

### ✅ Do's

**Do use `--permanent` with `--reload` for persistent rules:**
```bash
firewall-cmd --add-service=https --permanent
firewall-cmd --reload              # Required to activate permanent rules
```

**Do allow ESTABLISHED,RELATED before default DROP:**
```bash
# iptables — most critical rule
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# Without this, replies to your own connections get blocked!
```

**Do test firewall changes with a timeout in mind:**
```bash
# If you lock yourself out of SSH, this saves you:
# Schedule a reload to restore rules in 5 minutes
at now + 5 minutes << 'EOF'
systemctl restart firewalld
# or: iptables -P INPUT ACCEPT && iptables -F
EOF
# Then make your changes — if SSH breaks, wait 5 minutes for auto-recovery
```

**Do log dropped packets for security audit:**
```bash
firewall-cmd --zone=public \
  --add-rich-rule='rule family="ipv4" source address="0.0.0.0/0" log prefix="FW-DROP " level="warning" limit value="5/m" drop' \
  --permanent
```

### ❌ Don'ts

**Don't mix firewalld and iptables directly:**
```bash
# ❌ firewalld manages iptables/nftables — manual rules get wiped on reload
# If using firewalld: use firewalld commands only
# If using raw iptables: disable firewalld first
systemctl disable --now firewalld
```

**Don't set default policy to DROP without allowing SSH first:**
```bash
# ❌ This instantly locks you out if you're remote
iptables -P INPUT DROP     # Run this AFTER adding ESTABLISHED + SSH rules
```

**Don't forget `--permanent` for rules you want to survive reboot:**
```bash
# ❌ Lost after reload or reboot
firewall-cmd --add-service=http

# ✅ Persistent
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
```

**Don't use INPUT DROP without loopback exception:**
```bash
# Many services use localhost (127.0.0.1) — blocking loopback breaks them
iptables -A INPUT -i lo -j ACCEPT    # Must come before DROP policy
```

---

## Quick Reference

```bash
# firewalld
firewall-cmd --list-all
firewall-cmd --add-service=http --permanent
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --add-rich-rule='rule family="ipv4" source address="10.0.0.0/8" service name="ssh" accept' --permanent
firewall-cmd --reload
firewall-cmd --zone=public --add-masquerade --permanent

# iptables
iptables -L -n -v --line-numbers
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -j DROP
iptables-save > /etc/sysconfig/iptables

# nftables
nft list ruleset
nft -f /etc/nftables.conf
```

---

> **Prev:** [07 — Networking on RHEL](./07-networking-rhel.md)  
> **Next:** [09 — SELinux Mastery](./09-selinux.md)  
> **Repository:** [github.com/eknatha/linux](https://github.com/eknatha/linux)
