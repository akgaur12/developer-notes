# Chapter 02 — Linux Filesystem Hierarchy

## Learning Objectives

By the end of this chapter, you will:
- Understand why Linux uses a single unified directory tree
- Know the purpose of every major directory in the filesystem
- Navigate to the right place to find config files, logs, binaries, and data
- Understand the difference between `/etc`, `/var`, `/usr`, `/opt`, `/tmp`, and more
- Know where DevOps tools store their files

## Prerequisites

- Chapter 01 — Introduction & Why Linux

---

## 2.1 The Unified Filesystem Tree

In Windows, you have `C:\`, `D:\` — separate trees for each drive. In Linux, everything lives in **one unified tree** starting from `/` (called "root").

```
/                          ← root of everything
├── etc/                   ← system configuration
├── var/                   ← variable data (logs, caches)
├── usr/                   ← user programs and libraries
├── opt/                   ← optional/third-party software
├── home/                  ← user home directories
├── root/                  ← root user's home
├── bin/                   ← essential binaries
├── sbin/                  ← system admin binaries
├── lib/                   ← shared libraries
├── tmp/                   ← temporary files
├── proc/                  ← virtual: running processes
├── sys/                   ← virtual: kernel/hardware info
├── dev/                   ← device files
├── mnt/                   ← mount points for drives
├── media/                 ← removable media (USB, CD)
├── boot/                  ← bootloader and kernel
├── srv/                   ← service data (web servers)
└── run/                   ← runtime process data
```

This standard is called the **Filesystem Hierarchy Standard (FHS)**, maintained by the Linux Foundation. Any Linux distro follows this layout — which means once you learn it, you know where to look on *any* Linux system.

### Why One Tree?

- **Simplicity** — one namespace, no drive letters to remember
- **Flexibility** — you can mount a physical drive at any directory (`/data`, `/mnt/backup`)
- **Portability** — scripts and configs work the same everywhere

---

## 2.2 The Root Directory `/`

The root `/` is the top of the entire filesystem. Everything starts here.

```bash
ls /
# bin  boot  dev  etc  home  lib  lib64  media  mnt  opt
# proc  root  run  sbin  srv  sys  tmp  usr  var
```

> **Critical:** Do NOT confuse `/` (root directory) with the `root` user. They are different things that happen to share a name.

---

## 2.3 `/etc` — System Configuration

`/etc` stands for **"Editable Text Configuration"** (historically "et cetera"). This is where **all system-wide configuration files live**.

```bash
ls /etc
```

### Key Files and Dirs in `/etc`

| Path | Purpose |
|------|---------|
| `/etc/hostname` | The machine's hostname |
| `/etc/hosts` | Local DNS overrides (IP → hostname mapping) |
| `/etc/passwd` | User account information |
| `/etc/shadow` | Encrypted user passwords |
| `/etc/group` | Group definitions |
| `/etc/fstab` | Filesystem mount table (what to mount at boot) |
| `/etc/crontab` | System-wide cron jobs |
| `/etc/sudoers` | Who can use sudo and with what permissions |
| `/etc/ssh/sshd_config` | SSH server configuration |
| `/etc/nginx/nginx.conf` | Nginx web server configuration |
| `/etc/apt/sources.list` | Package repository list (Ubuntu/Debian) |
| `/etc/environment` | System-wide environment variables |
| `/etc/os-release` | OS name and version |
| `/etc/resolv.conf` | DNS server configuration |
| `/etc/systemd/` | Systemd service definitions |

### Practical DevOps Use

```bash
# View hostname
cat /etc/hostname

# View DNS servers your system uses
cat /etc/resolv.conf

# View all users on the system
cat /etc/passwd

# Check SSH server config (important for security)
cat /etc/ssh/sshd_config | grep -E "PermitRootLogin|PasswordAuthentication"

# View nginx config (if installed)
cat /etc/nginx/nginx.conf
```

> **Rule:** Configuration files live in `/etc`. When you install software (nginx, PostgreSQL, Docker), their config files end up here. This is always where you look first when debugging a service.

---

## 2.4 `/var` — Variable Data

`/var` holds data that **changes frequently during normal system operation** — logs, caches, databases, mail spools, and more.

```
/var/
├── log/          ← system and application logs
├── lib/          ← application state data (databases, docker)
├── cache/        ← application cache files
├── tmp/          ← temp files that persist across reboots
├── spool/        ← print/mail queues
├── www/          ← web server document root (sometimes)
└── run/          ← runtime PIDs and sockets (also at /run)
```

### `/var/log` — The Most Important Directory for DevOps

```bash
ls /var/log
```

| Log File | Purpose |
|----------|---------|
| `/var/log/syslog` | General system messages (Ubuntu) |
| `/var/log/messages` | General system messages (RHEL/CentOS) |
| `/var/log/auth.log` | Authentication events, sudo usage, SSH logins |
| `/var/log/kern.log` | Kernel messages |
| `/var/log/dmesg` | Boot-time hardware messages |
| `/var/log/apt/` | Package install/remove history |
| `/var/log/nginx/` | Nginx access and error logs |
| `/var/log/docker/` | Docker daemon logs |
| `/var/log/postgresql/` | PostgreSQL logs |

```bash
# View last 50 lines of system log
tail -50 /var/log/syslog

# Watch logs in real-time (like tail -f)
tail -f /var/log/auth.log

# Find failed login attempts
grep "Failed password" /var/log/auth.log

# View disk usage of /var/log (logs can grow large!)
du -sh /var/log/*
```

### `/var/lib` — Application State

```bash
ls /var/lib
# docker/    mysql/    postgresql/    apt/    dpkg/
```

Docker stores all images, containers, and volumes in `/var/lib/docker`. PostgreSQL stores database files in `/var/lib/postgresql`. These directories are large — always monitor disk space here.

---

## 2.5 `/usr` — User Programs

Despite the name, `/usr` is **not** where user home directories live. It stands for **Unix System Resources** and contains the bulk of the system's installed software.

```
/usr/
├── bin/       ← most user commands (ls, grep, python, git...)
├── sbin/      ← system admin commands (usually need sudo)
├── lib/       ← shared libraries for /usr/bin programs
├── lib64/     ← 64-bit libraries
├── local/     ← locally compiled/installed software
│   ├── bin/
│   ├── lib/
│   └── share/
├── share/     ← architecture-independent data (docs, icons)
├── include/   ← C header files for development
└── src/       ← source code (sometimes)
```

### Key Distinction: `/bin` vs `/usr/bin`

Historically:
- `/bin` — **essential** binaries needed to boot the system (ls, cp, bash, cat)
- `/usr/bin` — **all other** user programs installed after boot

On modern Ubuntu (18.04+), `/bin` is actually a **symlink** to `/usr/bin`. They're merged. But knowing the old distinction helps when you see `/bin` in scripts.

```bash
ls -la /bin
# /bin -> usr/bin   ← it's a symlink!

which ls        # shows: /usr/bin/ls
which python3   # shows: /usr/bin/python3
which git       # shows: /usr/bin/git
```

### `/usr/local` — Your Custom Installations

When you compile and install software from source (not via package manager), it goes in `/usr/local`. This keeps custom software separate from OS-managed packages.

```bash
ls /usr/local/bin    # custom scripts and programs
ls /usr/local/lib    # custom libraries
```

> **DevOps relevance:** Many tools like Helm, kubectl, and custom binaries get installed to `/usr/local/bin`. This directory is in your `$PATH` by default.

---

## 2.6 `/opt` — Optional/Third-Party Software

`/opt` is for **self-contained, optional software packages** — typically commercial or large applications that don't split their files across the system.

```bash
ls /opt
# google/    intellij/    splunk/    kubernetes/
```

### When Software Goes in `/opt` vs `/usr/local`

| Location | Used For |
|----------|---------|
| `/usr/local/bin` | Single binaries, compiled programs |
| `/opt/toolname/` | Self-contained packages that include their own libs, configs, and data |

Examples of `/opt` users:
- Google Chrome: `/opt/google/chrome/`
- IntelliJ IDEA: `/opt/idea/`
- Splunk: `/opt/splunk/`
- Some Kubernetes distributions

```bash
# Check what's installed in /opt
ls -la /opt

# A self-contained app in /opt has its own structure
ls /opt/splunk/
# bin/  etc/  lib/  var/  ...
```

---

## 2.7 `/home` — User Home Directories

Each user gets a home directory at `/home/username`:

```bash
ls /home
# akash   ubuntu   deploy   jenkins

# Your home directory (shortcut: ~)
ls ~
ls /home/$USER   # same thing

# Important dotfiles in home
ls -la ~
# .bashrc     ← bash configuration (loaded for each shell)
# .bash_profile ← loaded at login
# .bash_history ← command history
# .ssh/       ← SSH keys and config
```

### The `~` Shortcut

`~` always means your home directory:
```bash
cd ~            # go home
cd ~/projects   # go to ~/projects
cat ~/.bashrc   # read your bash config
```

---

## 2.8 `/proc` — The Process Filesystem (Virtual)

`/proc` is **not a real directory on disk**. It's a virtual filesystem created by the kernel in memory, giving you a window into running processes and kernel internals.

```bash
ls /proc
# 1   2   100  ...  (numbers = process IDs)
# cpuinfo  meminfo  version  uptime  ...
```

### Useful `/proc` Files

```bash
# CPU information
cat /proc/cpuinfo

# Memory information
cat /proc/meminfo

# Kernel version
cat /proc/version

# System uptime (seconds)
cat /proc/uptime

# Network interfaces and stats
cat /proc/net/dev

# Mounted filesystems
cat /proc/mounts

# Per-process info (replace 1 with any PID)
ls /proc/1/
cat /proc/1/cmdline    # what command started this process
cat /proc/1/status     # process status
ls -la /proc/1/fd/     # open file descriptors
```

> **Why DevOps cares:** Monitoring tools like Prometheus read `/proc` to collect metrics. Understanding `/proc` helps you debug processes, check memory leaks, and understand what a running program is doing.

---

## 2.9 `/sys` — Kernel and Hardware Information (Virtual)

Like `/proc`, `/sys` is virtual — it exposes kernel and hardware information and allows runtime kernel tuning.

```bash
# See all network interfaces
ls /sys/class/net/
# eth0  lo  docker0

# Check if a service is enabled at boot (via udev)
cat /sys/class/net/eth0/operstate   # up or down

# System power management
ls /sys/power/
```

---

## 2.10 `/dev` — Device Files

In Linux, **hardware devices are represented as files** in `/dev`. This is the "everything is a file" philosophy in action.

```bash
ls /dev
# sda   sdb   sda1   tty0   null   zero   random   urandom
```

| Device | Purpose |
|--------|---------|
| `/dev/sda` | First SATA/SCSI disk |
| `/dev/sda1` | First partition of first disk |
| `/dev/nvme0n1` | NVMe SSD |
| `/dev/tty` | Current terminal |
| `/dev/null` | Discard all input (black hole) |
| `/dev/zero` | Infinite stream of zeros |
| `/dev/random` | Cryptographically secure random data |
| `/dev/urandom` | Faster pseudorandom data |

### Practical Uses

```bash
# Discard output (send to /dev/null)
some-noisy-command > /dev/null 2>&1

# Generate a random password
head -c 16 /dev/urandom | base64

# Check disk devices
ls /dev/sd*   # SATA disks
ls /dev/nvme* # NVMe disks
```

---

## 2.11 `/tmp` — Temporary Files

`/tmp` is for files that are **only needed temporarily**. The OS clears this directory on every reboot (or via a cleanup timer on some systems).

```bash
# Create a temp file
echo "test data" > /tmp/mytest.txt

# Most systems auto-clear /tmp at reboot
# Some use systemd-tmpfiles which clears after 10 days

# Check /tmp usage
df -h /tmp
du -sh /tmp/*
```

> **Warning:** Never store important data in `/tmp`. It will be deleted. Use it for scratch space during scripts.

---

## 2.12 `/boot` — Boot Files

Contains the kernel, initramfs (initial RAM filesystem), and bootloader config.

```bash
ls /boot
# grub/         ← GRUB bootloader config
# vmlinuz-*     ← Linux kernel binary
# initrd.img-*  ← initial RAM disk
# config-*      ← kernel config

# Check kernel version currently running
uname -r
# 5.15.0-1041-aws
```

> Don't modify `/boot` unless you know exactly what you're doing. Mistakes here make the system unbootable.

---

## 2.13 Disk Usage — Finding Space Problems

```bash
# Disk free space (human readable)
df -h
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/root        29G  6.2G   23G  22% /
# /dev/sda1       511M  4.0K  511M   1% /boot/efi

# Disk usage of a directory tree
du -sh /var/log
du -sh /var/lib/docker

# Find largest directories
du -h /var | sort -rh | head -20

# Find largest files
find /var -type f -size +100M
```

---

## Summary

| Directory | Remember It As |
|-----------|---------------|
| `/etc` | **Configuration** — system and app configs |
| `/var` | **Variable data** — logs, databases, caches |
| `/var/log` | **Logs** — always check here when debugging |
| `/usr/bin` | **Programs** — all installed user commands |
| `/usr/local` | **Custom installs** — manually installed tools |
| `/opt` | **Third-party** — self-contained big packages |
| `/home` | **Users** — personal files and dotfiles |
| `/proc` | **Processes** — live kernel/process data |
| `/dev` | **Devices** — hardware as files |
| `/tmp` | **Scratch** — temporary, cleared on reboot |
| `/boot` | **Boot** — kernel and bootloader |

---

## Knowledge Check

1. Where would you find the configuration file for an SSH server?
2. Where do application logs typically live?
3. What is the difference between `/bin` and `/usr/bin` on modern systems?
4. What is `/proc` and why is it special?
5. A developer installed custom software from source — where should it end up?
6. You see a file at `/dev/null` — what is it used for?

---

## Hands-On Exercise

```bash
# 1. Explore each major directory and list its contents
ls /etc | head -20
ls /var/log
ls /usr/bin | head -20
ls /opt 2>/dev/null || echo "Nothing in /opt"
ls /home

# 2. Find your machine's hostname two different ways
cat /etc/hostname
hostname

# 3. Find how much disk space /var/log is using
du -sh /var/log

# 4. Look at your system's CPU info
cat /proc/cpuinfo | grep "model name" | head -1

# 5. Find the total and available RAM
cat /proc/meminfo | grep -E "MemTotal|MemAvailable"

# 6. List all files in /dev that start with 'sd'
ls /dev/sd* 2>/dev/null || echo "No SATA disks found (might be NVMe)"
ls /dev/nvme* 2>/dev/null || echo "No NVMe disks found"

# 7. Find how much free disk space you have
df -h /

# 8. List what's in /usr/local/bin
ls /usr/local/bin
```

**Challenge:** Find which directory is consuming the most space under `/var`. Use `du -sh /var/* | sort -rh | head -5`.

---

## Further Reading

- `man hier` — the filesystem hierarchy manual page (built-in!)
- [FHS Specification](https://refspecs.linuxfoundation.org/fhs.shtml)
- [The Linux Command Line](http://linuxcommand.org/tlcl.php) — Chapter 2

---

<div style="display:flex; justify-content:space-between; align-items:center;">
  <a href="./01-introduction-and-prerequisites.md">← Previous: Introduction & Why Linux</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./03-file-and-directory-commands.md">Next: File & Directory Commands →</a>
</div>

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="01-introduction-and-prerequisites.md">← Previous: Introduction & Why Linux</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="03-file-and-directory-commands.md">Next: File & Directory Commands →</a>
</div>
