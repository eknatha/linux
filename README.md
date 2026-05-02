# LinuxLab — linux.eknathalabs.com

> **A fully offline, browser-based Linux learning lab — LFCS exam prep, interactive tools, cheatsheet, quiz, and hands-on labs. No server. No API. No signup.**

Built by [Eknatha Reddy Puli](https://eknathalabs.com) 

---

## 🚀 Live Demo

**[linux.eknathalabs.com](https://linux.eknathalabs.com)**

---

## 📸 What's Inside

```
index.html  ←  Single file. Drop anywhere. Works offline.
README.md   ←  You are here.
```

No build step. No `npm install`. No framework. Open `index.html` in any browser and everything works.

---

## ✨ Features

### 🌙 Dark / Light Mode
Toggle between dark terminal aesthetic and clean light theme. Preference persists via `localStorage`.

---

### ⚡ Command of the Day
A curated Linux command shown on load — rotates daily (30-command database). Real production commands: `strace`, `awk` IP extraction, `openssl` cert expiry checks, `lvextend + resize2fs`, and more.

---

### 📚 14 Learning Modules
Structured learning path from beginner to advanced — filterable by level and LFCS exam domain.

| Module | Level | LFCS Domain |
|--------|-------|-------------|
| Essential Commands | Beginner | ✓ 20% |
| Users, Groups & Permissions | Beginner | ✓ 10% |
| systemd & Service Management | Beginner | ✓ 25% |
| Storage & Filesystems | Intermediate | ✓ 20% |
| LVM — Logical Volume Manager | Intermediate | — |
| Networking Fundamentals | Intermediate | ✓ 25% |
| Firewall — iptables & firewalld | Intermediate | — |
| SSH & Remote Access | Intermediate | — |
| Package Management | Intermediate | — |
| Process & Resource Management | Intermediate | — |
| Cron & Task Scheduling | Intermediate | — |
| Shell Scripting for Sysadmins | Advanced | — |
| Containers & Docker | Advanced | ✓ |
| SELinux & AppArmor | Advanced | — |

Click any module card to open a full command reference modal.

---

### 🔧 10 Interactive Offline Tools

All tools are pure HTML + JavaScript — no API calls, no internet required.

#### 🔴 Error Encyclopedia
- 16 common Linux errors (searchable)
- Each entry: root cause → exact fix commands → prevention
- Covers: EACCES, ENOSPC, ECONNREFUSED, ETIMEDOUT, GRUB rescue, OOM killer, SELinux AVC, dpkg lock, and more

#### ⚙️ systemd Unit Builder
- Form-based generator: description, ExecStart, WorkingDirectory, User, Restart policy, environment vars
- Optional security hardening block (`NoNewPrivileges`, `ProtectSystem`, `PrivateTmp`)
- Copy-ready `.service` file output with deploy instructions

#### 🔐 Permission Calculator
- Click `rwx` checkboxes for Owner / Group / Other
- Live output: octal value, symbolic notation, `chmod` commands
- 8 quick presets: `755`, `644`, `600`, `777`, `750`, `640`, `444`, `400`

#### 🕐 Cron Expression Builder
- 5-field visual builder (Minute / Hour / Day / Month / Weekday)
- Live expression display with plain-English description
- Next 6 predicted run times
- `systemd` `OnCalendar` equivalent
- 7 quick presets (hourly, daily, weekly, monthly, every 5 min, weekdays 9am)

#### 🌐 Port Reference
- 39 well-known ports (TCP/UDP) — searchable by number or service name
- Click any row to copy the `ss` or `iptables` command for that port
- Covers: SSH, HTTP, HTTPS, MySQL, PostgreSQL, Redis, MongoDB, Kafka, Kubernetes, Docker, Elasticsearch, Prometheus, and more

#### 💾 LVM Command Generator
- Input: disk(s), VG name, LV name, size, filesystem, mount point
- Output: complete shell script — `pvcreate` → `vgcreate` → `lvcreate` → `mkfs` → `mount` → `/etc/fstab` entry
- Includes online extend commands for later

#### 📡 Signals Reference
- All 25 key Linux signals with number, name, default action, and common use case
- Searchable
- Click any row to copy `kill -N PID` command

#### 🔍 grep / Regex Tester
- Paste sample text (log lines, config output, etc.)
- Type a regex pattern — matches highlight live
- Toggle flags: `-i` (case insensitive), `-E` (extended), `-v` (invert), `-n` (line numbers)
- Shows match count and outputs a ready-to-run `grep` command

#### 🧮 Subnet Calculator
- Input: IP address + CIDR prefix (0–32)
- Output: Network address, Subnet mask, Wildcard mask, Broadcast address, First/Last usable host, Usable host count, Total addresses
- 9 CIDR presets: `/8`, `/16`, `/24`, `/25`, `/26`, `/27`, `/28`, `/30`, `/32`
- Generates: `ip addr add`, `ip route`, `nmap -sn`, `iptables` commands for the subnet

#### 📋 Log Analyzer
- Paste any log: syslog, journalctl, nginx access/error, dmesg, application logs
- Color-coded output: `CRIT` (red) · `ERROR` (orange) · `WARN` (amber) · `INFO` (green) · `DEBUG` (gray)
- Severity count dashboard (Total / CRIT / ERROR / WARN / INFO)
- Extracts all IP addresses with frequency count — click any to copy
- Detects repeated lines (crash loops, log spam)

---

### ⌨ Cheatsheet
20 production-grade commands — live search, click any row to copy to clipboard.

| Category | Examples |
|----------|---------|
| Find & Search | `find / -perm /4000`, `grep -r "pattern" /etc/` |
| Service Mgmt | `systemctl list-units --failed`, `journalctl -u nginx` |
| Networking | `ss -tlnp`, `nmcli con mod eth0 ipv4.method manual` |
| Storage & LVM | `lvcreate -L 10G -n data vg0`, `lsblk -f`, `df -hT` |
| Security | `ausearch -m avc -ts recent`, `getfacl /path` |
| Process | `ps aux --sort=-%mem | head -10`, `strace -p $PID` |

---

### ✦ Quiz — 10 LFCS Questions
- Scenario-based multiple-choice questions
- Mapped to official LFCS exam domains (Networking 25% · systemd 25% · Storage 20% · Commands 20% · Permissions 10%)
- Score tracking, skip option, instant feedback with explanation
- Result screen with grade and domain breakdown

---

### ⚗ Practice Labs — 6 Scenarios
Step-by-step labs modelled on real production incidents and LFCS exam tasks. Filterable by level.

| # | Lab | Level | Time |
|---|-----|-------|------|
| 01 | Recover a Broken Boot — GRUB Rescue | Intermediate | ~45 min |
| 02 | LVM Full Disk — Expand Without Downtime | Intermediate | ~30 min |
| 03 | Nginx Fails to Start — Port Conflict Debug | Beginner | ~20 min |
| 04 | SSH Hardening — No-Password Auth | Intermediate | ~35 min |
| 05 | Firewall: Isolate a Compromised Web Server | Advanced | ~40 min |
| 06 | SELinux Denial — Fix Without Disabling | Advanced | ~45 min |

---

### 🔴 Error Encyclopedia (Main Section)
8 common Linux errors with expandable accordion. Filterable by category: systemd · Network · Storage · Permissions · Boot.

---

### 🗺 LFCS Domain Coverage
Visual domain bars mapped to the official LFCS exam blueprint:

| Domain | Weight |
|--------|--------|
| Operations & Deployment | 25% |
| Networking | 25% |
| Storage | 20% |
| Essential Commands | 20% |
| Users & Groups | 10% |

---

## 🏗 Tech Stack

| Layer | Choice | Why |
|-------|--------|-----|
| Structure | Vanilla HTML5 | Zero dependencies |
| Styling | CSS custom properties | Theme toggle, no framework |
| Logic | Vanilla JavaScript (ES5 compatible) | Works in any browser, offline |
| Fonts | System fonts (`system-ui`) | No external requests |
| Storage | `localStorage` | Theme persistence — no backend |
| Hosting | Any static host | GitHub Pages, Netlify, Apache, Nginx — all work |

**No build step. No bundler. No CDN dependencies. No cookies. No tracking.**

---

## 📁 File Structure

```
linuxlab/
├── index.html      # Everything — 1 file, ~138KB, fully self-contained
└── README.md       # This file
```

All CSS, JavaScript, data (error DB, port DB, signal DB, quiz questions, COTD commands, module content) is embedded directly in `index.html`.

---

## 🗺 Roadmap

- [ ] **LFCS Task Simulator** — type commands in response to real exam-style task descriptions
- [ ] **Module Progress Tracker** — localStorage checkmarks, % complete per LFCS domain
- [ ] **Flashcard Mode** — flip-card command → explanation with keyboard navigation
- [ ] **Timed Exam Mode** — 20 questions, 2-hour countdown mirroring real LFCS
- [ ] **Domain Weakness Heatmap** — visual score per LFCS domain after quiz
- [ ] **Incident Runbook Generator** — fill service + symptoms → structured runbook
- [ ] **systemd Unit Validator** — paste a unit file, check for common mistakes
- [ ] **Print Cheatsheet** — one-click print-optimized PDF of filtered commands

---

## 👤 Author

**Eknatha Reddy** 

| Link | URL |
|------|-----|
| 🌐 Main Site | [eknathalabs.com](https://eknathalabs.com) |
| 🐧 LinuxLab | [linux.eknathalabs.com](https://linux.eknathalabs.com) |
| 🐙 GitHub | [github.com/eknathareddyp](https://github.com/eknathareddyp) |
| 🐳 KubeLab | [kubelab.eknathalabs.com](https://kubelab.eknathalabs.com) |
| 🏗 TerraformLab | [terraform.eknathalabs.com](https://terraform.eknathalabs.com) |
| 🏗 DockerLab | [docker.eknathalabs.com](https://docker.eknathalabs.com) |

---

## 📄 License

MIT — free to use, fork, and deploy. Attribution appreciated but not required.

---

> *"The best way to learn Linux is to break things and fix them. LinuxLab gives you the reference to fix them faster."*
>
> — Eknatha Reddy Puli
