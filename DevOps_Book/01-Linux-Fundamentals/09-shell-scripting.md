# Chapter 09 — Shell Scripting

## Learning Objectives

By the end of this chapter, you will:
- Write shell scripts for DevOps automation
- Use variables, arrays, and special variables
- Write conditionals and loops
- Create functions and handle arguments
- Use exit codes and error handling
- Process command output in scripts
- Write production-quality scripts with best practices

## Prerequisites

- Chapters 01-08 (all previous chapters)

---

## 9.1 Why Shell Scripting?

Every DevOps task involves repetition:
- Deploy a new version (stop service → backup → update code → restart)
- Check disk space across 20 servers
- Parse logs and send alerts
- Set up a new server (install packages → configure → start services)

Shell scripts automate these sequences. Unlike Python or Go, shell scripts require no installation and work on any Linux system.

---

## 9.2 Your First Script

```bash
# Create the script
cat > ~/hello.sh << 'EOF'
#!/bin/bash
echo "Hello, DevOps World!"
echo "Today is: $(date)"
echo "You are: $(whoami)"
echo "On host: $(hostname)"
EOF

# Make it executable
chmod +x ~/hello.sh

# Run it
./hello.sh
```

### The Shebang `#!/bin/bash`

The first line `#!/bin/bash` (shebang/hashbang) tells the OS which interpreter to use. Always include it.

```bash
#!/bin/bash     # use bash
#!/bin/sh       # use POSIX sh (more portable, fewer features)
#!/usr/bin/env bash   # find bash anywhere in PATH (better portability)
```

> **Best practice:** Always use `#!/bin/bash` for DevOps scripts. Use `#!/bin/sh` only when you need maximum portability to systems without bash.

---

## 9.3 Variables

### Defining and Using Variables

```bash
#!/bin/bash

# Define variables (NO spaces around =)
name="Akash"
version=1.5
app_dir="/opt/myapp"
current_date=$(date +%Y-%m-%d)    # capture command output

# Use variables with $
echo "Name: $name"
echo "Version: $version"
echo "Directory: $app_dir"
echo "Date: $current_date"

# Curly braces for clarity
echo "Backup: ${app_dir}_backup"   # outputs: /opt/myapp_backup
echo "Version: ${version}+1"       # outputs: 1.5+1 (not arithmetic, just concatenation)
```

### Special Variables

| Variable | Meaning |
|----------|---------|
| `$0` | Script name |
| `$1`, `$2`, ... | Command-line arguments (positional parameters) |
| `$#` | Number of arguments |
| `$@` | All arguments (as separate words) |
| `$*` | All arguments (as one string) |
| `$?` | Exit code of last command |
| `$$` | PID of current script |
| `$!` | PID of last background command |
| `$_` | Last argument of previous command |
| `$USER` | Current username (environment variable) |
| `$HOME` | Home directory |
| `$PWD` | Current directory |
| `$PATH` | Executable search path |

```bash
#!/bin/bash
echo "Script name: $0"
echo "First arg: $1"
echo "All args: $@"
echo "Arg count: $#"
echo "My user: $USER"
echo "My home: $HOME"
```

### Variable Best Practices

```bash
# Quote variables to handle spaces and special characters
filename="my file with spaces.txt"
cat "$filename"        # CORRECT
cat $filename          # WRONG: breaks on spaces

# Read-only variables
readonly MAX_RETRIES=3

# Local variables in functions (covered later)
local my_var="local value"

# Default values
name="${1:-default_user}"        # use $1, or "default_user" if $1 is empty
port="${PORT:-8080}"             # use $PORT env var, or 8080
```

---

## 9.4 Input and Output

```bash
#!/bin/bash

# Read user input
read -p "Enter your name: " name
echo "Hello, $name!"

# Read with timeout
read -t 10 -p "Enter within 10 seconds: " input

# Read password (hidden input)
read -s -p "Password: " password
echo    # new line after hidden input

# Read from file
while IFS= read -r line; do
    echo "Line: $line"
done < file.txt

# Here document
cat << EOF
This is line 1
This is line 2 with var: $name
EOF

# Here string
grep "pattern" <<< "string to search in"
```

### Output Redirection

```bash
echo "message" > file.txt       # write (overwrite)
echo "message" >> file.txt      # append
echo "error" >&2                # write to stderr
command 2>/dev/null             # suppress stderr
command > /dev/null 2>&1        # suppress all output
command 2>&1 | tee logfile.txt  # send to both file and terminal
```

---

## 9.5 Conditionals

### `if` Statement

```bash
#!/bin/bash

# Basic if
if [ condition ]; then
    # commands
elif [ other_condition ]; then
    # commands
else
    # commands
fi

# Example
age=25
if [ $age -ge 18 ]; then
    echo "Adult"
else
    echo "Minor"
fi
```

### Comparison Operators

#### Numeric Comparisons

| Operator | Meaning |
|----------|---------|
| `-eq` | Equal |
| `-ne` | Not equal |
| `-lt` | Less than |
| `-le` | Less than or equal |
| `-gt` | Greater than |
| `-ge` | Greater than or equal |

#### String Comparisons

| Operator | Meaning |
|----------|---------|
| `=` or `==` | Equal |
| `!=` | Not equal |
| `-z` | Empty string |
| `-n` | Non-empty string |
| `<` | Less than (lexicographic, inside `[[ ]]`) |
| `>` | Greater than (lexicographic) |

#### File Tests

| Operator | Meaning |
|----------|---------|
| `-f file` | Exists and is a regular file |
| `-d dir` | Exists and is a directory |
| `-e path` | Exists (any type) |
| `-r file` | Exists and is readable |
| `-w file` | Exists and is writable |
| `-x file` | Exists and is executable |
| `-s file` | Exists and has size > 0 |
| `-L file` | Is a symbolic link |

### `[ ]` vs `[[ ]]`

```bash
# [ ] is POSIX, works everywhere
if [ "$name" = "akash" ]; then ...

# [[ ]] is bash-specific, more powerful
if [[ "$name" == "akash" ]]; then ...
if [[ "$name" =~ ^[A-Z] ]]; then ...   # regex support
if [[ -f "$file" && -r "$file" ]]; then ...  # && without quotes
```

> **Use `[[ ]]` in bash scripts** — it's safer (no word splitting on variables) and more powerful.

### Practical Conditionals

```bash
#!/bin/bash

# Check if file exists
if [[ -f "/etc/nginx/nginx.conf" ]]; then
    echo "Nginx is configured"
fi

# Check if directory exists, create if not
if [[ ! -d "/opt/myapp/logs" ]]; then
    mkdir -p /opt/myapp/logs
    echo "Created logs directory"
fi

# Check if command exists
if command -v docker &>/dev/null; then
    echo "Docker is installed: $(docker --version)"
else
    echo "Docker is NOT installed"
fi

# Check disk usage
USAGE=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
if [[ $USAGE -gt 90 ]]; then
    echo "CRITICAL: Disk usage at ${USAGE}%"
elif [[ $USAGE -gt 80 ]]; then
    echo "WARNING: Disk usage at ${USAGE}%"
else
    echo "OK: Disk usage at ${USAGE}%"
fi

# Check if service is running
if systemctl is-active --quiet nginx; then
    echo "Nginx is running"
else
    echo "Nginx is NOT running - starting..."
    sudo systemctl start nginx
fi
```

### `case` Statement

```bash
#!/bin/bash
action="$1"

case "$action" in
    start)
        echo "Starting service..."
        ;;
    stop)
        echo "Stopping service..."
        ;;
    restart)
        echo "Restarting service..."
        ;;
    status)
        echo "Checking status..."
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
        ;;
esac
```

---

## 9.6 Loops

### `for` Loop

```bash
#!/bin/bash

# Loop over list
for server in web1 web2 web3; do
    echo "Checking $server..."
    ping -c 1 "$server" &>/dev/null && echo "$server: UP" || echo "$server: DOWN"
done

# Loop over files
for file in /var/log/*.log; do
    size=$(du -sh "$file" | cut -f1)
    echo "$size  $file"
done

# C-style for loop
for ((i=1; i<=5; i++)); do
    echo "Iteration $i"
done

# Loop over command output
for user in $(cut -d: -f1 /etc/passwd | head -5); do
    echo "User: $user"
done

# Loop over array
servers=("web1" "web2" "db1")
for server in "${servers[@]}"; do
    echo "Server: $server"
done
```

### `while` Loop

```bash
#!/bin/bash

# Count down
count=5
while [[ $count -gt 0 ]]; do
    echo "$count..."
    ((count--))
    sleep 1
done
echo "Done!"

# Read file line by line
while IFS= read -r line; do
    echo "Processing: $line"
done < /etc/hosts

# Retry loop
max_attempts=3
attempt=0
while [[ $attempt -lt $max_attempts ]]; do
    if curl -sf http://localhost:8080/health; then
        echo "Service is up!"
        break
    fi
    ((attempt++))
    echo "Attempt $attempt failed, retrying..."
    sleep 5
done

if [[ $attempt -eq $max_attempts ]]; then
    echo "Service did not start after $max_attempts attempts"
    exit 1
fi

# Infinite loop (use carefully with break)
while true; do
    if check_condition; then
        break
    fi
    sleep 10
done
```

### `until` Loop

```bash
# Loop UNTIL condition is true (opposite of while)
until systemctl is-active --quiet nginx; do
    echo "Waiting for nginx to start..."
    sleep 2
done
echo "Nginx is up!"
```

### `break` and `continue`

```bash
for i in {1..10}; do
    if [[ $i -eq 5 ]]; then
        continue    # skip 5
    fi
    if [[ $i -eq 8 ]]; then
        break       # stop at 8
    fi
    echo $i
done
# Output: 1 2 3 4 6 7
```

---

## 9.7 Functions

```bash
#!/bin/bash

# Define function
greet() {
    local name="$1"        # local: only accessible inside function
    echo "Hello, $name!"
}

# Call function
greet "Akash"
greet "DevOps"

# Function with return value (return codes 0-255)
is_running() {
    local service="$1"
    systemctl is-active --quiet "$service"
    return $?    # 0 = success, non-zero = failure
}

if is_running "nginx"; then
    echo "Nginx is running"
fi

# Function returning data via echo
get_disk_usage() {
    df / | awk 'NR==2 {print $5}' | tr -d '%'
}

usage=$(get_disk_usage)
echo "Disk usage: ${usage}%"
```

### Practical Function Library

```bash
#!/bin/bash

# ─── Logging Functions ───────────────────────────────────────────
log_info()    { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO]  $*"; }
log_warn()    { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [WARN]  $*" >&2; }
log_error()   { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [ERROR] $*" >&2; }

# ─── Utility Functions ───────────────────────────────────────────
require_command() {
    if ! command -v "$1" &>/dev/null; then
        log_error "Required command not found: $1"
        exit 1
    fi
}

check_root() {
    if [[ $EUID -ne 0 ]]; then
        log_error "This script must be run as root"
        exit 1
    fi
}

# Usage
log_info "Starting deployment"
require_command "docker"
require_command "kubectl"
log_info "All dependencies satisfied"
```

---

## 9.8 Exit Codes and Error Handling

### Exit Codes

```bash
exit 0      # success
exit 1      # general error
exit 2      # misuse of shell command
exit 127    # command not found

# Every command returns an exit code
ls /nonexistent
echo $?     # 2 (ls error)

true
echo $?     # 0 (success)

false
echo $?     # 1 (failure)
```

### Error Handling Best Practices

```bash
#!/bin/bash
set -e          # exit on any error
set -u          # exit on undefined variable
set -o pipefail # exit if any command in pipe fails
# Shorthand: set -euo pipefail

# Or at the top of every production script:
set -euo pipefail

# Error trap: run a function when script exits
cleanup() {
    echo "Cleaning up..."
    rm -f /tmp/deploy_lock
}
trap cleanup EXIT          # run cleanup on any exit

# Trap errors specifically
error_handler() {
    local line_no="$1"
    log_error "Script failed at line $line_no"
    # Add notification here (Slack, email, etc.)
}
trap 'error_handler $LINENO' ERR

# Check command success explicitly
if ! command; then
    echo "Command failed"
    exit 1
fi

# OR check exit code
command
if [[ $? -ne 0 ]]; then
    echo "Command failed with exit code $?"
    exit 1
fi
```

### `||` and `&&` in Scripts

```bash
# Run second command only if first FAILS
mkdir /opt/myapp || { echo "Failed to create directory"; exit 1; }

# Run second command only if first SUCCEEDS
curl -sf http://api/health && echo "API is healthy"

# Chain: stop on failure
update_code && restart_service && run_health_check || rollback
```

---

## 9.9 Arrays

```bash
#!/bin/bash

# Define array
servers=("web1" "web2" "web3" "db1")
files=()     # empty array

# Access elements
echo "${servers[0]}"        # first element: web1
echo "${servers[-1]}"       # last element: db1
echo "${servers[@]}"        # all elements
echo "${#servers[@]}"       # count: 4

# Add elements
servers+=("db2")
files+=("/var/log/app.log")

# Loop over array
for server in "${servers[@]}"; do
    echo "Checking: $server"
done

# Array slicing
echo "${servers[@]:1:2}"    # elements 1 and 2: web2 web3

# Check if element exists
needle="web2"
found=false
for item in "${servers[@]}"; do
    if [[ "$item" == "$needle" ]]; then
        found=true
        break
    fi
done
```

---

## 9.10 String Operations

```bash
#!/bin/bash

str="Hello World DevOps"

# Length
echo "${#str}"              # 19

# Substring
echo "${str:6}"             # World DevOps
echo "${str:6:5}"           # World

# Uppercase/lowercase
echo "${str,,}"             # hello world devops
echo "${str^^}"             # HELLO WORLD DEVOPS

# Replace
echo "${str/World/Linux}"   # Hello Linux DevOps (first match)
echo "${str//o/0}"          # Hell0 W0rld Dev0ps (all matches)

# Remove prefix/suffix
file="archive.tar.gz"
echo "${file%.gz}"          # archive.tar (remove shortest .gz suffix)
echo "${file%.*}"           # archive.tar (remove extension)
echo "${file%%.*}"          # archive (remove all after first dot)

path="/home/akash/scripts/deploy.sh"
echo "${path##*/}"          # deploy.sh (filename only)
echo "${path%/*}"           # /home/akash/scripts (directory only)

# Default value
echo "${UNDEFINED_VAR:-default}"    # prints: default
echo "${REQUIRED_VAR:?Error: REQUIRED_VAR not set}"  # exits with error if unset
```

---

## 9.11 A Real-World Deployment Script

```bash
#!/bin/bash
# deploy.sh — Deploy application to server
set -euo pipefail

# ─── Configuration ──────────────────────────────────────────────
APP_NAME="myapp"
APP_DIR="/opt/${APP_NAME}"
BACKUP_DIR="/opt/backups/${APP_NAME}"
LOG_FILE="/var/log/${APP_NAME}/deploy.log"
SERVICE_NAME="${APP_NAME}"
DEPLOY_USER="deploy"
REQUIRED_COMMANDS=("git" "systemctl")

# ─── Logging ────────────────────────────────────────────────────
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }
die() { log "ERROR: $*"; exit 1; }

# ─── Cleanup on exit ────────────────────────────────────────────
cleanup() {
    local exit_code=$?
    if [[ $exit_code -ne 0 ]]; then
        log "Deploy failed with exit code $exit_code"
        # Attempt rollback
        if [[ -d "${BACKUP_DIR}/latest" ]]; then
            log "Rolling back..."
            cp -r "${BACKUP_DIR}/latest/." "${APP_DIR}/"
            systemctl restart "$SERVICE_NAME" 2>/dev/null || true
        fi
    fi
}
trap cleanup EXIT

# ─── Validation ─────────────────────────────────────────────────
for cmd in "${REQUIRED_COMMANDS[@]}"; do
    command -v "$cmd" &>/dev/null || die "Required command not found: $cmd"
done

[[ $EUID -eq 0 ]] || die "Must run as root"
[[ -d "$APP_DIR" ]] || die "App directory not found: $APP_DIR"

# ─── Backup ─────────────────────────────────────────────────────
log "Creating backup..."
mkdir -p "$BACKUP_DIR"
cp -r "$APP_DIR" "${BACKUP_DIR}/$(date +%Y%m%d_%H%M%S)"
ln -sfn "${BACKUP_DIR}/$(ls -t "$BACKUP_DIR" | head -1)" "${BACKUP_DIR}/latest"

# ─── Deploy ─────────────────────────────────────────────────────
log "Deploying ${APP_NAME}..."
cd "$APP_DIR"
git pull origin main
# pip install -r requirements.txt  # if Python app

# ─── Restart ────────────────────────────────────────────────────
log "Restarting service..."
systemctl restart "$SERVICE_NAME"

# ─── Health Check ───────────────────────────────────────────────
log "Running health check..."
max_attempts=10
for ((i=1; i<=max_attempts; i++)); do
    if curl -sf http://localhost:8080/health &>/dev/null; then
        log "Health check passed"
        break
    fi
    if [[ $i -eq $max_attempts ]]; then
        die "Health check failed after $max_attempts attempts"
    fi
    log "Attempt $i/$max_attempts — waiting..."
    sleep 3
done

log "Deploy completed successfully!"
```

---

## Summary

```bash
# Script template
#!/bin/bash
set -euo pipefail

# Variables
VARIABLE="value"

# Functions
my_function() {
    local arg="$1"
    echo "Processing: $arg"
}

# Main logic
if [[ $# -lt 1 ]]; then
    echo "Usage: $0 <argument>"
    exit 1
fi

my_function "$1"
```

---

## Knowledge Check

1. What does `set -euo pipefail` do?
2. What is the difference between `$@` and `$*`?
3. How do you make a variable local to a function?
4. What does `${variable:-default}` do?
5. How do you loop over all lines in a file?

---

## Hands-On Exercise

```bash
# Exercise 1: Argument validation script
cat > ~/validate.sh << 'EOF'
#!/bin/bash
set -euo pipefail

if [[ $# -ne 2 ]]; then
    echo "Usage: $0 <name> <age>"
    exit 1
fi

name="$1"
age="$2"

if [[ ! "$age" =~ ^[0-9]+$ ]]; then
    echo "Error: age must be a number"
    exit 1
fi

echo "Name: $name, Age: $age"
if [[ $age -ge 18 ]]; then
    echo "$name is an adult"
else
    echo "$name is a minor"
fi
EOF
chmod +x ~/validate.sh
~/validate.sh Akash 25
~/validate.sh Test abc   # should error

# Exercise 2: System health check script
cat > ~/health-check.sh << 'EOF'
#!/bin/bash
set -euo pipefail

echo "=== System Health Report ==="
echo "Date: $(date)"
echo ""

# CPU load
echo "--- CPU ---"
uptime | awk '{print "Load Average: " $(NF-2) " " $(NF-1) " " $NF}'

# Memory
echo ""
echo "--- Memory ---"
free -h | awk 'NR==2 {printf "Total: %s, Used: %s, Free: %s\n", $2, $3, $4}'

# Disk
echo ""
echo "--- Disk Usage ---"
df -h / | awk 'NR==2 {printf "Root: %s used of %s (%s)\n", $3, $2, $5}'

# Top processes by CPU
echo ""
echo "--- Top 3 Processes (CPU) ---"
ps aux --sort=-%cpu | awk 'NR>1 && NR<=4 {printf "%.1f%% CPU  %s\n", $3, $11}'

echo ""
echo "==========================="
EOF
chmod +x ~/health-check.sh
~/health-check.sh
```

**Challenge:** Write a script that monitors disk usage on `/` and sends an alert (just `echo "ALERT"` for now) if it exceeds 80%. Run it in a loop every 30 seconds. Use `sleep` and a `while true` loop.

---

## Further Reading

- `man bash` — the complete bash reference
- [Bash Hackers Wiki](https://wiki.bash-hackers.org/)
- [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html)
- [ShellCheck](https://www.shellcheck.net/) — paste your script to find bugs

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="08-networking-tools.md">← Previous: Networking Tools</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="10-users-and-system-administration.md">Next: Users & System Administration →</a>
</div>
