# Chapter 10 — Users & System Administration

## Learning Objectives

By the end of this chapter, you will:
- Manage users and groups on a Linux system
- Configure cron jobs for scheduled automation
- Manage services with systemd
- Work with system logs and `journald`
- Manage software packages with `apt` and `yum`
- Configure the system environment

## Prerequisites

- Chapter 09 — Shell Scripting

---

## 10.1 User Management Deep Dive

### The `/etc/passwd` File

```bash
cat /etc/passwd

# Format: username:x:UID:GID:GECOS:home:shell
# akash:x:1000:1000:Akash Gaur,,,:/home/akash:/bin/bash
#   │    │  │     │       │          │            │
#   │    │  │     │       │          │            └─ login shell
#   │    │  │     │       │          └─ home directory
#   │    │  │     │       └─ GECOS: full name, room, etc.
#   │    │  │     └─ primary group ID
#   │    │  └─ user ID
#   │    └─ password placeholder ('x' means in /etc/shadow)
#   └─ username
```

### UID Ranges

| UID Range | Type |
|-----------|------|
| 0 | root |
| 1–999 | System users (services: www-data, mysql, docker) |
| 1000+ | Regular users |
| 65534 | `nobody` (minimal permissions) |

```bash
# Find all regular users (UID >= 1000)
awk -F: '$3 >= 1000 && $3 < 65534 {print $1, $3}' /etc/passwd

# Find all system users
awk -F: '$3 < 1000 {print $1, $3}' /etc/passwd

# Find users with bash shell (can log in interactively)
grep "/bin/bash$" /etc/passwd | cut -d: -f1
```

### Creating Users

```bash
# Basic user creation
sudo useradd username

# Full user creation (home dir, bash shell, real name)
sudo useradd -m -s /bin/bash -c "Akash Gaur" akash

# Create system user (for services — no login shell, no home)
sudo useradd --system --no-create-home --shell /usr/sbin/nologin appuser

# Set password
sudo passwd akash

# Create user with specific UID and GID
sudo useradd -m -u 1500 -g 1500 deployer
```

### Modifying Users

```bash
sudo usermod -aG docker akash          # add to docker group
sudo usermod -aG sudo akash            # add to sudo group
sudo usermod -s /bin/zsh akash         # change shell
sudo usermod -d /home/new_home akash   # change home dir
sudo usermod -l newname oldname        # rename user
sudo usermod -L akash                  # lock account (disable login)
sudo usermod -U akash                  # unlock account
sudo usermod -e 2027-01-01 akash       # set account expiry
```

### Deleting Users

```bash
sudo userdel username          # delete user (keep home dir)
sudo userdel -r username       # delete user AND home directory
```

### Group Management

```bash
sudo groupadd developers       # create group
sudo groupadd -g 1500 deploy   # with specific GID
sudo gpasswd -a akash developers   # add user to group
sudo gpasswd -d akash developers   # remove user from group
sudo groupdel developers       # delete group

# View group members
getent group developers
grep "^developers:" /etc/group
```

### Service Account Best Practices

```bash
# Create a service account for your application
sudo useradd \
    --system \                    # system account (UID < 1000)
    --no-create-home \            # no home directory
    --shell /usr/sbin/nologin \   # cannot log in interactively
    --comment "MyApp Service" \
    myapp

# Assign ownership
sudo chown -R myapp:myapp /opt/myapp/
sudo chmod 750 /opt/myapp/
```

---

## 10.2 Package Management

### Ubuntu/Debian — `apt`

```bash
# Update package list (always do this first)
sudo apt update

# Upgrade installed packages
sudo apt upgrade
sudo apt full-upgrade      # also handles dependencies

# Install packages
sudo apt install nginx
sudo apt install docker.io python3 git vim

# Remove packages
sudo apt remove nginx      # remove but keep config
sudo apt purge nginx       # remove including config
sudo apt autoremove        # remove unused dependencies

# Search for packages
apt search nginx
apt-cache search python

# Show package info
apt show nginx
apt-cache show nginx

# List installed packages
dpkg -l
dpkg -l | grep nginx

# Check if package is installed
dpkg -l nginx 2>/dev/null | grep -q "^ii" && echo "Installed" || echo "Not installed"

# Find which package provides a file
dpkg -S /usr/bin/git
apt-file search git       # needs: sudo apt install apt-file
```

### RHEL/CentOS/Amazon Linux — `yum` / `dnf`

```bash
sudo yum update               # update all packages
sudo yum install nginx        # install
sudo yum remove nginx         # remove
sudo yum search nginx         # search
sudo yum info nginx           # info
yum list installed            # list installed

# dnf (modern yum, Fedora/RHEL 8+)
sudo dnf install nginx
sudo dnf update
sudo dnf remove nginx
```

### Adding PPAs and External Repositories

```bash
# Example: add Docker's official repository
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
```

---

## 10.3 Cron — Scheduled Jobs

Cron runs commands on a schedule. Essential for:
- Log rotation and cleanup
- Database backups
- Report generation
- Health checks
- Certificate renewal

### Crontab Syntax

```
┌───────────── minute (0–59)
│ ┌───────────── hour (0–23)
│ │ ┌───────────── day of month (1–31)
│ │ │ ┌───────────── month (1–12)
│ │ │ │ ┌───────────── day of week (0–7, 0 and 7 = Sunday)
│ │ │ │ │
* * * * *  command to execute
```

```bash
# Every minute
* * * * * /path/to/script.sh

# Every 5 minutes
*/5 * * * * /path/to/script.sh

# Every hour at minute 0
0 * * * * /path/to/script.sh

# Every day at 2:30 AM
30 2 * * * /path/to/backup.sh

# Every Sunday at 3 AM
0 3 * * 0 /path/to/weekly-cleanup.sh

# First day of every month at midnight
0 0 1 * * /path/to/monthly-report.sh

# Every weekday (Mon-Fri) at 9 AM
0 9 * * 1-5 /path/to/weekday-task.sh

# Multiple times: at 8 AM and 8 PM
0 8,20 * * * /path/to/script.sh

# Shorthand
@reboot    /path/to/script.sh   # run at system startup
@daily     /path/to/script.sh   # run once a day (= 0 0 * * *)
@weekly    /path/to/script.sh   # run once a week
@monthly   /path/to/script.sh   # run once a month
```

> Use [crontab.guru](https://crontab.guru/) to verify cron expressions.

### Managing Crontabs

```bash
crontab -e          # edit your crontab (opens editor)
crontab -l          # list your crontab
crontab -r          # REMOVE your entire crontab (careful!)
crontab -u akash -l # view another user's crontab (as root)

# System-wide cron (root)
sudo crontab -e

# Cron drop-in directories (scripts placed here run automatically)
ls /etc/cron.d/       # individual cron files
ls /etc/cron.daily/   # scripts run daily
ls /etc/cron.weekly/  # scripts run weekly
ls /etc/cron.monthly/ # scripts run monthly
```

### Cron Best Practices

```bash
# 1. Always use full paths in cron (PATH is not set)
0 2 * * * /usr/bin/python3 /opt/myapp/backup.py

# 2. Redirect output to log file
0 2 * * * /opt/backup.sh >> /var/log/backup.log 2>&1

# 3. Add timestamps in your scripts for log readability
echo "[$(date '+%Y-%m-%d %H:%M:%S')] Starting backup..."

# 4. Use locks to prevent overlapping runs
0 * * * * flock -n /tmp/myapp.lock /opt/myapp/job.sh

# 5. Test your cron command manually first
/opt/backup.sh   # run manually, verify it works
```

---

## 10.4 Systemd Deep Dive

Systemd is the init system on modern Linux. It manages the entire lifecycle of services.

### Service Units

```bash
# Service unit locations (in priority order)
/etc/systemd/system/        # admin-created (highest priority)
/run/systemd/system/        # runtime units
/lib/systemd/system/        # package-installed
/usr/lib/systemd/system/    # package-installed (RHEL)

# View a service unit file
systemctl cat nginx
cat /lib/systemd/system/nginx.service
```

### Unit File Anatomy

```ini
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=network.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t
ExecStart=/usr/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

### Service Types

| Type | Meaning |
|------|---------|
| `simple` | Main process is the service (default) |
| `forking` | Service forks to background (traditional daemons) |
| `oneshot` | Runs once and exits (e.g., setup scripts) |
| `notify` | Like simple, but process signals readiness |
| `idle` | Start after other jobs finish |

### Writing a Custom Service

```bash
sudo cat > /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Python Application
Documentation=https://myapp.example.com
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp

# Environment
Environment="APP_ENV=production"
Environment="APP_PORT=8080"
EnvironmentFile=/opt/myapp/.env

# Commands
ExecStart=/opt/myapp/venv/bin/python -m myapp
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID

# Restart policy
Restart=on-failure
RestartSec=5s
StartLimitBurst=3
StartLimitIntervalSec=60

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/opt/myapp /var/log/myapp

# Resource limits
LimitNOFILE=65536
MemoryLimit=512M

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp
systemctl status myapp
```

### Systemd Targets (Runlevels)

```bash
systemctl get-default               # current default target
sudo systemctl set-default multi-user.target  # boot to CLI (no GUI)
sudo systemctl set-default graphical.target   # boot to GUI

# Runlevel equivalents
# 0 = poweroff.target
# 1 = rescue.target
# 3 = multi-user.target (server mode)
# 5 = graphical.target
# 6 = reboot.target
```

---

## 10.5 System Logs with `journald`

systemd includes `journald` for centralized log management. All service output goes here.

```bash
# View all logs
journalctl

# View logs for a specific service
journalctl -u nginx
journalctl -u docker
journalctl -u myapp

# Follow logs in real time
journalctl -u nginx -f

# Last N lines
journalctl -u nginx -n 100

# Time-based filtering
journalctl --since "1 hour ago"
journalctl --since "2026-06-23 10:00:00"
journalctl --since "2026-06-23" --until "2026-06-24"

# Priority filtering
journalctl -p err              # errors only
journalctl -p warning          # warnings and above
journalctl -u nginx -p err     # nginx errors

# Boot logs
journalctl -b                  # current boot
journalctl -b -1               # previous boot

# Disk usage
journalctl --disk-usage

# Vacuum old logs
sudo journalctl --vacuum-size=500M    # keep 500MB
sudo journalctl --vacuum-time=7d      # keep 7 days
```

### Traditional Log Files (still exist alongside journald)

```bash
# On Ubuntu:
/var/log/syslog          # general system
/var/log/auth.log        # authentication
/var/log/kern.log        # kernel
/var/log/dpkg.log        # package installs

# View with traditional tools
tail -f /var/log/syslog
grep "error" /var/log/syslog
```

### `logrotate` — Managing Log File Size

```bash
# logrotate config example
cat /etc/logrotate.d/nginx
# /var/log/nginx/*.log {
#     daily
#     missingok
#     rotate 52
#     compress
#     delaycompress
#     notifempty
#     create 640 www-data adm
#     sharedscripts
#     postrotate
#         nginx -s reopen
#     endscript
# }

# Test logrotate config
sudo logrotate --debug /etc/logrotate.d/nginx

# Force rotation
sudo logrotate --force /etc/logrotate.conf
```

---

## 10.6 Environment Configuration

### `/etc/environment` — System-Wide Variables

```bash
cat /etc/environment
# PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# Add a system-wide variable
echo 'JAVA_HOME="/opt/java-17"' | sudo tee -a /etc/environment
```

### Shell Initialization Files

| File | When Loaded |
|------|------------|
| `/etc/profile` | Login shells, all users |
| `/etc/profile.d/*.sh` | Login shells (drop-in scripts) |
| `/etc/bash.bashrc` | Interactive non-login bash, all users |
| `~/.bash_profile` | Login shells, specific user |
| `~/.bashrc` | Interactive non-login shells, specific user |
| `~/.bash_logout` | On logout |

```bash
# Add to your .bashrc (applies every new terminal)
cat >> ~/.bashrc << 'EOF'

# Custom aliases
alias ll='ls -la'
alias gs='git status'
alias dc='docker-compose'

# Custom PATH
export PATH="$HOME/.local/bin:$PATH"

# Custom environment
export EDITOR=vim
export HISTSIZE=10000
export HISTFILESIZE=20000
EOF

# Apply without logging out
source ~/.bashrc
# Or:
. ~/.bashrc
```

---

## Summary

| Task | Command |
|------|---------|
| Create user | `sudo useradd -m -s /bin/bash username` |
| Add to group | `sudo usermod -aG groupname username` |
| Install package | `sudo apt install package` |
| Edit crontab | `crontab -e` |
| Service status | `systemctl status service` |
| Service logs | `journalctl -u service -f` |
| Reload systemd | `sudo systemctl daemon-reload` |

---

## Knowledge Check

1. What UID range is used for regular users?
2. How do you add a user to the `docker` group?
3. Write a cron expression that runs at 2:30 AM every Sunday.
4. What does `set -euo pipefail` have to do with systemd? (Trick question — explain each)
5. How do you view the last 50 log lines for the nginx service?
6. What's the difference between `apt remove` and `apt purge`?

---

## Hands-On Exercise

```bash
# 1. Create a service account
sudo useradd --system --no-create-home --shell /usr/sbin/nologin webworker
id webworker

# 2. Set up a cron job
crontab -e
# Add: */5 * * * * echo "[$(date)] health check" >> /tmp/health.log
# Wait a few minutes, then:
tail /tmp/health.log

# Remove test crontab entry when done:
crontab -l | grep -v "health check" | crontab -

# 3. Install and configure nginx
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
systemctl status nginx

# 4. View nginx logs via journald
journalctl -u nginx -n 20

# 5. Check service logs for errors
journalctl -u nginx -p err --since "1 hour ago"

# 6. List all running services
systemctl list-units --type=service --state=running | head -20
```

---

## Further Reading

- `man systemctl`, `man journalctl`, `man crontab`
- [systemd documentation](https://systemd.io/)
- [The Linux Command Line](http://linuxcommand.org/tlcl.php) — Chapters 9-10

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="09-shell-scripting.md">← Previous: Shell Scripting</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="11-intermediate-concepts.md">Next: Pipes, Redirects & Environment →</a>
</div>
