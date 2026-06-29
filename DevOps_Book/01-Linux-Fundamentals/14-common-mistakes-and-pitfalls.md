# Chapter 14 — Common Mistakes & Pitfalls

## Learning Objectives

By the end of this chapter, you will:
- Know the most dangerous Linux mistakes and how to avoid them
- Debug common issues you'll encounter in production
- Understand why certain operations fail and how to fix them
- Build defensive habits that prevent catastrophes

## Prerequisites

- All previous chapters

---

## 14.1 The Deadly Commands

### `rm -rf` Without Thinking

**The Incident:**
```bash
# Wanted to delete: /var/log/myapp/
# Accidentally ran:
rm -rf /var/log/myapp /         # space between path and /

# Or the classic:
rm -rf $DIRECTORY/              # if $DIRECTORY is empty, this becomes rm -rf /
```

**Prevention:**
```bash
# 1. Always echo first
echo rm -rf "${DIRECTORY}/"

# 2. Use a variable check
[[ -n "$DIRECTORY" ]] && rm -rf "${DIRECTORY}/"

# 3. Use trash instead of rm
sudo apt install trash-cli
trash-put file.txt             # goes to ~/.local/share/Trash/

# 4. Restrict rm in your .bashrc
alias rm='rm -I'               # -I: ask before removing 3+ files

# 5. Use rmdir for directories you expect to be empty
rmdir directory/               # fails if not empty (safe!)
```

### `chmod -R 777`

```bash
# This is almost always wrong
sudo chmod -R 777 /var/www/html

# What went wrong: now anyone can modify your web files
# Security implications: script injection, data theft

# Correct approach:
find /var/www/html -type d -exec chmod 755 {} \;
find /var/www/html -type f -exec chmod 644 {} \;
chown -R www-data:www-data /var/www/html
```

### Running Scripts from the Internet Without Reading Them

```bash
# NEVER do this
curl https://sketchy-site.com/install.sh | sudo bash

# DO this instead
curl -o install.sh https://trusted-site.com/install.sh
cat install.sh          # read it first!
chmod +x install.sh
./install.sh            # or sudo ./install.sh
```

---

## 14.2 Permissions Mistakes

### "Permission Denied" Debugging

```bash
# Step 1: Check the actual permissions
ls -la problematic-file
ls -la containing-directory

# Step 2: Check who you are
id
whoami

# Step 3: Check if parent directory allows access
# To access /var/app/config.yaml, you need:
# - execute on /
# - execute on /var
# - execute on /var/app
# - read on /var/app/config.yaml
ls -la /var/app/   # check parent dir

# Common fix: forgot to set directory execute permission
chmod +x /opt/myapp/           # need x to enter directory
chmod 755 /opt/myapp/          # standard directory permission
```

### Wrong Ownership After Deployment

```bash
# Problem: files deployed as root, but app runs as www-data
ls -la /var/www/html/
# -rw-r--r-- root root app.php

# Fix:
sudo chown -R www-data:www-data /var/www/html/
```

### SSH Key Permission Issues

```bash
# SSH refuses to use keys with wrong permissions
# Error: "WARNING: UNPROTECTED PRIVATE KEY FILE!"
chmod 700 ~/.ssh/
chmod 600 ~/.ssh/id_rsa         # private key: owner read/write only
chmod 644 ~/.ssh/id_rsa.pub     # public key: readable by all
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/config
```

---

## 14.3 Process and Service Mistakes

### Killing the Wrong Process

```bash
# Wrong: pkill kills ALL matching processes
pkill python              # kills ALL python processes!

# Right: find the specific PID first
pgrep -a python           # see all matching PIDs with full command
kill 12345                # kill specific PID

# Or use more specific matching
pkill -f "python3 myapp.py"    # match full command line
```

### Service Won't Start — Debugging

```bash
# Step 1: Check status
sudo systemctl status myapp

# Step 2: Check logs
journalctl -u myapp -n 50 --no-pager

# Step 3: Check if port is in use
ss -tlnp | grep :8080

# Step 4: Check config syntax
nginx -t
apache2ctl configtest

# Step 5: Run manually to see real error
sudo -u www-data /usr/bin/nginx    # run as service user
sudo -u myapp /opt/myapp/start.sh  # see actual error

# Step 6: Check file permissions
ls -la /var/log/myapp/          # can the service write logs?
ls -la /opt/myapp/              # does service user own app dir?
```

### Zombie Processes

```bash
# Find zombies
ps aux | awk '$8 == "Z" { print }'

# Cause: parent process not calling wait() on children
# Fix: you can't kill a zombie (it's already dead)
# Solution: kill the parent process, which adopts and cleans up zombies

# Find zombie's parent
ps -o pid,ppid,stat,cmd aux | awk '$3 ~ /Z/ {print "zombie:", $1, "parent:", $2}'
kill -SIGCHLD <parent_pid>    # signal parent to collect exit status
```

---

## 14.4 Disk Space Issues

### Disk Full — Finding the Culprit

```bash
# System reports no space left
# Step 1: Check disk usage
df -h

# Step 2: Find the big directories
du -sh /var/* 2>/dev/null | sort -rh | head -10
du -sh /home/* 2>/dev/null | sort -rh
du -sh /tmp/* 2>/dev/null | sort -rh

# Step 3: Find large files
find / -xdev -size +500M -type f 2>/dev/null | sort -k5 -rn

# Common culprits:
# /var/log growing unboundedly (forgot logrotate)
# /var/lib/docker (dangling Docker images/volumes)
# /tmp not cleared (system setting)
# Core dumps in /var/core or /tmp
# Old kernels in /boot

# Quick fixes:
sudo journalctl --vacuum-size=500M     # trim journal logs
sudo apt autoremove --purge            # remove old kernels
docker system prune -af               # remove Docker junk (careful!)
find /tmp -mtime +7 -delete           # clear old temp files
```

### Running Out of Inodes

```bash
# You can run out of inodes even with disk space free
df -i         # check inode usage

# Common cause: millions of tiny files
# Find directories with many files:
find / -xdev -printf '%h\n' 2>/dev/null | sort | uniq -c | sort -rn | head -20
```

---

## 14.5 Scripting Mistakes

### Unquoted Variables (Word Splitting)

```bash
# Problem:
file="my file with spaces.txt"
rm $file        # runs: rm my file with spaces.txt (removes 'my', 'file', etc.)

# Fix:
rm "$file"      # runs: rm "my file with spaces.txt"

# Also matters in conditionals:
if [ $var = "value" ]; then  # breaks if var is empty or has spaces
if [[ "$var" = "value" ]]; then  # correct
```

### Forgetting `set -e` (Ignoring Errors)

```bash
# Without set -e:
mkdir /new-dir
cp important-file /new-dir/      # if this fails, script continues
chmod 644 /new-dir/important-file  # operates on nothing!

# With set -euo pipefail:
# Script exits immediately when cp fails
set -euo pipefail
```

### Off-By-One in Loops

```bash
# Wrong: creates 0.txt through 9.txt (10 files, not 10)
for i in $(seq 0 9); do ...

# Correct for "exactly 10 iterations":
for i in $(seq 1 10); do ...
for ((i=0; i<10; i++)); do ...
```

### Pipe in Condition (Exit Code Lost Without pipefail)

```bash
# Without set -o pipefail:
cat /nonexistent | grep "something"
echo $?    # prints 0 (grep's exit code), not 1 (cat's exit code)

# With set -o pipefail:
set -o pipefail
cat /nonexistent | grep "something"
echo $?    # prints 1 (pipeline fails if any command fails)
```

---

## 14.6 Networking Mistakes

### Service Listening on Wrong Interface

```bash
# Service starts but can't be reached externally
# Check what interface it's listening on:
ss -tlnp

# 127.0.0.1:8080 — only localhost (not reachable externally)
# 0.0.0.0:8080   — all interfaces (reachable externally)
# :::8080         — all IPv4 and IPv6 interfaces

# Fix in app config: change bind address
# For nginx: listen 0.0.0.0:80;   not   listen 127.0.0.1:80;
```

### Forgot to Open Firewall

```bash
# Service works locally but not from outside
# Check firewall:
sudo ufw status
sudo iptables -L INPUT

# Fix:
sudo ufw allow 8080
```

### DNS Not Resolving

```bash
# "Could not resolve host"
# Check DNS config:
cat /etc/resolv.conf
dig google.com

# Test with IP to verify it's DNS (not network):
curl http://8.8.8.8    # if this works but domain doesn't → DNS problem

# Fix: use reliable DNS servers
sudo vim /etc/resolv.conf
# nameserver 8.8.8.8
# nameserver 1.1.1.1
```

---

## 14.7 The "Works on My Machine" Trap

Common causes and fixes:

```bash
# 1. PATH differences between interactive and cron
# Fix: always use full paths in cron and scripts
/usr/bin/python3 not python3
/usr/local/bin/kubectl not kubectl

# 2. Environment variables not set
# Fix: explicitly export in script, don't rely on parent environment
export APP_ENV=production
# Or use .env file
set -a; source /opt/myapp/.env; set +a

# 3. Different user context
# Test as the actual service user:
sudo -u www-data ./script.sh
sudo -u myapp bash -c 'cd /opt/myapp && ./start.sh'

# 4. File not found due to relative path
# Fix: use absolute paths or SCRIPT_DIR variable
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
CONFIG_FILE="${SCRIPT_DIR}/config.yaml"
```

---

## 14.8 Quick Diagnosis Checklist

When something is broken, run through this:

```bash
# 1. Check the service status
systemctl status <service>

# 2. Check the logs
journalctl -u <service> -n 50
tail -50 /var/log/<service>/error.log

# 3. Check if listening on correct port/interface
ss -tlnp | grep :<port>

# 4. Check if reachable locally
curl http://localhost:<port>

# 5. Check firewall
sudo ufw status
sudo iptables -L

# 6. Check file permissions
ls -la <config-file>
ls -la <data-directory>

# 7. Check disk space
df -h
df -i    # inodes

# 8. Check memory
free -h

# 9. Check CPU/load
uptime
top

# 10. Check for error in config
nginx -t    # test nginx config
sshd -t     # test sshd config
```

---

## Summary

**Top 10 Linux Mistakes to Avoid:**

1. Running `rm -rf` without verifying the path
2. Using `chmod -R 777` as a "fix"
3. Running scripts from the internet without reading them
4. Not quoting variables in shell scripts
5. Forgetting `set -euo pipefail` in scripts
6. Running services as root when not needed
7. Storing secrets in plain environment variables
8. Not testing SSH config before restarting sshd
9. Running long operations in plain SSH (use tmux)
10. Skipping staging and deploying direct to production

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="13-best-practices.md">← Previous: Linux Best Practices</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="15-projects.md">Next: Hands-On Projects →</a>
</div>
