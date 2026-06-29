# Chapter 12 — Advanced Linux Concepts

## Learning Objectives

By the end of this chapter, you will:
- Use advanced `find` and `xargs` patterns
- Debug programs with `strace` and `ltrace`
- Understand and read `/proc` filesystem data
- Analyze system performance with `perf` and `sar`
- Manage disk and filesystems with `fdisk`, `lvm`, and `mount`
- Work with archives and compression
- Use `screen`/`tmux` for session management
- Apply Linux namespaces and cgroups (Docker's foundations)

## Prerequisites

- Chapters 01–11

---

## 12.1 Advanced `find` and `xargs`

```bash
# Find files and act on them
find /var/log -name "*.log" -mtime +30 -exec gzip {} \;

# Find + xargs for parallel processing
find . -name "*.py" -print0 | xargs -0 -P4 pylint

# Find and count files per extension
find . -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn

# Find duplicate files (by size first, then md5)
find . -type f -printf '%s\n' | sort | uniq -d | while read size; do
    find . -type f -size "${size}c" -exec md5sum {} \; | sort | uniq -d -w 32
done

# Find files changed since a reference file
find /etc -newer /etc/passwd -type f

# Find and remove files > 100MB not accessed in 90 days
find /var/archive -size +100M -atime +90 -delete

# Delete empty directories
find . -type d -empty -delete
```

---

## 12.2 System Debugging with `strace`

`strace` traces system calls — how a program interacts with the kernel. Invaluable for debugging "why isn't this working?":

```bash
# Trace a command from start
strace ls /tmp

# Trace only specific system calls
strace -e trace=open,read,write ls /tmp
strace -e trace=network curl google.com    # network calls
strace -e trace=file ls /tmp              # file-related calls

# Attach to a running process
sudo strace -p 1234

# Trace with timestamps
strace -tt -T ls /tmp

# Save to file
strace -o /tmp/trace.txt ls /tmp

# Count syscall frequency (very useful!)
strace -c ls /tmp
# %time   seconds  usecs/call     calls    errors syscall
# 22.45    0.000090          9        10        0 mmap

# Practical: why does a program fail?
strace ./myapp 2>&1 | grep -E "open|ENOENT|EPERM"
```

---

## 12.3 Deep Dive into `/proc`

`/proc` exposes kernel internals as files:

```bash
# Per-process directories
ls /proc/$$                    # your current shell's /proc entry
cat /proc/$$/cmdline | tr '\0' ' '  # exact command line
cat /proc/$$/environ | tr '\0' '\n' # environment variables
ls -la /proc/$$/fd             # open file descriptors
cat /proc/$$/status            # detailed process status
cat /proc/$$/maps              # memory map

# System-wide
cat /proc/cpuinfo              # CPU details
cat /proc/meminfo              # memory details
cat /proc/version              # kernel version
cat /proc/uptime               # uptime in seconds
cat /proc/loadavg              # load averages
cat /proc/net/tcp              # TCP connections (hex)
cat /proc/net/dev              # network interface stats
cat /proc/diskstats            # disk I/O statistics
cat /proc/sys/fs/file-max      # maximum open files
cat /proc/sys/net/core/somaxconn  # TCP backlog limit
```

### Tuning Kernel Parameters via `/proc/sys`

```bash
# Temporary changes (lost on reboot)
echo 1 > /proc/sys/net/ipv4/ip_forward     # enable IP forwarding
echo 65535 > /proc/sys/net/core/somaxconn  # increase TCP backlog

# Permanent changes via sysctl
sudo sysctl net.ipv4.ip_forward=1
sudo sysctl -w vm.swappiness=10

# Persist across reboots (/etc/sysctl.conf or /etc/sysctl.d/)
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-networking.conf
sudo sysctl -p /etc/sysctl.d/99-networking.conf

# View all sysctl settings
sysctl -a
sysctl net.ipv4    # filtered
```

---

## 12.4 Performance Analysis

### `vmstat` — Virtual Memory Statistics

```bash
vmstat 1 10         # every 1 second, 10 times
vmstat -s           # memory stats summary
vmstat -d           # disk stats

# Output columns:
# procs: r (running), b (blocked)
# memory: swpd (swap used), free, buff, cache
# swap: si (swap in), so (swap out)
# io: bi (blocks in), bo (blocks out)
# system: in (interrupts), cs (context switches)
# cpu: us, sy, id, wa (user, system, idle, iowait)
```

### `iostat` — I/O Statistics

```bash
sudo apt install sysstat     # install if not present
iostat 1 5                   # device I/O every 1 sec, 5 times
iostat -x 1 5                # extended stats
iostat -x -d 1 5             # disk only
```

### `sar` — System Activity Reporter

```bash
sar -u 1 5          # CPU usage every 1 sec, 5 times
sar -r 1 5          # memory stats
sar -d 1 5          # disk stats
sar -n DEV 1 5      # network stats
sar -A              # all stats (historical from /var/log/sysstat/)
```

### `dstat` — All-in-One Stats

```bash
sudo apt install dstat
dstat                # cpu, disk, network, paging, system
dstat -cdngy 1       # cpu, disk, net, page, system every 1 sec
```

### `perf` — Linux Performance Tools

```bash
sudo apt install linux-tools-common linux-tools-$(uname -r)
perf top             # live performance, like top but by function
perf stat ls         # count hardware events for a command
perf record ./myapp  # record performance data
perf report          # analyze recorded data
```

---

## 12.5 Archive and Compression

### `tar` — Archive Tool

```bash
# Create archive
tar -cvf archive.tar files/           # create, verbose, filename
tar -czvf archive.tar.gz files/       # create + gzip compress
tar -cjvf archive.tar.bz2 files/      # create + bzip2 compress
tar -cJvf archive.tar.xz files/       # create + xz compress (best compression)

# Extract archive
tar -xvf archive.tar                   # extract
tar -xzvf archive.tar.gz              # extract gzipped
tar -xjvf archive.tar.bz2             # extract bzip2
tar -xJvf archive.tar.xz              # extract xz
tar -xvf archive.tar -C /tmp/         # extract to specific dir

# List contents
tar -tvf archive.tar                   # list without extracting

# Extract specific file
tar -xvf archive.tar path/to/file.txt

# Append to existing archive
tar -rvf archive.tar newfile.txt
```

### `gzip`, `bzip2`, `xz`

```bash
gzip file.txt             # compress: creates file.txt.gz (deletes original)
gzip -d file.txt.gz       # decompress (same as gunzip)
gzip -k file.txt          # keep original
gzip -9 file.txt          # maximum compression
gunzip file.txt.gz         # decompress

bzip2 file.txt            # compress: file.txt.bz2
bzip2 -d file.txt.bz2     # decompress
bunzip2 file.txt.bz2       # decompress

xz file.txt               # compress: file.txt.xz (best ratio)
xz -d file.txt.xz         # decompress
unxz file.txt.xz           # decompress
```

### `zip` and `unzip`

```bash
zip archive.zip file1 file2 file3     # create zip
zip -r archive.zip directory/         # recursive
zip -r archive.zip dir/ -x "*.git/*"  # exclude pattern
unzip archive.zip                     # extract
unzip archive.zip -d /tmp/            # extract to directory
unzip -l archive.zip                  # list contents
```

---

## 12.6 Disk and Filesystem Management

### Disk Information

```bash
lsblk                  # list block devices (tree view)
fdisk -l               # list partition tables (needs sudo)
blkid                  # show device UUIDs and filesystem types
df -h                  # disk usage
du -sh /var/*          # directory sizes
```

### Mounting Filesystems

```bash
# Mount a device
sudo mount /dev/sdb1 /mnt/data

# Mount with options
sudo mount -o ro /dev/sdb1 /mnt/data          # read-only
sudo mount -o remount,rw /mnt/data             # remount read-write

# Unmount
sudo umount /mnt/data

# View mounted filesystems
mount | grep "^/dev"
cat /proc/mounts
```

### `/etc/fstab` — Persistent Mounts

```bash
cat /etc/fstab
# UUID=xxx  /       ext4    errors=remount-ro  0 1
# UUID=xxx  /boot   ext4    defaults           0 2
# UUID=xxx  none    swap    sw                 0 0

# Add a persistent mount
echo "UUID=$(blkid -s UUID -o value /dev/sdb1) /data ext4 defaults 0 2" | sudo tee -a /etc/fstab
sudo mount -a    # mount all fstab entries (test your entry)
```

---

## 12.7 `tmux` — Terminal Multiplexer

`tmux` lets you run multiple terminal sessions, persist them across SSH disconnections, and create split-screen layouts.

### Essential tmux

```bash
# Installation
sudo apt install tmux

# Start/attach sessions
tmux                          # start new session
tmux new -s mysession         # named session
tmux attach -t mysession      # attach to existing
tmux ls                       # list sessions

# Inside tmux: prefix key = Ctrl+B
```

### Key Bindings (all start with `Ctrl+B`)

| Key Combination | Action |
|-----------------|--------|
| `Ctrl+B d` | Detach (session keeps running) |
| `Ctrl+B c` | Create new window |
| `Ctrl+B n` | Next window |
| `Ctrl+B p` | Previous window |
| `Ctrl+B 0-9` | Switch to window N |
| `Ctrl+B "` | Split horizontally |
| `Ctrl+B %` | Split vertically |
| `Ctrl+B Arrow` | Move between panes |
| `Ctrl+B z` | Zoom current pane (toggle fullscreen) |
| `Ctrl+B x` | Kill current pane |
| `Ctrl+B ?` | Help (list all keys) |

### `~/.tmux.conf` — Configuration

```bash
cat > ~/.tmux.conf << 'EOF'
# Change prefix to Ctrl+A (like screen)
set -g prefix C-a
unbind C-b
bind C-a send-prefix

# Mouse support
set -g mouse on

# Start windows from 1 (not 0)
set -g base-index 1

# Status bar customization
set -g status-style bg=black,fg=white
set -g window-status-current-style bg=blue,fg=white

# Increase history
set -g history-limit 50000

# Easy reload
bind r source-file ~/.tmux.conf \; display "Reloaded!"
EOF
```

---

## 12.8 Linux Namespaces and Cgroups (Docker's Foundation)

Understanding these helps you understand how Docker, Kubernetes, and containers work.

### Namespaces — Isolation

Namespaces isolate different aspects of the system for each container:

| Namespace | Isolates |
|-----------|---------|
| `mnt` | Mount points — each container has its own `/` |
| `pid` | Process IDs — container sees PID 1 as its init |
| `net` | Network — each container gets its own network stack |
| `ipc` | IPC — inter-process communication |
| `uts` | Hostname — each container can have its own hostname |
| `user` | User IDs — map container root to non-root on host |
| `cgroup` | cgroup root — resource limits |

```bash
# See namespaces for a process
ls -la /proc/$$/ns/

# Create a new PID namespace (Docker does this internally)
sudo unshare --pid --fork --mount-proc bash
# Now "ps aux" shows only processes in this namespace!
```

### cgroups — Resource Control

cgroups (control groups) limit CPU, memory, I/O for process groups:

```bash
# View cgroup hierarchy
ls /sys/fs/cgroup/

# Docker uses cgroups to implement --memory and --cpus flags
# /sys/fs/cgroup/docker/<container-id>/

# Create a cgroup (v1)
sudo mkdir /sys/fs/cgroup/memory/mygroup
echo 104857600 | sudo tee /sys/fs/cgroup/memory/mygroup/memory.limit_in_bytes  # 100MB
echo $$ | sudo tee /sys/fs/cgroup/memory/mygroup/tasks  # add current process
```

> **Why this matters for DevOps:** Docker containers ARE Linux namespaces + cgroups + a root filesystem. Understanding this helps you debug container issues, resource limits, and security.

---

## 12.9 Advanced Text Processing Patterns

```bash
# Multi-file log analysis
awk '{print FILENAME, $0}' /var/log/nginx/*.log | grep "500"

# Extract JSON values (without jq)
grep -oP '"status":\s*"\K[^"]+' response.json

# Process CSV
awk -F, 'NR>1 {sum += $3} END {print "Total:", sum}' data.csv

# Generate statistics from logs
awk '{
    split($4, t, ":")
    hour = substr(t[1], 2)
    requests[hour]++
}
END {
    for (h in requests) print h, requests[h]
}' /var/log/nginx/access.log | sort -n

# Rolling 5-minute window (complex)
awk 'BEGIN {OFMT="%.0f"}
{
    cmd = "date -d \"" $1 " " $2 "\" +%s"
    cmd | getline ts
    close(cmd)
    window[NR] = ts
    data[NR] = $3
    while (window[NR] - window[NR-count+1] > 300) count--
    count++
    sum += $3
    print ts, sum/count
}' timeseries.dat
```

---

## Summary

| Topic | Key Commands |
|-------|-------------|
| Debugging | `strace -p PID`, `lsof -p PID` |
| Performance | `vmstat 1`, `iostat -x 1`, `sar -u 1` |
| Archives | `tar -czvf`, `tar -xzvf` |
| Sessions | `tmux new -s name`, `Ctrl+B d` to detach |
| Kernel tuning | `sysctl`, `/proc/sys/` |

---

## Knowledge Check

1. What does `strace -c ls` show you?
2. What kernel parameters are tunable via `/proc/sys`?
3. What is the difference between a namespace and a cgroup?
4. How do you compress a directory into a `.tar.gz` file?
5. How do you detach from a tmux session without killing it?

---

## Hands-On Exercise

```bash
# 1. strace a simple command
strace -c ls /etc 2>&1 | tail -15    # show syscall statistics

# 2. Create and extract archives
mkdir /tmp/archive-test
echo "File 1" > /tmp/archive-test/file1.txt
echo "File 2" > /tmp/archive-test/file2.txt

tar -czvf /tmp/myarchive.tar.gz /tmp/archive-test/
tar -tvf /tmp/myarchive.tar.gz    # list contents
tar -xzvf /tmp/myarchive.tar.gz -C /tmp/extracted/
# verify:
ls /tmp/extracted/tmp/archive-test/

# 3. Monitor system performance
vmstat 1 5
iostat 1 3 2>/dev/null || echo "iostat not available"

# 4. tmux session
tmux new -s practice
# Create 2 windows: Ctrl+B c
# Split current window: Ctrl+B %
# Navigate: Ctrl+B arrow
# Detach: Ctrl+B d
# Reattach:
tmux attach -t practice

# 5. Check open files for a process
sudo lsof -p $(pgrep sshd | head -1) | head -20
```

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="11-intermediate-concepts.md">← Previous: Pipes, Redirects & Environment</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="13-best-practices.md">Next: Linux Best Practices →</a>
</div>
