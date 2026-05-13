# 11 — Shell Scripting for SysAdmins (RHEL)

> **Author:** Eknatha Reddy P · [eknathalabs.com](https://eknathalabs.com) · [github.com/eknatha](https://github.com/eknatha)  
> **Applies to:** RHEL 8/9, CentOS Stream, Rocky Linux, AlmaLinux  
> **Level:** Advanced

---

## Table of Contents
1. [Script Boilerplate & Safety Flags](#1-script-boilerplate)
2. [Variables & Substitution](#2-variables)
3. [Conditionals & Tests](#3-conditionals)
4. [Loops](#4-loops)
5. [Functions](#5-functions)
6. [Error Handling](#6-error-handling)
7. [Input/Output & Redirection](#7-io--redirection)
8. [String & Text Processing](#8-string-processing)
9. [Real-World Automation Scripts](#9-real-world-scripts)
10. [Do's and Don'ts](#10-dos-and-donts)

---

## 1. Script Boilerplate

Every production sysadmin script should start with:

```bash
#!/usr/bin/env bash
# ============================================================
# Script: disk-alert.sh
# Description: Alert if disk usage exceeds threshold
# Author: Eknatha Reddy P (eknathalabs.com)
# Version: 1.0.0
# Usage: ./disk-alert.sh [threshold%] [email]
# ============================================================

set -euo pipefail
# set -e  — Exit immediately on error
# set -u  — Treat unset variables as errors
# set -o pipefail — Pipeline fails if any command fails

# Trap for cleanup on exit, interrupt, and error
trap cleanup EXIT INT TERM

# Global variables
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
readonly LOG_FILE="/var/log/${SCRIPT_NAME%.sh}.log"
readonly TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

# Configuration with defaults
THRESHOLD="${1:-80}"
ALERT_EMAIL="${2:-admin@company.com}"
DRY_RUN="${DRY_RUN:-false}"

# Logging function
log() {
    local level="$1"
    shift
    echo "[${TIMESTAMP}] [${level}] $*" | tee -a "$LOG_FILE"
}

log_info()  { log "INFO"  "$@"; }
log_warn()  { log "WARN"  "$@"; }
log_error() { log "ERROR" "$@" >&2; }

cleanup() {
    local exit_code=$?
    if [[ $exit_code -ne 0 ]]; then
        log_error "Script exited with code: $exit_code"
    fi
    # Add cleanup: remove temp files, release locks, etc.
}

# Usage function
usage() {
    cat << EOF
Usage: $SCRIPT_NAME [OPTIONS] [threshold] [email]

Options:
  -h, --help    Show this help
  -n, --dry-run Don't send alerts, just print

Example:
  $SCRIPT_NAME 85 ops@company.com
EOF
    exit 0
}
```

---

## 2. Variables

```bash
# Variable assignment (no spaces around =)
NAME="eknatha"
COUNT=0
readonly MAX=100                    # Immutable constant

# String quoting rules
echo "$NAME"                        # Double: expands variables, safe with spaces
echo '$NAME'                        # Single: literal, no expansion
echo "${NAME}_suffix"               # Brace notation: clear variable boundaries

# Command substitution
DATE=$(date +%Y-%m-%d)             # Preferred syntax
DATE=`date +%Y-%m-%d`             # Old syntax (avoid in new scripts)
DISK_FREE=$(df -h / | awk 'NR==2 {print $4}')

# Arithmetic
COUNT=$((COUNT + 1))
RESULT=$((5 * 3 + 2))
((COUNT++))
((COUNT--))

# Arrays
SERVERS=("web01" "web02" "db01" "db02")
echo "${SERVERS[0]}"               # First element
echo "${SERVERS[@]}"               # All elements
echo "${#SERVERS[@]}"              # Number of elements
SERVERS+=("web03")                 # Append

# Associative arrays (bash 4+)
declare -A CONFIG
CONFIG[host]="10.0.0.50"
CONFIG[port]="5432"
CONFIG[db]="appdb"
echo "${CONFIG[host]}"

# Special variables
$0      # Script name
$1..$9  # Positional arguments
$@      # All arguments (array)
$*      # All arguments (string)
$#      # Number of arguments
$?      # Exit code of last command
$$      # Current shell PID
$!      # PID of last background command

# Parameter expansion
FILE="/var/log/nginx/access.log"
echo "${FILE%.*}"                  # Remove suffix: /var/log/nginx/access
echo "${FILE##*/}"                 # Remove path prefix: access.log
echo "${FILE%/*}"                  # Directory: /var/log/nginx
echo "${FILE#/var/}"               # Remove prefix: log/nginx/access.log
echo "${NAME:-default}"            # Use 'default' if NAME is unset/empty
echo "${NAME:=default}"            # Assign 'default' if NAME is unset/empty
echo "${NAME:?'NAME is required'}" # Error and exit if NAME is unset
echo "${NAME:+set}"                # Use 'set' if NAME is set
echo "${NAME^^}"                   # Uppercase
echo "${NAME,,}"                   # Lowercase
echo "${#NAME}"                    # Length of NAME
echo "${NAME:2:5}"                 # Substring: start=2, length=5
```

---

## 3. Conditionals

### if / elif / else
```bash
# File tests
if [[ -f /etc/nginx/nginx.conf ]]; then
    echo "Config exists"
fi

if [[ -d /opt/app ]] && [[ -x /opt/app/start.sh ]]; then
    /opt/app/start.sh
elif [[ -d /opt/app ]]; then
    echo "App dir exists but start.sh is not executable"
else
    echo "App not found"
fi

# String comparison (use [[ ]] not [ ] for safety)
if [[ "$ENVIRONMENT" == "production" ]]; then
    echo "Production!"
elif [[ "$ENVIRONMENT" =~ ^(staging|qa)$ ]]; then
    echo "Staging or QA"
fi

# Number comparison
if (( DISK_USAGE > 80 )); then
    echo "Disk critical!"
fi
# or: if [[ $DISK_USAGE -gt 80 ]]; then ...
```

### Test Operators
```bash
# File tests
[[ -e path ]]    # Exists (any type)
[[ -f path ]]    # Regular file
[[ -d path ]]    # Directory
[[ -L path ]]    # Symlink
[[ -r path ]]    # Readable
[[ -w path ]]    # Writable
[[ -x path ]]    # Executable
[[ -s path ]]    # Non-empty file
[[ -z path ]]    # File exists and has zero size

# String tests
[[ -z "$var" ]]   # Empty string
[[ -n "$var" ]]   # Non-empty string
[[ "$a" == "$b" ]]
[[ "$a" != "$b" ]]
[[ "$str" =~ ^regex$ ]]   # Regex match

# Number tests ([ ] style)
[ $a -eq $b ]   # Equal
[ $a -ne $b ]   # Not equal
[ $a -lt $b ]   # Less than
[ $a -le $b ]   # Less than or equal
[ $a -gt $b ]   # Greater than
[ $a -ge $b ]   # Greater than or equal
```

### case Statement
```bash
case "$1" in
    start)
        systemctl start myapp
        ;;
    stop)
        systemctl stop myapp
        ;;
    restart|reload)
        systemctl restart myapp
        ;;
    status)
        systemctl status myapp
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|reload|status}"
        exit 1
        ;;
esac
```

---

## 4. Loops

### for Loop
```bash
# Over list
for server in web01 web02 db01; do
    echo "Checking $server..."
    ssh "$server" 'df -h /'
done

# Over array
SERVERS=("web01" "web02" "db01")
for server in "${SERVERS[@]}"; do
    ping -c 1 -W 2 "$server" && echo "$server: UP" || echo "$server: DOWN"
done

# C-style
for ((i=1; i<=10; i++)); do
    echo "Iteration $i"
done

# Over files (preferred over ls)
for logfile in /var/log/*.log; do
    size=$(stat -c%s "$logfile")
    echo "$logfile: ${size} bytes"
done

# Range
for i in {1..5}; do echo "$i"; done
for i in {0..100..10}; do echo "$i"; done   # 0, 10, 20, ... 100
```

### while Loop
```bash
# Read file line by line
while IFS= read -r line; do
    echo "Processing: $line"
done < /etc/hosts

# Countdown
count=10
while (( count > 0 )); do
    echo "Countdown: $count"
    ((count--))
    sleep 1
done

# Polling loop
while ! systemctl is-active --quiet myapp; do
    echo "Waiting for myapp to start..."
    sleep 2
done
echo "myapp is running!"
```

---

## 5. Functions

```bash
# Define functions (before calling them)
check_root() {
    if [[ $EUID -ne 0 ]]; then
        log_error "This script must be run as root"
        exit 1
    fi
}

# Function with return value
is_service_running() {
    local service="$1"
    systemctl is-active --quiet "$service"
    return $?   # 0=running, non-zero=not running
}

# Function with output (use echo + command substitution)
get_disk_usage() {
    local mountpoint="${1:-/}"
    df -h "$mountpoint" | awk 'NR==2 {print $5}' | tr -d '%'
}

# Using functions
check_root

DISK_PCT=$(get_disk_usage "/var")
log_info "Disk usage on /var: ${DISK_PCT}%"

if is_service_running nginx; then
    log_info "nginx is running"
else
    log_warn "nginx is not running — attempting restart"
    systemctl start nginx
fi
```

### Local Variables in Functions
```bash
process_server() {
    local server="$1"              # local prevents global variable leak
    local result
    local status

    result=$(ssh "$server" 'uptime' 2>&1)
    status=$?

    if (( status == 0 )); then
        echo "OK: $server — $result"
    else
        echo "FAIL: $server — SSH error"
        return 1
    fi
}
```

---

## 6. Error Handling

```bash
# Pattern 1: Exit on error with message
die() {
    echo "ERROR: $*" >&2
    exit 1
}

[[ -f /etc/nginx/nginx.conf ]] || die "nginx.conf not found"

# Pattern 2: Check command result
if ! nginx -t; then
    die "nginx configuration test failed"
fi

# Pattern 3: Capture and check
output=$(dnf install nginx -y 2>&1)
exit_code=$?
if (( exit_code != 0 )); then
    log_error "dnf install failed: $output"
    exit 1
fi

# Pattern 4: Retry with backoff
retry() {
    local max_attempts="${1}"; shift
    local delay="${1}"; shift
    local attempt=1

    while (( attempt <= max_attempts )); do
        if "$@"; then
            return 0
        fi
        log_warn "Attempt $attempt/$max_attempts failed. Retrying in ${delay}s..."
        sleep "$delay"
        ((attempt++))
        delay=$((delay * 2))   # Exponential backoff
    done

    log_error "All $max_attempts attempts failed for: $*"
    return 1
}

# Usage: retry 5 2 curl -sf http://api/health
```

### Locking (Prevent Concurrent Runs)
```bash
LOCKFILE="/var/run/${SCRIPT_NAME%.sh}.lock"

acquire_lock() {
    if ! mkdir "$LOCKFILE" 2>/dev/null; then
        local pid
        pid=$(cat "$LOCKFILE/pid" 2>/dev/null || echo "unknown")
        die "Script already running (PID: $pid). Remove $LOCKFILE if stale."
    fi
    echo $$ > "$LOCKFILE/pid"
}

release_lock() {
    rm -rf "$LOCKFILE"
}

# In cleanup trap:
cleanup() {
    release_lock
}

acquire_lock
```

---

## 7. I/O & Redirection

```bash
# Redirect stdout
command > file.txt             # Overwrite
command >> file.txt            # Append

# Redirect stderr
command 2> error.txt
command 2>> error.txt

# Redirect both
command > file.txt 2>&1
command &> file.txt            # Bash shorthand

# /dev/null
command > /dev/null            # Discard stdout
command 2>/dev/null            # Discard stderr
command &>/dev/null            # Discard everything

# Pipe stderr through grep
command 2>&1 | grep "error"

# tee — write to file AND stdout
command | tee output.txt
command | tee -a output.txt    # Append mode

# Here-doc
cat << 'EOF'                   # Single-quoted: no variable expansion
server {
    listen 80;
    server_name $hostname;     # Literal $hostname — not expanded
}
EOF

cat << EOF                     # Double-quoted (default): variables expand
server {
    listen 80;
    server_name $HOSTNAME;     # Expanded
}
EOF

# Here-string
grep "pattern" <<< "test string with pattern"
```

---

## 8. String Processing

```bash
# grep
grep "pattern" file.txt
grep -i "pattern" file.txt     # Case insensitive
grep -r "pattern" /etc/        # Recursive
grep -l "pattern" *.conf       # Files with match
grep -c "error" /var/log/app.log  # Count matches
grep -v "DEBUG" app.log        # Invert match
grep -E "error|warning" app.log  # Extended regex

# sed
sed 's/old/new/' file          # Replace first occurrence
sed 's/old/new/g' file         # Replace all occurrences
sed 's/old/new/gi' file        # Case-insensitive global replace
sed -i 's/old/new/g' file      # Edit in place
sed -i.bak 's/old/new/g' file  # Edit in place, backup to .bak
sed -n '5,10p' file            # Print lines 5-10
sed '/^#/d' file               # Delete comment lines
sed '/^$/d' file               # Delete empty lines

# awk
awk '{print $1}' file          # Print first field
awk -F: '{print $1}' /etc/passwd  # Custom delimiter
awk 'NR==2 {print $4}' file    # Second line, fourth field
awk '{sum+=$1} END {print sum}' file  # Sum first column
awk '$3 > 100 {print}' file    # Print if third field > 100
awk '/pattern/ {print NR": "$0}' file  # Pattern with line number
awk 'NR%2==0 {print}' file     # Even lines only

# cut
cut -d: -f1 /etc/passwd        # First field, colon delimiter
cut -c1-10 file                # First 10 characters

# sort & uniq
sort file
sort -n file                   # Numeric sort
sort -r file                   # Reverse
sort -k2 -n file               # Sort by second field numerically
sort | uniq                    # Unique lines
sort | uniq -c | sort -rn      # Count occurrences, sort by count
uniq -d file                   # Only duplicate lines

# tr
echo "hello" | tr 'a-z' 'A-Z'  # Uppercase
echo "hello world" | tr ' ' '\n'  # Words to lines
echo "  spaces  " | tr -s ' '    # Squeeze spaces
cat file | tr -d '\r'            # Remove Windows carriage returns
```

---

## 9. Real-World Scripts

### Disk Space Monitor
```bash
#!/usr/bin/env bash
set -euo pipefail

THRESHOLD="${1:-85}"
ALERT_EMAIL="${2:-ops@company.com}"

check_disk() {
    while IFS= read -r line; do
        local usage mountpoint
        usage=$(echo "$line" | awk '{print $5}' | tr -d '%')
        mountpoint=$(echo "$line" | awk '{print $6}')

        if (( usage >= THRESHOLD )); then
            local msg="DISK ALERT: $mountpoint at ${usage}% on $(hostname)"
            echo "$msg"
            echo "$msg" | mail -s "[DISK ALERT] $(hostname)" "$ALERT_EMAIL" || true
        fi
    done < <(df -h --output=pcent,target | tail -n +2 | grep -v tmpfs)
}

check_disk
```

### Service Health Check with Restart
```bash
#!/usr/bin/env bash
set -euo pipefail

SERVICES=("nginx" "postgresql" "redis")
LOG="/var/log/health-check.log"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG"; }

for svc in "${SERVICES[@]}"; do
    if systemctl is-active --quiet "$svc"; then
        log "OK: $svc is running"
    else
        log "WARN: $svc is down — attempting restart"
        if systemctl restart "$svc"; then
            log "OK: $svc restarted successfully"
        else
            log "ERROR: Failed to restart $svc"
            # Send alert...
        fi
    fi
done
```

### Log Archiver
```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_DIR="${1:-/var/log/nginx}"
ARCHIVE_DIR="/backup/logs"
KEEP_DAYS=30
DATE=$(date +%Y%m%d)

mkdir -p "$ARCHIVE_DIR"

# Compress logs older than 1 day
find "$LOG_DIR" -name "*.log.*" -mtime +1 ! -name "*.gz" \
    -exec gzip -9 {} \;

# Archive compressed logs older than 7 days
find "$LOG_DIR" -name "*.gz" -mtime +7 \
    -exec mv {} "$ARCHIVE_DIR/" \;

# Remove archives older than 30 days
find "$ARCHIVE_DIR" -name "*.gz" -mtime +$KEEP_DAYS -delete

echo "Log archival complete: $(date)"
```

---

## 10. Do's and Don'ts

### ✅ Do's

**Do always use `set -euo pipefail`:**
```bash
set -euo pipefail    # At the top of every script
```

**Do quote all variables:**
```bash
FILE="$1"
cat "$FILE"          # Handles spaces in filenames
# Not: cat $FILE    # Breaks on "my file.txt"
```

**Do use `[[ ]]` over `[ ]`:**
```bash
[[ -f "$file" ]]    # Safer, supports =~, no word splitting
```

**Do use `local` in functions:**
```bash
my_func() {
    local my_var="$1"   # Doesn't pollute global scope
}
```

**Do test with `bash -n` and `shellcheck`:**
```bash
bash -n script.sh          # Syntax check
shellcheck script.sh       # Static analysis (install: dnf install ShellCheck)
```

### ❌ Don'ts

**Don't parse ls output:**
```bash
# ❌ Breaks on spaces and special chars
for f in $(ls /tmp); do ...

# ✅ Use glob or find
for f in /tmp/*; do [[ -f "$f" ]] && process "$f"; done
find /tmp -maxdepth 1 -type f -print0 | xargs -0 process
```

**Don't ignore exit codes:**
```bash
# ❌ Fails silently
cp /important/file /backup/
rm /important/file          # Runs even if cp failed!

# ✅ Check each step
cp /important/file /backup/ || { echo "Backup failed!"; exit 1; }
rm /important/file
```

**Don't use `echo` for passwords or secrets:**
```bash
# ❌ Appears in process list and logs
echo "$DB_PASSWORD" | mysql -u root -p

# ✅ Use environment variables or files
mysql -u root -p"${DB_PASSWORD}" -e "SHOW DATABASES"
# or: mysql --defaults-file=/root/.my.cnf
```

---

## Quick Reference

```bash
# Safety header
#!/usr/bin/env bash
set -euo pipefail
trap cleanup EXIT

# Variables
VAR="${1:-default}"
echo "${VAR:-unset}"
echo "${VAR:?'VAR is required'}"

# Conditionals
[[ -f "$file" ]] && echo "exists"
[[ "$str" =~ ^[0-9]+$ ]] && echo "numeric"
(( count > 10 )) && echo "large"

# Loops
for f in /var/log/*.log; do echo "$f"; done
while IFS= read -r line; do echo "$line"; done < file

# Functions
my_func() { local arg="$1"; echo "$arg"; }
result=$(my_func "value")

# Error handling
command || die "command failed"
output=$(command 2>&1) || die "failed: $output"
```

---

> **Prev:** [10 — Package Management (RHEL)](./10-package-management-rhel.md)  
> **Next:** [12 — Performance Tuning & Troubleshooting](./12-performance-tuning.md)  
> **Repository:** [github.com/eknatha/linux](https://github.com/eknatha/linux)
