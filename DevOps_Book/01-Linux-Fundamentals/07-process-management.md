# Chapter 07 — Process Management

## Learning Objectives

By the end of this chapter, you will:
- Understand what a process is and how Linux manages them
- List and inspect running processes with `ps` and `top`
- Send signals to processes with `kill`
- Manage foreground/background jobs
- Control process priority with `nice` and `renice`
- Use `systemctl` to manage services
- Debug stuck or crashed processes

## Prerequisites

- Chapter 06 — Permissions & Ownership

---

## 7.1 What Is a Process?

A **process** is a running instance of a program. When you run `ls`, that creates a process. When nginx serves requests, it's multiple processes. Every process has:

- **PID** (Process ID) — a unique number (1 to ~4 million)
- **PPID** (Parent Process ID) — the process that spawned it
- **UID** — which user owns it (and its permissions)
- **State** — running, sleeping, stopped, zombie
- **CPU and memory usage**

### Process Hierarchy

Linux processes form a tree. The root of the tree is **PID 1** (`systemd` on modern systems):

```
PID 1 (systemd)
├── sshd
│   └── bash (your SSH session)
│       └── ps (the command you just ran)
├── nginx
│   ├── nginx worker 1
│   └── nginx worker 2
├── docker
│   └── container process
└── cron
    └── backup.sh
```

```bash
# View the process tree
pstree
pstree -p        # with PIDs
pstree -a        # with command arguments
```

---

## 7.2 `ps` — Process Snapshot

`ps` takes a snapshot of current processes.

### Common `ps` Invocations

```bash
ps                      # processes in your current terminal
ps aux                  # ALL processes, detailed (most common)
ps -ef                  # another common format (BSD vs System V style)
ps aux | grep nginx     # find nginx processes
ps -u akash             # processes owned by user 'akash'
ps --sort=-%cpu | head  # sorted by CPU usage (descending)
ps --sort=-%mem | head  # sorted by memory usage
```

### Reading `ps aux` Output

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1  22560  4912 ?        Ss   10:00   0:01 /sbin/init
www-data  1234  0.5  1.2 145272 24500 ?        S    10:00   0:30 nginx worker
akash     5678  2.1  0.8  45000 16000 pts/0    R+   11:00   0:02 python app.py
```

| Column | Meaning |
|--------|---------|
| USER | Owner of the process |
| PID | Process ID |
| %CPU | CPU usage percentage |
| %MEM | Memory usage percentage |
| VSZ | Virtual memory size (KB) |
| RSS | Resident Set Size — actual RAM used (KB) |
| TTY | Terminal (`?` = no terminal, background) |
| STAT | Process state |
| START | Start time |
| TIME | Total CPU time consumed |
| COMMAND | The command that started the process |

### Process States (STAT column)

| State | Meaning |
|-------|---------|
| `R` | Running or runnable |
| `S` | Sleeping (interruptible) — waiting for event |
| `D` | Sleeping (uninterruptible) — usually I/O wait |
| `Z` | Zombie — finished but parent hasn't read exit status |
| `T` | Stopped (Ctrl+Z or SIGSTOP) |
| `+` | In foreground process group |
| `s` | Session leader |
| `l` | Multi-threaded |

```bash
# Find zombie processes
ps aux | awk '$8 == "Z" { print }'

# Find all processes in uninterruptible sleep (potential I/O issue)
ps aux | awk '$8 == "D" { print }'
```

---

## 7.3 `top` — Live Process Monitor

`top` shows a live, auto-refreshing view of processes:

```bash
top              # interactive process monitor
top -u akash     # only processes for user akash
top -p 1234      # watch a specific PID
```

### top Header Explained

```
top - 10:00:01 up 3 days, 5:30,  2 users,  load average: 0.42, 0.38, 0.35
Tasks: 145 total,   1 running, 144 sleeping,   0 stopped,   0 zombie
%Cpu(s):  5.3 us,  1.2 sy,  0.0 ni, 92.8 id,  0.5 wa,  0.0 hi,  0.2 si
MiB Mem :   7856.2 total,   2345.1 free,   3210.5 used,   2300.6 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   4200.1 avail Mem
```

- **load average:** System load over last 1, 5, 15 minutes. Roughly = avg number of processes wanting CPU. On a 4-core system, load of 4.0 = 100% busy.
- **us:** user space CPU, **sy:** kernel CPU, **wa:** waiting for I/O, **id:** idle
- **buff/cache:** RAM used for disk caching (can be reclaimed)

### top Interactive Commands

| Key | Action |
|-----|--------|
| `q` | Quit |
| `k` | Kill a process (enter PID) |
| `r` | Renice (change priority) |
| `u` | Filter by user |
| `M` | Sort by memory usage |
| `P` | Sort by CPU usage |
| `1` | Show individual CPU cores |
| `f` | Field management |
| `h` | Help |

### `htop` — Better top (Install Separately)

```bash
sudo apt install htop
htop                    # color, mouse support, easier interface
```

htop features: mouse support, color coding, tree view, easier kill, better filtering.

---

## 7.4 Sending Signals with `kill`

**Signals** are messages sent to processes to change their behavior. Despite the name, `kill` sends any signal — not just termination.

### Common Signals

| Signal | Number | Meaning |
|--------|--------|---------|
| `SIGHUP` | 1 | Hangup — reload config (don't restart) |
| `SIGINT` | 2 | Interrupt — same as Ctrl+C |
| `SIGQUIT` | 3 | Quit with core dump |
| `SIGKILL` | 9 | Kill immediately — cannot be ignored |
| `SIGTERM` | 15 | Terminate gracefully (default) — can be caught |
| `SIGSTOP` | 19 | Pause process — cannot be ignored |
| `SIGCONT` | 18 | Resume a stopped process |
| `SIGUSR1` | 10 | User-defined signal 1 (app-specific) |
| `SIGUSR2` | 12 | User-defined signal 2 (app-specific) |

### Using `kill`

```bash
kill 1234            # send SIGTERM (graceful terminate) to PID 1234
kill -15 1234        # same (SIGTERM = 15)
kill -SIGTERM 1234   # same with name

kill -9 1234         # SIGKILL — force kill immediately (no cleanup)
kill -SIGKILL 1234   # same

kill -1 1234         # SIGHUP — reload config
kill -HUP 1234       # same

# Kill by name
killall nginx        # kill all processes named "nginx"
pkill nginx          # same, but more flexible pattern matching
pkill -u akash       # kill all processes owned by user akash

# Send SIGHUP to reload nginx config (instead of full restart)
sudo kill -HUP $(cat /run/nginx.pid)
sudo nginx -s reload   # same effect, nginx-specific way
```

### When to Use SIGTERM vs SIGKILL

1. **Always try SIGTERM first** — it gives the process a chance to:
   - Flush buffers to disk
   - Close database connections
   - Clean up temp files
   - Save state

2. **Use SIGKILL only if SIGTERM doesn't work** — the process may be stuck or ignoring signals

```bash
# Graceful approach:
kill 1234              # SIGTERM
sleep 5
kill -9 1234           # SIGKILL if still running

# Or use pkill with timeout:
pkill -15 myapp
sleep 3
pkill -9 myapp 2>/dev/null  # force if still running
```

### `pgrep` and `pkill` — Pattern-Based Process Management

```bash
pgrep nginx              # get PIDs of all nginx processes
pgrep -l nginx           # PIDs with names
pgrep -u www-data        # PIDs owned by www-data

pkill nginx              # kill by name
pkill -9 python          # force kill all python processes
pkill -HUP sshd          # reload SSH config
pkill -u testuser        # kill all processes owned by testuser
```

---

## 7.5 Background and Foreground Jobs

### Starting Background Jobs

```bash
command &               # start command in background
sleep 100 &             # runs in background
# [1] 12345             # job number and PID printed
```

### Job Control Commands

```bash
jobs                    # list background jobs
jobs -l                 # with PIDs

fg                      # bring most recent background job to foreground
fg %1                   # bring job 1 to foreground
fg %2                   # bring job 2 to foreground

bg                      # resume stopped job in background
bg %1                   # resume job 1 in background

Ctrl+Z                  # stop (pause) foreground job
Ctrl+C                  # terminate foreground job
```

### `nohup` — Keep Running After Logout

```bash
nohup command &         # run command immune to hangup, output to nohup.out
nohup ./deploy.sh &
nohup python3 server.py > server.log 2>&1 &
```

### `disown` — Detach Job from Shell

```bash
./long-running-script.sh &
disown %1               # detach job 1 from this shell session
```

### `screen` / `tmux` — Session Multiplexers

For running processes that must survive SSH disconnection:

```bash
# tmux (modern, recommended)
tmux                    # start session
tmux new -s mysession   # start named session
# Run your command
# Ctrl+B, then D to detach (session keeps running)
tmux attach -t mysession  # reattach later

# screen (older)
screen -S mysession
# Run command
# Ctrl+A, then D to detach
screen -r mysession     # reattach
```

---

## 7.6 Process Priority with `nice` and `renice`

Linux schedules processes based on **priority** (-20 = highest priority, +19 = lowest). The default is 0.

```bash
nice -n 10 ./backup.sh        # start with lower priority (higher nice = lower priority)
nice -n -5 ./critical.sh      # higher priority (requires sudo for negative nice)
sudo nice -n -20 ./urgent.sh  # maximum priority

renice +10 1234               # lower priority of running process 1234
renice -5 1234                # raise priority (requires sudo)
sudo renice -n 5 -p 1234      # another syntax

# View priority in ps
ps aux -o pid,ni,pri,cmd | head -20
```

> **DevOps use:** Lower priority (`nice +10`) for batch jobs (backups, reports) so they don't steal CPU from production services.

---

## 7.7 `systemctl` — Managing Services

On modern Linux (systemd-based), services are managed with `systemctl`:

```bash
# Check service status
systemctl status nginx
systemctl status docker
systemctl status ssh

# Start/stop/restart
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx     # reload config without full restart

# Enable/disable at boot
sudo systemctl enable nginx     # start nginx at every boot
sudo systemctl disable nginx    # don't start at boot

# Check if running
systemctl is-active nginx       # prints "active" or "inactive"
systemctl is-enabled nginx      # prints "enabled" or "disabled"

# List all services
systemctl list-units --type=service
systemctl list-units --type=service --state=running
systemctl list-units --type=service --state=failed

# View service logs
journalctl -u nginx              # all logs for nginx
journalctl -u nginx -f           # follow nginx logs
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -n 50        # last 50 lines
```

### Writing a systemd Service Unit

```bash
# Create a service file
sudo vim /etc/systemd/system/myapp.service
```

```ini
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 /opt/myapp/app.py
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
# After creating the file:
sudo systemctl daemon-reload      # reload systemd config
sudo systemctl enable myapp       # enable at boot
sudo systemctl start myapp        # start now
systemctl status myapp            # verify
```

---

## 7.8 Monitoring Resource Usage

### Memory

```bash
free -h                 # RAM and swap usage (human-readable)
free -m                 # in megabytes

cat /proc/meminfo       # detailed memory info
vmstat 1 5              # virtual memory stats every 1 sec, 5 times
```

### CPU

```bash
uptime                  # load averages
mpstat 1 5              # per-CPU stats (install sysstat)
iostat -x 1 5           # CPU + I/O stats
sar -u 1 10             # CPU usage over time
```

### Disk I/O

```bash
iostat -d 1 5           # disk I/O stats
iotop                   # like top but for disk I/O (sudo apt install iotop)
sudo iotop -o           # only show processes doing I/O
```

### `lsof` — List Open Files

```bash
lsof                            # all open files (huge output)
lsof -p 1234                    # open files for process 1234
lsof -u akash                   # open files by user
lsof /var/log/nginx/access.log  # which process has this file open?
lsof -i :80                     # what's using port 80?
lsof -i :8080                   # what's using port 8080?
lsof -i TCP:443                 # what's using TCP port 443?
```

### `strace` — System Call Tracer (Debugging)

```bash
strace ls                        # trace system calls of 'ls'
strace -p 1234                   # attach to running process
strace -e trace=file ls          # trace only file-related calls
strace -o output.txt ./program   # save to file
```

---

## 7.9 Practical Process Debugging Scenarios

### Scenario 1: Port Already in Use

```bash
# Error: "Address already in use" on port 8080
lsof -i :8080
ss -tlnp | grep :8080
fuser 8080/tcp           # shows PID using port 8080
kill $(fuser 8080/tcp)   # kill that process
```

### Scenario 2: Finding Memory Hog

```bash
ps aux --sort=-%mem | head -10
# Or in one line:
ps aux | awk 'NR>1 {print $4, $11}' | sort -rn | head -10
```

### Scenario 3: Process Not Dying

```bash
# Try graceful first
kill 1234
sleep 3
# Check if still alive
ps -p 1234 && echo "still alive" || echo "dead"
# Force kill
kill -9 1234
```

### Scenario 4: CPU Spike Investigation

```bash
top                       # press P to sort by CPU
# Find the PID, then:
strace -p <PID>           # see what it's doing
lsof -p <PID>             # see what files it has open
cat /proc/<PID>/cmdline   # exact command line
```

---

## Summary

| Command | Purpose |
|---------|---------|
| `ps aux` | Snapshot of all processes |
| `top` | Live process monitor |
| `htop` | Better live monitor (install separately) |
| `kill -15 PID` | Graceful terminate |
| `kill -9 PID` | Force kill |
| `pkill name` | Kill by name |
| `jobs` | List background jobs |
| `fg` / `bg` | Foreground/background jobs |
| `nohup cmd &` | Run immune to hangup |
| `systemctl status svc` | Check service status |
| `lsof -i :PORT` | What's using a port |

---

## Knowledge Check

1. What is the difference between SIGTERM and SIGKILL?
2. How do you run a process in the background?
3. What does `Ctrl+Z` do to a process?
4. How do you restart a service and verify it's running?
5. A process is using port 8080 and you need to free it — what commands do you run?
6. What does a "zombie" process mean?

---

## Hands-On Exercise

```bash
# 1. View all processes and their CPU/memory
ps aux | sort -k3 -rn | head -10   # top by CPU
ps aux | sort -k4 -rn | head -10   # top by memory

# 2. Start a background job
sleep 300 &
sleep 200 &
jobs -l         # list jobs with PIDs

# 3. Bring first job to foreground, then stop it
fg %1
# Press Ctrl+Z to stop
jobs            # see it's stopped
bg %1           # resume in background
kill %1         # kill it

# 4. Kill the second sleep
kill %2
# Or: kill $(pgrep sleep)

# 5. Monitor nginx (if installed)
sudo systemctl status nginx
# If not installed:
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl status nginx
curl localhost              # test it works
sudo systemctl stop nginx

# 6. Find what's using port 80
sudo lsof -i :80
sudo ss -tlnp | grep :80

# 7. Check system load
uptime
top
# Press q to quit
```

**Challenge:** Write a shell one-liner that: finds the PID of the process using the most memory, prints its name and PID, and then asks you to confirm before killing it.

---

## Further Reading

- `man ps`, `man kill`, `man systemctl`
- `man 7 signal` — signal reference
- [Linux Process Management](https://www.redhat.com/sysadmin/linux-process-management)

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="06-permissions-and-ownership.md">← Previous: Permissions & Ownership</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="08-networking-tools.md">Next: Networking Tools →</a>
</div>
