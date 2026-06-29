# Chapter 15 — Hands-On Projects

## Overview

Four real-world projects that progressively build your skills from beginner to production-grade.

---

## Project 1 — Beginner: Server Setup Script

**Goal:** Automate the initial setup of a fresh Ubuntu server  
**Skills:** File operations, package management, user management, services  
**Time:** 2–3 hours

### Requirements

- Create a non-root admin user with sudo access
- Install essential packages
- Configure SSH security
- Set up basic firewall rules
- Configure timezone and locale
- Create a setup log

### Architecture

```
server-setup.sh
├── 1. Create admin user
├── 2. Install packages (git, vim, curl, htop, ufw)
├── 3. Configure SSH (disable root login, disable password auth)
├── 4. Configure UFW (allow SSH, HTTP, HTTPS)
├── 5. Set timezone
└── 6. Log completion
```

### Implementation

```bash
#!/bin/bash
# server-setup.sh — Initial server hardening
set -euo pipefail

LOG_FILE="/var/log/server-setup.log"
ADMIN_USER="admin"
TIMEZONE="Asia/Kolkata"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

# Must run as root
[[ $EUID -eq 0 ]] || { echo "Run as root"; exit 1; }

log "Starting server setup"

# ─── 1. Create Admin User ──────────────────────────────────────
if ! id "$ADMIN_USER" &>/dev/null; then
    log "Creating user: $ADMIN_USER"
    useradd -m -s /bin/bash -G sudo "$ADMIN_USER"
    # Set initial password (change on first login)
    echo "${ADMIN_USER}:ChangeMe2026!" | chpasswd
    # Force password change on first login
    chage -d 0 "$ADMIN_USER"
    log "User $ADMIN_USER created"
else
    log "User $ADMIN_USER already exists"
fi

# ─── 2. Install Packages ───────────────────────────────────────
log "Installing packages"
apt update -q
apt install -y vim curl wget git htop tmux tree jq ufw fail2ban unattended-upgrades

# ─── 3. SSH Configuration ──────────────────────────────────────
log "Hardening SSH"
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
cat > /etc/ssh/sshd_config.d/hardening.conf << 'EOF'
PermitRootLogin no
PasswordAuthentication no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
EOF
sshd -t && systemctl restart sshd
log "SSH hardened"

# ─── 4. Firewall ───────────────────────────────────────────────
log "Configuring firewall"
ufw --force reset
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
log "Firewall configured"

# ─── 5. Timezone ───────────────────────────────────────────────
log "Setting timezone to $TIMEZONE"
timedatectl set-timezone "$TIMEZONE"
log "Timezone set to: $(timedatectl show --value -p Timezone)"

# ─── 6. Summary ────────────────────────────────────────────────
log "Setup complete!"
log "Summary:"
log "  Admin user: $ADMIN_USER"
log "  UFW status: $(ufw status | head -1)"
log "  SSH: PermitRootLogin=no, PasswordAuth=no"
log "  Timezone: $(date +%Z)"
```

### Extensions

- Add SSH key setup for the admin user
- Configure automatic security updates
- Set up fail2ban for SSH brute force protection
- Add email alerts on user login

---

## Project 2 — Intermediate: Log Analysis Tool

**Goal:** Build a log analysis pipeline for nginx access logs  
**Skills:** `awk`, `grep`, `sed`, `sort`, `uniq`, shell scripting  
**Time:** 3–4 hours

### Requirements

- Parse nginx access logs
- Generate a report with:
  - Total requests
  - Status code distribution
  - Top 10 IPs by request count
  - Top 10 URLs
  - Average response time
  - Error rate percentage
- Output to terminal and save as HTML report
- Accept date filter argument

### Sample Log Format

```
192.168.1.1 - - [23/Jun/2026:10:00:01 +0530] "GET /index.html HTTP/1.1" 200 1234 0.123
10.0.0.5 - - [23/Jun/2026:10:00:02 +0530] "POST /api/login HTTP/1.1" 401 56 0.045
```

### Implementation

```bash
#!/bin/bash
# analyze-logs.sh — Nginx log analyzer
set -euo pipefail

LOG_FILE="${1:-/var/log/nginx/access.log}"
DATE_FILTER="${2:-}"  # optional: filter by date like "23/Jun/2026"
REPORT_FILE="/tmp/nginx-report-$(date +%Y%m%d_%H%M%S).txt"

[[ -f "$LOG_FILE" ]] || { echo "Log file not found: $LOG_FILE"; exit 1; }

# Apply date filter if provided
get_log() {
    if [[ -n "$DATE_FILTER" ]]; then
        grep "\[$DATE_FILTER" "$LOG_FILE"
    else
        cat "$LOG_FILE"
    fi
}

echo "Analyzing: $LOG_FILE"
[[ -n "$DATE_FILTER" ]] && echo "Filter: $DATE_FILTER"
echo ""

TOTAL=$(get_log | wc -l)
echo "Total Requests: $TOTAL"
echo ""

echo "── Status Code Distribution ──"
get_log | awk '{print $9}' | sort | uniq -c | sort -rn | \
    awk '{printf "  HTTP %-5s %5d requests\n", $2, $1}'
echo ""

echo "── Top 10 IPs ──"
get_log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10 | \
    awk '{printf "  %-20s %5d requests\n", $2, $1}'
echo ""

echo "── Top 10 URLs ──"
get_log | awk '{print $7}' | sort | uniq -c | sort -rn | head -10 | \
    awk '{printf "  %5d  %s\n", $1, $2}'
echo ""

echo "── Error Rate ──"
ERRORS=$(get_log | awk '$9 >= 400' | wc -l)
if [[ $TOTAL -gt 0 ]]; then
    ERROR_RATE=$(echo "scale=2; $ERRORS * 100 / $TOTAL" | bc)
    echo "  Errors (4xx/5xx): $ERRORS / $TOTAL = ${ERROR_RATE}%"
fi
echo ""

echo "── Average Response Time (if logged) ──"
AVG_TIME=$(get_log | awk 'NF>=10 {sum += $NF; count++} END {if(count>0) printf "%.3f\n", sum/count}')
[[ -n "$AVG_TIME" ]] && echo "  Average: ${AVG_TIME}s" || echo "  Not available"

echo ""
echo "Report saved to: $REPORT_FILE"
```

---

## Project 3 — Advanced: Automated Deployment Script

**Goal:** Full application deployment with backup, health check, and rollback  
**Skills:** Everything — scripting, services, networking, error handling  
**Time:** 4–6 hours

### Folder Structure

```
deploy/
├── deploy.sh              ← main deployment script
├── config/
│   ├── app.conf           ← app configuration
│   └── deploy.conf        ← deployment configuration
├── scripts/
│   ├── backup.sh          ← backup function
│   ├── health-check.sh    ← health check
│   └── rollback.sh        ← rollback procedure
└── README.md
```

### `deploy.sh`

```bash
#!/bin/bash
# deploy.sh — Production-grade deployment script
set -euo pipefail

# ─── Config ──────────────────────────────────────────────────
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/config/deploy.conf"

LOG_FILE="${LOG_DIR}/deploy_$(date +%Y%m%d_%H%M%S).log"
LOCK_FILE="/tmp/${APP_NAME}.deploy.lock"

# ─── Logging ─────────────────────────────────────────────────
log()    { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO]  $*" | tee -a "$LOG_FILE"; }
warn()   { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [WARN]  $*" | tee -a "$LOG_FILE" >&2; }
error()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [ERROR] $*" | tee -a "$LOG_FILE" >&2; }

# ─── Lock ────────────────────────────────────────────────────
acquire_lock() {
    if ! flock -xn "$LOCK_FILE" true 2>/dev/null; then
        error "Another deploy is in progress"
        exit 1
    fi
    exec 200>"$LOCK_FILE"
    flock -xn 200 || { error "Cannot acquire lock"; exit 1; }
    trap 'flock -u 200; rm -f "$LOCK_FILE"' EXIT
}

# ─── Backup ──────────────────────────────────────────────────
create_backup() {
    local backup_path="${BACKUP_DIR}/${APP_NAME}_$(date +%Y%m%d_%H%M%S)"
    log "Creating backup: $backup_path"
    cp -r "$APP_DIR" "$backup_path"
    echo "$backup_path" > "${BACKUP_DIR}/latest"
    log "Backup created: $backup_path"
}

# ─── Health Check ────────────────────────────────────────────
health_check() {
    local max_attempts="${1:-12}"
    local attempt=0
    log "Running health check (max ${max_attempts} attempts)"
    while [[ $attempt -lt $max_attempts ]]; do
        if curl -sf "${HEALTH_CHECK_URL}" &>/dev/null; then
            log "Health check passed on attempt $((attempt+1))"
            return 0
        fi
        ((attempt++))
        log "Attempt $attempt/$max_attempts failed, retrying in 5s..."
        sleep 5
    done
    return 1
}

# ─── Rollback ────────────────────────────────────────────────
rollback() {
    local backup_path
    backup_path=$(cat "${BACKUP_DIR}/latest" 2>/dev/null) || {
        error "No backup available for rollback"
        return 1
    }
    warn "Rolling back to: $backup_path"
    cp -r "${backup_path}/." "${APP_DIR}/"
    systemctl restart "$SERVICE_NAME"
    if health_check 6; then
        warn "Rollback successful"
    else
        error "Rollback FAILED — manual intervention required"
    fi
}

# ─── Main Deploy ─────────────────────────────────────────────
main() {
    log "==========================================="
    log "Starting deployment of ${APP_NAME}"
    log "Version: ${GIT_BRANCH}"
    log "==========================================="

    acquire_lock

    # Cleanup trap: rollback on failure
    local rollback_needed=false
    trap 'if $rollback_needed; then rollback; fi' ERR

    # Pre-deploy checks
    [[ -d "$APP_DIR" ]] || error "App directory not found: $APP_DIR"
    command -v git &>/dev/null || error "git not installed"
    systemctl is-active --quiet "$SERVICE_NAME" || warn "Service was already stopped"

    # Backup
    create_backup
    rollback_needed=true

    # Deploy
    log "Pulling latest code from $GIT_BRANCH"
    cd "$APP_DIR"
    git fetch origin
    git checkout "${GIT_BRANCH}"
    git pull origin "${GIT_BRANCH}"

    # Install dependencies (example for Python)
    if [[ -f requirements.txt ]]; then
        log "Installing Python dependencies"
        python3 -m pip install -r requirements.txt --quiet
    fi

    # Restart service
    log "Restarting service: $SERVICE_NAME"
    systemctl restart "$SERVICE_NAME"

    # Health check
    if ! health_check; then
        error "Health check failed — rolling back"
        rollback
        rollback_needed=false
        exit 1
    fi

    rollback_needed=false
    log "Deployment completed successfully!"
    log "==========================================="
}

main "$@"
```

---

## Project 4 — Production Capstone: System Monitoring Dashboard

**Goal:** Build a comprehensive system monitoring tool  
**Skills:** All Linux fundamentals + advanced scripting + data presentation  
**Time:** 6–8 hours

### Requirements

1. **Real-time monitoring** of CPU, memory, disk, network
2. **Service health** checks with configurable services list
3. **Log monitoring** — tail and alert on error patterns
4. **Alerting** — console + file when thresholds exceeded
5. **Report generation** — daily summary email/file
6. **History** — save metrics to CSV for trending

### Architecture

```
monitor/
├── monitor.sh          ← main daemon
├── lib/
│   ├── metrics.sh      ← collect metrics
│   ├── alerts.sh       ← threshold checking
│   ├── report.sh       ← generate reports
│   └── display.sh      ← terminal display
├── config/
│   └── monitor.conf    ← thresholds and settings
├── data/
│   └── metrics.csv     ← historical data
└── logs/
    └── monitor.log
```

### `monitor.sh` (Core)

```bash
#!/bin/bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
CONFIG="${SCRIPT_DIR}/config/monitor.conf"

# Defaults (override in config)
INTERVAL=60
CPU_WARN=80
CPU_CRIT=95
MEM_WARN=80
MEM_CRIT=95
DISK_WARN=80
DISK_CRIT=95
SERVICES="nginx docker sshd"
METRICS_FILE="${SCRIPT_DIR}/data/metrics.csv"
LOG_FILE="${SCRIPT_DIR}/logs/monitor.log"

[[ -f "$CONFIG" ]] && source "$CONFIG"

mkdir -p "$(dirname "$METRICS_FILE")" "$(dirname "$LOG_FILE")"

# Initialize CSV
[[ -f "$METRICS_FILE" ]] || echo "timestamp,cpu_pct,mem_pct,disk_pct,load_1m" > "$METRICS_FILE"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }

collect_metrics() {
    local cpu mem disk load

    # CPU idle → usage
    cpu=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | tr -d '%us,')
    cpu=${cpu%.*}

    # Memory usage %
    mem=$(free | awk 'NR==2{printf "%.0f", $3*100/$2}')

    # Root disk usage
    disk=$(df / | awk 'NR==2{print $5}' | tr -d '%')

    # 1-minute load
    load=$(uptime | awk -F'load average:' '{print $2}' | awk -F, '{print $1}' | tr -d ' ')

    # Save to CSV
    echo "$(date +%Y-%m-%dT%H:%M:%S),${cpu},${mem},${disk},${load}" >> "$METRICS_FILE"

    # Alert if thresholds exceeded
    [[ $cpu -gt $CPU_CRIT ]]  && log "CRITICAL: CPU at ${cpu}%"
    [[ $cpu -gt $CPU_WARN ]]  && log "WARNING: CPU at ${cpu}%"
    [[ $mem -gt $MEM_CRIT ]]  && log "CRITICAL: Memory at ${mem}%"
    [[ $disk -gt $DISK_CRIT ]] && log "CRITICAL: Disk at ${disk}%"

    echo "$cpu $mem $disk $load"
}

check_services() {
    for svc in $SERVICES; do
        if ! systemctl is-active --quiet "$svc" 2>/dev/null; then
            log "ALERT: Service DOWN: $svc"
        fi
    done
}

display_dashboard() {
    clear
    echo "╔══════════════════════════════════════════╗"
    echo "║       System Monitor — $(date '+%H:%M:%S')         ║"
    echo "╠══════════════════════════════════════════╣"

    read -r cpu mem disk load <<< "$(collect_metrics)"

    printf "║  CPU:    %3d%%  %-20s       ║\n" "$cpu" "$(python3 -c "print('█'*int($cpu/5) + '░'*(20-int($cpu/5)))" 2>/dev/null || echo "")"
    printf "║  Memory: %3d%%  %-20s       ║\n" "$mem" ""
    printf "║  Disk:   %3d%%  %-20s       ║\n" "$disk" ""
    printf "║  Load:   %-32s  ║\n" "$load"

    echo "╠══════════════════════════════════════════╣"
    echo "║  Services:                               ║"
    for svc in $SERVICES; do
        if systemctl is-active --quiet "$svc" 2>/dev/null; then
            printf "║    %-15s  ✓ running              ║\n" "$svc"
        else
            printf "║    %-15s  ✗ DOWN                 ║\n" "$svc"
        fi
    done
    echo "╚══════════════════════════════════════════╝"
    echo "  Press Ctrl+C to stop | Refresh: ${INTERVAL}s"
}

log "Monitor started (interval: ${INTERVAL}s)"
while true; do
    display_dashboard
    check_services
    sleep "$INTERVAL"
done
```

### Extensions

- Send Slack/email alerts when thresholds exceeded
- Generate weekly trending graphs (with gnuplot)
- Add network bandwidth monitoring
- Implement metric storage in a proper time-series format
- Build a web dashboard with `bash` + `netcat` serving JSON

---

## Project Completion Checklist

For each project:
- [ ] Script runs without errors on a fresh Ubuntu 22.04
- [ ] `shellcheck` shows no issues
- [ ] All functions have error handling
- [ ] Logs are written with timestamps
- [ ] Script has usage/help (`-h` flag)
- [ ] Tested with edge cases (empty files, missing dependencies)
- [ ] Code is commented where non-obvious

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="14-common-mistakes-and-pitfalls.md">← Previous: Common Mistakes & Pitfalls</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="16-interview-preparation.md">Next: Interview Preparation →</a>
</div>
