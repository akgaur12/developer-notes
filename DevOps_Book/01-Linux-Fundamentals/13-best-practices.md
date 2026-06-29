# Chapter 13 — Linux Best Practices for DevOps

## Learning Objectives

By the end of this chapter, you will:
- Apply production-grade Linux security practices
- Write reliable, maintainable shell scripts
- Establish good habits for server management
- Understand least-privilege principles
- Implement proper logging and monitoring
- Follow DevOps-specific Linux conventions

## Prerequisites

- Chapters 01–12

---

## 13.1 Security Best Practices

### SSH Security

```bash
# /etc/ssh/sshd_config — harden SSH
sudo vim /etc/ssh/sshd_config
```

```ini
# Essential SSH hardening settings

# Disable root login (use sudo instead)
PermitRootLogin no

# Disable password authentication (use keys only)
PasswordAuthentication no
PubkeyAuthentication yes

# Change default port (reduces bot noise)
Port 2222

# Limit login attempts
MaxAuthTries 3

# Disconnect idle sessions
ClientAliveInterval 300
ClientAliveCountMax 2

# Allow specific users only
AllowUsers akash deploy

# Disable old protocols
Protocol 2
```

```bash
# Restart SSH after config changes
sudo systemctl restart sshd

# Test config before restarting (never lock yourself out!)
sudo sshd -t               # test config syntax
```

### Firewall Configuration

```bash
# Default deny, explicitly allow needed ports
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh           # or: ufw allow 22
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
sudo ufw status verbose

# Advanced: allow from specific subnet only
sudo ufw allow from 10.0.0.0/8 to any port 5432    # PostgreSQL from internal only
```

### Least Privilege Principle

```bash
# Never run services as root
# Use dedicated service accounts
sudo useradd --system --no-create-home --shell /usr/sbin/nologin appuser

# Set minimal permissions
chmod 640 /opt/app/config.env    # owner=rw, group=r, other=none
chown appuser:appuser /opt/app/

# Use sudo for specific commands only (not full sudo access)
# In /etc/sudoers.d/deploy:
echo "deploy ALL=(ALL) NOPASSWD: /bin/systemctl restart myapp" | \
  sudo tee /etc/sudoers.d/deploy
```

### Secret Management

```bash
# NEVER put passwords in scripts or environment variables in production
# BAD:
export DB_PASSWORD="mysecret"

# BETTER: read from a restricted file
DB_PASSWORD=$(cat /etc/myapp/secrets/db_password)
chmod 600 /etc/myapp/secrets/db_password
chown appuser:appuser /etc/myapp/secrets/db_password

# BEST: use a secrets manager (Vault, AWS Secrets Manager, etc.)
DB_PASSWORD=$(aws secretsmanager get-secret-value --secret-id prod/db/password --query SecretString --output text)
```

### Log Sensitive Operations

```bash
# All authentication events are logged to /var/log/auth.log
# Monitor it:
tail -f /var/log/auth.log

# Set up audit logging for sensitive files
sudo apt install auditd
sudo auditctl -w /etc/passwd -p wa -k passwd_changes
sudo auditctl -w /etc/sudoers -p wa -k sudoers_changes
sudo ausearch -k passwd_changes
```

---

## 13.2 Shell Script Best Practices

### Script Template

```bash
#!/bin/bash
# ─────────────────────────────────────────────────────────────
# script-name.sh
# Description: What this script does
# Usage: ./script-name.sh [options] <arguments>
# ─────────────────────────────────────────────────────────────
set -euo pipefail

# ─── Constants (UPPER_CASE) ───────────────────────────────────
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "$0")"
readonly LOG_FILE="/var/log/myapp/${SCRIPT_NAME%.sh}.log"

# ─── Logging ──────────────────────────────────────────────────
log()   { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO]  $*" | tee -a "$LOG_FILE"; }
warn()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [WARN]  $*" | tee -a "$LOG_FILE" >&2; }
error() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [ERROR] $*" | tee -a "$LOG_FILE" >&2; }
die()   { error "$*"; exit 1; }

# ─── Cleanup ──────────────────────────────────────────────────
cleanup() {
    local exit_code=$?
    [[ $exit_code -ne 0 ]] && error "Script failed with code $exit_code"
    # Add cleanup here: rm temp files, release locks
}
trap cleanup EXIT

# ─── Help ─────────────────────────────────────────────────────
usage() {
    cat << EOF
Usage: $SCRIPT_NAME [OPTIONS] ARGUMENT

Description of what the script does.

OPTIONS:
    -h, --help      Show this help
    -v, --verbose   Verbose output
    -d, --dry-run   Dry run (no changes)

ARGUMENTS:
    ARGUMENT        Description of argument
EOF
}

# ─── Argument Parsing ─────────────────────────────────────────
VERBOSE=false
DRY_RUN=false

while [[ $# -gt 0 ]]; do
    case "$1" in
        -h|--help)    usage; exit 0 ;;
        -v|--verbose) VERBOSE=true ;;
        -d|--dry-run) DRY_RUN=true ;;
        --)           shift; break ;;
        -*)           die "Unknown option: $1" ;;
        *)            break ;;
    esac
    shift
done

[[ $# -lt 1 ]] && { usage; die "Missing required argument"; }
ARGUMENT="$1"

# ─── Main ─────────────────────────────────────────────────────
main() {
    log "Starting $SCRIPT_NAME"
    log "Argument: $ARGUMENT"

    if $DRY_RUN; then
        log "DRY RUN: would do something with $ARGUMENT"
    else
        log "Doing something with $ARGUMENT"
        # actual work here
    fi

    log "Completed successfully"
}

main "$@"
```

### Key Script Rules

```bash
# 1. Always use set -euo pipefail
set -euo pipefail

# 2. Quote all variables
rm -rf "${old_dir}"          # not: rm -rf $old_dir

# 3. Use [[ ]] not [ ]
if [[ "$var" == "value" ]]; then

# 4. Use readonly for constants
readonly MAX_RETRIES=5

# 5. Use local in functions
my_func() {
    local var="value"    # not accessible outside function
}

# 6. Check command existence before using
command -v aws &>/dev/null || die "aws CLI not installed"

# 7. Use proper exit codes
exit 0    # success
exit 1    # generic error
exit 2    # wrong usage

# 8. Validate inputs
[[ -f "$config_file" ]] || die "Config file not found: $config_file"
[[ "$port" =~ ^[0-9]+$ ]] || die "Invalid port: $port"

# 9. Use full paths for cron scripts
readonly AWS=/usr/local/bin/aws   # not just 'aws'

# 10. Run ShellCheck on all scripts
shellcheck myscript.sh
```

---

## 13.3 Server Management Practices

### Never Work Directly on Production

```bash
# Have a staging/QA environment
# Test ALL changes in staging first

# Use configuration management (Ansible, Chef, Puppet) for repeatability
# Don't make manual changes you can't reproduce

# Document every manual change in a runbook or ticket
```

### Backup Before Changes

```bash
# Backup config before editing
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak.$(date +%Y%m%d_%H%M%S)

# Or use version control for configs (GitOps pattern)
cd /etc/nginx
sudo git init
sudo git add .
sudo git commit -m "Initial nginx config"
# After changes:
sudo git diff
sudo git commit -am "Update worker_processes"
```

### Use `screen` or `tmux` for Long Operations

```bash
# NEVER run long operations in a plain SSH session
# If disconnected, the operation stops

# Good:
tmux new -s deploy
./long-deploy-script.sh
# Ctrl+B d to detach
# Safe to disconnect — script keeps running
```

### System Change Checklist

Before any production change:
1. Know the rollback procedure
2. Have a backup
3. Test in staging first
4. Schedule a maintenance window (if user-facing)
5. Monitor after change
6. Document what you did

---

## 13.4 Logging and Monitoring Practices

### Application Logging Standard

```bash
# Log format: timestamp level message
# Example:
[2026-06-23 10:00:01] [INFO]  Starting deployment
[2026-06-23 10:00:05] [WARN]  Old backup found, skipping
[2026-06-23 10:00:10] [ERROR] Failed to connect to database

# Use structured logging (JSON) for machine parsing
{"timestamp": "2026-06-23T10:00:01Z", "level": "INFO", "message": "Started", "version": "1.5"}
```

### Log Rotation

```bash
# Configure logrotate for your application
cat > /etc/logrotate.d/myapp << 'EOF'
/var/log/myapp/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 644 myapp myapp
    postrotate
        systemctl reload myapp 2>/dev/null || true
    endscript
}
EOF
```

### System Monitoring Script

```bash
#!/bin/bash
# monitor.sh — Simple system health monitor

ALERT_THRESHOLD_CPU=90
ALERT_THRESHOLD_MEM=90
ALERT_THRESHOLD_DISK=85

check_cpu() {
    local cpu_idle
    cpu_idle=$(top -bn1 | grep "Cpu(s)" | awk '{print $8}' | tr -d '%id,')
    local cpu_used=$((100 - ${cpu_idle%.*}))
    if [[ $cpu_used -gt $ALERT_THRESHOLD_CPU ]]; then
        echo "ALERT: CPU usage at ${cpu_used}%"
    fi
}

check_memory() {
    local mem_used_pct
    mem_used_pct=$(free | awk 'NR==2{printf "%.0f", $3*100/$2}')
    if [[ $mem_used_pct -gt $ALERT_THRESHOLD_MEM ]]; then
        echo "ALERT: Memory usage at ${mem_used_pct}%"
    fi
}

check_disk() {
    while IFS= read -r line; do
        local usage filesystem
        usage=$(echo "$line" | awk '{print $5}' | tr -d '%')
        filesystem=$(echo "$line" | awk '{print $6}')
        if [[ $usage -gt $ALERT_THRESHOLD_DISK ]]; then
            echo "ALERT: Disk $filesystem at ${usage}%"
        fi
    done < <(df -h | tail -n +2)
}

check_services() {
    local services=("nginx" "docker" "sshd")
    for svc in "${services[@]}"; do
        if ! systemctl is-active --quiet "$svc"; then
            echo "ALERT: Service $svc is NOT running!"
        fi
    done
}

echo "=== Health Check: $(date) ==="
check_cpu
check_memory
check_disk
check_services
```

---

## 13.5 Performance Best Practices

```bash
# Check what's consuming resources before adding hardware
htop                               # visual process inspector
iotop                              # I/O per process
nethogs                            # network per process
ncdu /var                          # disk usage navigator (install: sudo apt install ncdu)

# Profile before optimizing
strace -c ./program               # count syscalls
time ./program                     # wall/user/sys time

# Use caching wisely
# Understand /proc/meminfo's "buff/cache" — it's GOOD (OS is caching disk)
# Only worry about "available" being too low

# I/O optimization
ionice -c 3 -p $$    # make current process use idle I/O class
ionice -c 2 -n 7 ./backup.sh   # background priority for backup
```

---

## Summary: Production Linux Habits

```
Security:
  ✓ Disable root SSH login
  ✓ Use SSH keys (not passwords)
  ✓ Run services as dedicated non-root users
  ✓ Firewall: default deny, explicit allow
  ✓ Secrets in files with 600 permissions

Scripts:
  ✓ set -euo pipefail at the top
  ✓ Quote all variables
  ✓ Validate inputs
  ✓ Log with timestamps
  ✓ Run ShellCheck

Operations:
  ✓ Backup before changing configs
  ✓ Use tmux for long-running operations
  ✓ Test in staging before production
  ✓ Monitor after changes
  ✓ Document what you did
```

---

## Knowledge Check

1. What three SSH settings should you always change on a new server?
2. Why should services run as dedicated non-root users?
3. What does `set -euo pipefail` do at the start of a script?
4. Why should you never run long operations in a plain SSH session?
5. What is the principle of least privilege?

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="12-advanced-concepts.md">← Previous: Advanced Linux Concepts</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="14-common-mistakes-and-pitfalls.md">Next: Common Mistakes & Pitfalls →</a>
</div>
