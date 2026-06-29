# Chapter 03 — File & Directory Commands

## Learning Objectives

By the end of this chapter, you will:
- Navigate the filesystem with confidence using `cd`, `pwd`, `ls`
- Create, copy, move, and delete files and directories
- Use wildcards and glob patterns to work with multiple files
- Find files and directories with `find`
- Create links between files with `ln`
- Use tab completion and command history to work faster

## Prerequisites

- Chapter 01 — Introduction & Why Linux
- Chapter 02 — Linux Filesystem Hierarchy

---

## 3.1 Navigation Commands

### `pwd` — Print Working Directory

You always need to know *where you are*. `pwd` tells you.

```bash
pwd
# /home/akash

cd /etc
pwd
# /etc
```

### `cd` — Change Directory

```bash
cd /etc              # absolute path — goes to /etc from anywhere
cd nginx             # relative path — goes into nginx/ from current dir
cd ..                # go up one level
cd ../..             # go up two levels
cd ~                 # go to your home directory
cd -                 # go back to the previous directory (very useful!)
cd                   # same as cd ~ (no argument = home)
```

### Absolute vs Relative Paths

```
Absolute path: starts with /
  /etc/nginx/nginx.conf
  /home/akash/projects
  /var/log/syslog

Relative path: starts from current directory
  If you're in /etc:
    nginx/nginx.conf   →  means /etc/nginx/nginx.conf
    ../var/log         →  means /var/log
    ./hosts            →  means /etc/hosts (. = current dir)
```

### `ls` — List Directory Contents

```bash
ls                   # basic list
ls -l                # long format (permissions, owner, size, date)
ls -a                # show hidden files (starting with .)
ls -la               # long format + hidden files
ls -lh               # long format + human-readable sizes
ls -lt               # long format + sort by modification time (newest first)
ls -ltr              # long format + sort by time, reversed (oldest first)
ls -R                # recursive — list all subdirectories
ls -d */             # list only directories
ls /etc              # list a specific directory
ls -la /var/log      # long list of /var/log
```

### Decoding `ls -l` Output

```
-rw-r--r-- 1 akash akash 4096 Jun 23 10:00 myfile.txt
│           │ │     │     │    │             └─ filename
│           │ │     │     │    └─ last modified time
│           │ │     │     └─ file size in bytes
│           │ │     └─ group name
│           │ └─ owner username
│           └─ number of hard links
└─ permissions (we cover this fully in Chapter 06)
```

The first character:
- `-` = regular file
- `d` = directory
- `l` = symbolic link
- `c` = character device
- `b` = block device

---

## 3.2 Creating Files and Directories

### `mkdir` — Make Directory

```bash
mkdir projects                      # create a directory
mkdir -p projects/app/src           # create nested dirs (parents too)
mkdir -p /opt/myapp/{bin,lib,conf}  # create multiple subdirs at once

# Verify
ls -la projects/
```

### `touch` — Create Empty File / Update Timestamp

```bash
touch myfile.txt            # create empty file (or update timestamp if exists)
touch file1.txt file2.txt   # create multiple files
touch notes/{jan,feb,mar}.md  # create multiple in a directory
```

### Brace Expansion (Powerful!)

```bash
mkdir -p app/{frontend,backend,database}
# Creates: app/frontend/  app/backend/  app/database/

touch app/backend/{server.py,config.py,requirements.txt}
# Creates three files inside app/backend/

echo {1..5}          # 1 2 3 4 5
echo {a..e}          # a b c d e
```

---

## 3.3 Copying Files and Directories

### `cp` — Copy

```bash
cp source.txt destination.txt       # copy file
cp source.txt /tmp/                  # copy to a directory (keeps filename)
cp source.txt /tmp/newname.txt       # copy and rename

cp -r sourcedir/ destdir/            # copy directory recursively
cp -rp sourcedir/ destdir/           # copy preserving permissions, timestamps
cp -rv sourcedir/ destdir/           # verbose — show what's being copied

cp *.txt /tmp/                       # copy all .txt files to /tmp
```

### Common `cp` Flags

| Flag | Meaning |
|------|---------|
| `-r` | Recursive (required for directories) |
| `-p` | Preserve permissions, timestamps, ownership |
| `-v` | Verbose output |
| `-i` | Interactive — ask before overwriting |
| `-n` | No-clobber — never overwrite |
| `-u` | Only copy if source is newer |

```bash
# Safe copy with backup
cp -b important.conf important.conf.bak
cp --backup=numbered config.yaml config.yaml  # creates config.yaml~
```

---

## 3.4 Moving and Renaming Files

### `mv` — Move (and Rename)

In Linux, **renaming is just moving to a new name in the same directory**:

```bash
mv oldname.txt newname.txt           # rename
mv file.txt /tmp/                    # move to directory
mv file.txt /tmp/newname.txt         # move and rename

mv -v *.log /var/log/archive/        # verbose: move all .log files
mv -i source dest                    # ask before overwriting
mv -n source dest                    # never overwrite existing files
```

```bash
# Move entire directory
mv /opt/app-v1/ /opt/app-v2/

# Rename multiple files with a loop (preview: scripting teaser)
for f in *.txt; do mv "$f" "${f%.txt}.md"; done
```

---

## 3.5 Deleting Files and Directories

### `rm` — Remove

```bash
rm file.txt                  # delete a file
rm file1.txt file2.txt       # delete multiple files
rm -v file.txt               # verbose
rm -i file.txt               # interactive (confirm each)

rm -r directory/             # delete directory and contents recursively
rm -rf directory/            # force delete (no prompts) — DANGEROUS
rm -rf /path/to/dir/*        # delete contents but keep the directory
```

> **WARNING: `rm -rf` is permanent. There is no Recycle Bin in Linux.**
>
> The most feared command in Linux: `rm -rf /` — deletes EVERYTHING.
> Modern Linux adds a `--no-preserve-root` safeguard, but don't test this.

### `rmdir` — Remove Empty Directory

```bash
rmdir empty_dir/        # only works if directory is empty
rmdir -p a/b/c/         # remove a/b/c, then a/b if empty, then a if empty
```

### Safe Deletion Practices

```bash
# Always check what you're about to delete first
ls -la directory_to_delete/

# Use -i flag for interactive confirmation
rm -ri directory/

# Move to /tmp instead of deleting (recoverable)
mv important_dir/ /tmp/important_dir_backup/

# Install 'trash-cli' for a safer delete
sudo apt install trash-cli
trash file.txt       # moves to trash instead of permanent delete
```

---

## 3.6 Wildcards and Glob Patterns

Wildcards let you match multiple files at once:

| Pattern | Matches |
|---------|---------|
| `*` | Any sequence of characters (including none) |
| `?` | Exactly one character |
| `[abc]` | One character from the set: a, b, or c |
| `[a-z]` | One character in range a to z |
| `[!abc]` | One character NOT in the set |
| `{a,b,c}` | Brace expansion: exactly a, b, or c |

```bash
ls *.txt              # all .txt files
ls *.log              # all .log files
ls file?.txt          # file1.txt, fileA.txt, but not file10.txt
ls file[0-9].txt      # file0.txt through file9.txt
ls [A-Z]*.conf        # .conf files starting with uppercase

cp *.conf /tmp/       # copy all .conf files
rm *.tmp              # delete all .tmp files
ls /etc/*.conf        # all .conf files in /etc
```

---

## 3.7 Finding Files with `find`

`find` is one of the most powerful Linux commands. It searches for files based on name, type, size, date, permissions, and more.

### Basic Syntax

```bash
find [where-to-search] [criteria] [action]
```

### Find by Name

```bash
find . -name "*.txt"           # all .txt files in current dir (recursive)
find /etc -name "*.conf"       # all .conf files in /etc
find /home -name ".bashrc"     # find .bashrc files
find / -name "nginx.conf" 2>/dev/null    # search everywhere (suppress errors)
find . -iname "*.TXT"          # case-insensitive name match
```

### Find by Type

```bash
find . -type f          # only files
find . -type d          # only directories
find . -type l          # only symbolic links
```

### Find by Size

```bash
find /var -size +100M         # files larger than 100MB
find /tmp -size -1k           # files smaller than 1KB
find / -size +1G 2>/dev/null  # files larger than 1GB
```

### Find by Time

```bash
find . -mtime -7         # modified in last 7 days
find . -mtime +30        # modified more than 30 days ago
find . -newer file.txt   # modified more recently than file.txt
find . -mmin -60         # modified in last 60 minutes
```

### Find by Permissions

```bash
find /etc -perm 644      # exact permission 644
find / -perm -4000       # files with SUID bit set (security scanning!)
find / -perm -2000       # files with SGID bit set
```

### Combining Criteria

```bash
find . -type f -name "*.log" -size +10M       # large log files
find /home -type f -mtime +90 -name "*.bak"   # old backup files
find . -type f -name "*.py" -not -name "test_*"  # Python files, not tests
```

### Find + Execute (the -exec flag)

```bash
# Delete all .tmp files
find /tmp -name "*.tmp" -exec rm {} \;

# Delete all .tmp files (faster with +)
find /tmp -name "*.tmp" -delete

# Show details for each found file
find . -name "*.conf" -exec ls -la {} \;

# Copy all .log files to archive
find /var/log -name "*.log" -exec cp {} /backup/ \;

# Find and compress large logs
find /var/log -size +50M -exec gzip {} \;
```

### Find with `xargs` (Even Faster)

```bash
# xargs passes found files as arguments to a command
find . -name "*.log" | xargs grep "ERROR"
find . -name "*.py" | xargs wc -l
find /tmp -name "*.tmp" | xargs rm

# Handle spaces in filenames safely
find . -name "*.txt" -print0 | xargs -0 grep "pattern"
```

---

## 3.8 Viewing File Contents (Quick Reference)

```bash
cat file.txt              # print entire file
less file.txt             # scrollable view (q to quit)
head -20 file.txt         # first 20 lines
tail -20 file.txt         # last 20 lines
tail -f /var/log/syslog   # follow file in real-time (logs!)
```

We cover these fully in Chapter 04.

---

## 3.9 Creating Links with `ln`

### Hard Links

A **hard link** creates another name for the same file on disk. Both names point to the same data (inode).

```bash
ln original.txt hardlink.txt
ls -li original.txt hardlink.txt
# Both show the same inode number
```

### Symbolic Links (Symlinks)

A **symlink** is like a shortcut — a pointer to another file or directory.

```bash
ln -s /usr/bin/python3 /usr/local/bin/python   # create symlink
ln -s /opt/myapp/bin/myapp /usr/local/bin/myapp

ls -la /usr/local/bin/python
# lrwxrwxrwx ... python -> /usr/bin/python3
```

### When to Use Each

| Type | Use Case |
|------|----------|
| Hard link | Backup references, same filesystem |
| Symlink | Aliasing paths, cross-filesystem, directories |

```bash
# DevOps common pattern: version management via symlinks
ln -s /opt/java-17/ /opt/java-current
# Update to new version:
ln -sfn /opt/java-21/ /opt/java-current
```

---

## 3.10 Useful File Utilities

### `file` — Determine File Type

```bash
file myfile             # determine type without relying on extension
file /bin/ls            # ELF 64-bit LSB pie executable
file /etc/hosts         # ASCII text
file image.jpg          # JPEG image data
```

### `stat` — Detailed File Information

```bash
stat myfile.txt
# File: myfile.txt
# Size: 4096      Blocks: 8     IO Block: 4096  regular file
# Inode: 1234567   Links: 1
# Access: 2026-06-23 10:00:00
# Modify: 2026-06-23 10:00:00
# Change: 2026-06-23 10:00:00
```

### `wc` — Word/Line/Byte Count

```bash
wc -l file.txt          # count lines
wc -w file.txt          # count words
wc -c file.txt          # count bytes
wc -l *.py              # count lines in all Python files
```

### `du` — Disk Usage

```bash
du -sh directory/       # total size of directory (human-readable)
du -sh *                # size of each item in current directory
du -sh /var/log/*       # size of each log directory
du -h --max-depth=1 /var  # one level deep
```

### `df` — Disk Free

```bash
df -h                   # disk space of all mounted filesystems
df -h /                 # space on root filesystem
df -i                   # inode usage (important: you can run out of inodes!)
```

---

## 3.11 Tab Completion and History

### Tab Completion

Press `Tab` to auto-complete:
```bash
cd /var/l<Tab>           # completes to /var/log/
cat /etc/nginx/n<Tab>    # completes to nginx.conf
git com<Tab>             # completes to git commit
```

Press `Tab` twice to see all options when multiple match.

### Command History

```bash
history                  # show all previous commands
history | grep docker    # search history
!!                       # repeat last command
!cd                      # repeat last command starting with 'cd'
!42                      # repeat command #42 from history
Ctrl+R                   # reverse search history (type to find)
```

### History Navigation

| Key | Action |
|-----|--------|
| `↑` / `↓` | Previous/next command |
| `Ctrl+R` | Search history |
| `Ctrl+A` | Jump to start of line |
| `Ctrl+E` | Jump to end of line |
| `Ctrl+W` | Delete word before cursor |
| `Ctrl+L` | Clear screen |
| `Ctrl+C` | Cancel current command |

---

## Summary

| Command | Purpose |
|---------|---------|
| `pwd` | Show current directory |
| `cd path` | Change directory |
| `ls -la` | List with details and hidden files |
| `mkdir -p path` | Create directories (including parents) |
| `touch file` | Create empty file |
| `cp -r src dst` | Copy files/directories |
| `mv src dst` | Move or rename |
| `rm -rf dir` | Delete directory (permanent!) |
| `find . -name "*.log"` | Find files by name |
| `ln -s target link` | Create symbolic link |

---

## Knowledge Check

1. What's the difference between an absolute path and a relative path?
2. How do you create a directory and all its parents in one command?
3. What does `rm -rf` do, and why is it dangerous?
4. How would you find all `.log` files larger than 50MB in `/var/log`?
5. What is the difference between a hard link and a symbolic link?
6. What does `cd -` do?

---

## Hands-On Exercise

```bash
# Set up a practice directory
mkdir -p ~/linux-practice/exercise3
cd ~/linux-practice/exercise3

# 1. Create a directory structure
mkdir -p project/{src,tests,docs,config}
touch project/src/{app.py,utils.py,models.py}
touch project/tests/{test_app.py,test_utils.py}
touch project/config/{dev.env,prod.env}
ls -R project/

# 2. Copy the entire project
cp -r project/ project-backup/

# 3. Find all .py files
find project/ -name "*.py"

# 4. Find all files (not directories)
find project/ -type f

# 5. Rename all .env files to .conf
find project/ -name "*.env" -exec bash -c 'mv "$0" "${0%.env}.conf"' {} \;
ls project/config/

# 6. Create a symlink
ln -s ~/linux-practice/exercise3/project ~/myproject
ls -la ~/myproject

# 7. Count total files in the project
find project/ -type f | wc -l

# 8. Clean up
cd ~
rm -ri ~/linux-practice/exercise3
```

**Challenge:** In one `find` command, find all `.py` files in `~/project` and display the total line count across all of them.

---

## Further Reading

- `man find` — extremely detailed find manual
- `man ls` — all ls options
- [The Linux Command Line](http://linuxcommand.org/tlcl.php) — Chapters 3-4

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="02-linux-filesystem-hierarchy.md">← Previous: Linux Filesystem Hierarchy</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="04-text-viewing-and-editing.md">Next: Text Viewing & Editing →</a>
</div>
