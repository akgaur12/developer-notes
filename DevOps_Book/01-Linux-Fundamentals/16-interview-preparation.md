# Chapter 16 — Interview Preparation

## Learning Objectives

By the end of this chapter, you will:
- Answer common Linux interview questions confidently
- Handle scenario-based troubleshooting questions
- Discuss Linux internals when asked
- Demonstrate practical command knowledge
- Approach system design questions with Linux in mind

---

## 16.1 Frequently Asked Questions — Beginner Level

### Q1: What is the difference between a process and a thread?

**Answer:** A **process** is an independent program in execution with its own memory space, PID, and file descriptors. A **thread** is a unit of execution within a process — multiple threads share the same memory space and file descriptors. In Linux, both are managed with `clone()` system calls, but processes have `CLONE_VM` set to false (separate memory) while threads share memory.

```bash
# See threads of a process
ps -T -p <PID>
ls /proc/<PID>/task/    # each subdir = one thread
```

### Q2: How do you find a file containing a specific string?

```bash
grep -r "search_string" /path/to/search/
grep -rl "search_string" /etc/    # only filenames
find / -type f -exec grep -l "string" {} \;  # using find
```

### Q3: What is the difference between `>` and `>>`?

- `>` **overwrites** the file (creates if not exists)
- `>>` **appends** to the file (creates if not exists)

### Q4: How do you check disk space?

```bash
df -h           # free/used space per filesystem
du -sh /var/*   # space used by each directory
```

### Q5: What does `chmod 755` mean?

- `7` (owner) = `rwx` = 4+2+1
- `5` (group) = `r-x` = 4+0+1
- `5` (others) = `r-x` = 4+0+1

Owner can read, write, and execute. Group and others can read and execute.

---

## 16.2 Frequently Asked Questions — Intermediate Level

### Q6: Explain the boot process of a Linux system.

**Answer:**
1. **BIOS/UEFI** — Power on, POST (Power-On Self-Test), finds bootable device
2. **MBR/GPT** — Master Boot Record, loads bootloader
3. **GRUB** — Grand Unified Bootloader, shows kernel choice, loads kernel
4. **Kernel initialization** — decompresses kernel, initializes hardware, mounts initramfs
5. **initramfs** — minimal root filesystem in RAM, loads drivers needed to mount real root
6. **Root filesystem mount** — switches to real root filesystem
7. **PID 1 (systemd/init)** — first real process, starts all other processes
8. **Targets/runlevels** — systemd starts services per target (multi-user, graphical)
9. **Login prompt** — getty starts, user logs in

### Q7: What is a zombie process? How do you handle it?

**Answer:** A zombie process has finished executing but still has an entry in the process table because its parent hasn't read its exit status via `wait()`. 

```bash
ps aux | awk '$8 == "Z"'    # find zombies

# You cannot kill a zombie (it's already dead)
# Solution: kill the parent process, which forces cleanup
# Or: send SIGCHLD to parent
kill -SIGCHLD <parent_pid>
```

### Q8: What is the difference between `kill -9` and `kill -15`?

- `kill -15` (SIGTERM) — **graceful termination**. Process receives signal, can clean up (close DB connections, flush buffers), then exit. Process CAN ignore it.
- `kill -9` (SIGKILL) — **forced termination**. Kernel immediately terminates the process. Process CANNOT ignore or catch it. No cleanup possible.

**Best practice:** Always try SIGTERM first, wait a few seconds, then SIGKILL if needed.

### Q9: How does the Linux permission system work?

Every file has: owner (user), owning group, permission bits for owner/group/others.  
Three permissions: read (r=4), write (w=2), execute (x=1).  
On files: r=read content, w=modify, x=run as program.  
On directories: r=list contents, w=create/delete inside, x=enter directory.

### Q10: What is `inode`?

An **inode** (index node) is a data structure storing file metadata: permissions, owner, timestamps, file size, and pointers to data blocks. It does NOT store the filename or the actual data.

```bash
ls -i filename    # show inode number
stat filename     # detailed inode info
df -i             # inode usage per filesystem
```

Filenames are stored in directory entries, which map names to inode numbers. Hard links = two names pointing to same inode.

---

## 16.3 Frequently Asked Questions — Advanced Level

### Q11: Explain namespaces and cgroups. How do containers use them?

**Answer:**
- **Namespaces** provide **isolation** — each container gets its own view of: PID tree, network stack, filesystem mount points, hostname, users
- **cgroups** provide **resource control** — limit CPU, memory, I/O, network bandwidth per container

Docker uses both: `docker run --memory=512m` creates a cgroup with 512MB memory limit. Each container runs in its own network and PID namespace.

### Q12: What is the difference between a hard link and a soft link?

**Hard link:** Another directory entry pointing to the same inode. Shares all data with original. Cannot cross filesystems. Cannot link to directories. If original deleted, data persists.

**Soft link (symlink):** A file containing a path to another file. Can cross filesystems. Can link to directories. If target deleted, symlink becomes dangling (broken).

```bash
ln source.txt hardlink.txt      # hard link
ln -s /etc/nginx/nginx.conf ./nginx.conf  # soft link
```

### Q13: How would you troubleshoot high CPU usage?

```bash
# 1. Identify which process
top                    # sort by P
ps aux --sort=-%cpu | head

# 2. Identify what it's doing
strace -p <PID>        # system calls
lsof -p <PID>          # open files
cat /proc/<PID>/cmdline # exact command

# 3. Check if it's a specific CPU or all
top → press 1          # per-CPU breakdown

# 4. Check if it's kernel or user space
# %us = user space, %sy = kernel/system
# High %sy suggests I/O or system call issues

# 5. Check for runaway loops
strace -p <PID> -c     # count syscalls — if one dominates, it's a loop
```

### Q14: Explain the `/proc/sys/vm/swappiness` parameter.

**Answer:** Controls how aggressively the kernel swaps memory pages to disk.
- Value 0–100 (default: 60)
- Higher value = more aggressive swapping
- For servers with plenty of RAM, set to 10 to avoid swapping (which is slow):

```bash
echo 10 | sudo tee /proc/sys/vm/swappiness     # temporary
echo "vm.swappiness=10" | sudo tee /etc/sysctl.d/99-swappiness.conf  # permanent
sudo sysctl -p /etc/sysctl.d/99-swappiness.conf
```

### Q15: What happens when you type `ls` and press Enter?

1. Shell searches `$PATH` directories for `ls` binary → finds `/usr/bin/ls`
2. Shell calls `fork()` → creates child process
3. Child process calls `execve("/usr/bin/ls", ...)` → loads ls binary, replacing child
4. `ls` makes `getdents64()` system call → kernel reads directory entries
5. `ls` calls `write()` → outputs to stdout (fd 1)
6. `ls` calls `exit()` → process terminates
7. Shell calls `wait()` → collects ls exit code, prints next prompt

---

## 16.4 Scenario-Based Questions

### Scenario 1: "The web server returns 502 Bad Gateway"

```
Candidate should walk through:

1. Check nginx error log:
   tail -50 /var/log/nginx/error.log

2. Check if backend service is running:
   systemctl status myapp
   ps aux | grep myapp

3. Check backend is listening on expected port:
   ss -tlnp | grep :8080

4. Test backend directly:
   curl http://localhost:8080

5. Check nginx upstream config:
   cat /etc/nginx/sites-enabled/myapp

6. Check SELinux/AppArmor (if applicable):
   getenforce
   aa-status
```

### Scenario 2: "Server is unresponsive — what do you do?"

```
1. Check if you can still SSH in
2. If yes:
   - Check load: uptime, top
   - Check memory: free -h
   - Check disk: df -h
   - Check OOM killer: dmesg | grep -i "oom"
   - Check zombie processes
3. If no, but ping works:
   - SSH on different port?
   - Try console access (cloud provider VNC/serial)
4. If ping doesn't work:
   - Check cloud provider console for instance health
   - Check instance is running, networking configured
```

### Scenario 3: "You accidentally deleted an important config file"

```
1. Don't panic. Check if there's a backup:
   ls /etc/nginx/nginx.conf.bak*
   ls /etc/nginx/*.bak

2. Check if the process still has the file open (data still in memory):
   lsof | grep nginx.conf
   # If pid is listed: cp /proc/<pid>/fd/<fd_num> /etc/nginx/nginx.conf

3. Check version control:
   git log /etc/nginx/nginx.conf  # if using etckeeper

4. Check package manager for default config:
   apt-get install --reinstall nginx  # reinstalls default config

5. Check another server with same config for reference
```

---

## 16.5 Command Challenges (Common in Interviews)

**"Show me how you would..."**

```bash
# Find all .log files modified in the last 24 hours
find /var/log -name "*.log" -mtime -1

# Show the 5th line of a file
sed -n '5p' file.txt
awk 'NR==5' file.txt

# Count unique IP addresses in an nginx log
awk '{print $1}' /var/log/nginx/access.log | sort -u | wc -l

# Find top 5 memory-consuming processes
ps aux --sort=-%mem | head -6

# Check if a port is open on a remote host
nc -zv hostname 80 2>&1
curl -m 3 telnet://hostname:80

# Replace a string in multiple files
find . -name "*.conf" -exec sed -i 's/old/new/g' {} \;

# Show files changed in the last hour
find / -newer /tmp/hour_ago -type f 2>/dev/null
# First: touch -t $(date -d '1 hour ago' +%Y%m%d%H%M) /tmp/hour_ago

# Extract the second column from a space-delimited file
awk '{print $2}' file.txt
cut -d' ' -f2 file.txt

# Show disk usage, sorted largest first
du -sh /var/* | sort -rh | head -10

# Check which process is using port 8080
ss -tlnp | grep :8080
lsof -i :8080
fuser 8080/tcp
```

---

## 16.6 Quick-Fire Knowledge Questions

| Question | Answer |
|----------|--------|
| What is PID 1? | `systemd` (modern) or `init` (legacy) |
| Default port for SSH? | 22 |
| Command to list all running services? | `systemctl list-units --type=service --state=running` |
| What does `/dev/null` do? | Discards all input written to it |
| How do you run a command as another user? | `sudo -u username command` |
| What is a daemon? | A background process (service) |
| Difference between `/tmp` and `/var/tmp`? | `/tmp` cleared on reboot, `/var/tmp` persists |
| What is sticky bit used for? | Prevents users from deleting others' files in shared directory |
| How to see all environment variables? | `env` or `printenv` |
| What file stores encrypted passwords? | `/etc/shadow` |
| How to reload a config without restarting? | `kill -HUP <PID>` or `systemctl reload service` |
| What does `2>&1` mean? | Redirect stderr to where stdout goes |

---

## 16.7 Interview Tips

1. **Think out loud** — interviewers want to see your debugging process, not just answers
2. **Be honest about unknowns** — "I'd check the man page / Google" is better than guessing
3. **Demonstrate safety habits** — mention backups, testing, not running as root
4. **Know your `man` pages** — "I'd check `man grep`" shows good habits
5. **Emphasize production thinking** — mention logging, monitoring, rollback in answers
6. **Use concrete examples** — "I once debugged a memory leak by reading `/proc/<PID>/status`..."

---

## Summary: Must-Know Commands Before Any Interview

```bash
# File operations
ls -la, cp -r, mv, rm -rf, find, chmod, chown

# Text processing
grep -r, awk '{print $1}', sed 's/old/new/g', cut -d: -f1, sort, uniq -c

# Processes
ps aux, top, kill -15, kill -9, pgrep, pkill, systemctl

# Networking
ss -tlnp, curl -I, dig, ping, traceroute, ssh

# System
df -h, du -sh, free -h, uptime, journalctl -u service -f

# Scripting
set -euo pipefail, [[ ]], $(), $1 $@, for/while/if
```

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="15-projects.md">← Previous: Hands-On Projects</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="17-course-summary.md">Next: Course Summary →</a>
</div>
