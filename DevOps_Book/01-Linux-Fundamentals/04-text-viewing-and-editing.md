# Chapter 04 — Text Viewing & Editing

## Learning Objectives

By the end of this chapter, you will:
- View file contents in multiple ways suited to different situations
- Navigate large files efficiently with `less`
- Follow live log files with `tail -f`
- Edit files from the terminal using `nano` and `vim`
- Know essential vim commands for survival in any server environment

## Prerequisites

- Chapter 03 — File & Directory Commands

---

## 4.1 Why Text Viewing Matters in DevOps

Almost everything in Linux is text — configuration files, logs, scripts, data. Reading and editing these files is a daily task:

- Checking application logs for errors
- Editing config files for nginx, docker, systemd
- Reading scripts to understand what they do
- Tailing logs to debug a live issue

You need multiple tools because different situations call for different approaches:
- Small file → `cat`
- Large file → `less`
- Last N lines → `tail`
- Live log stream → `tail -f`
- Edit a file → `nano` (easy) or `vim` (professional)

---

## 4.2 `cat` — Concatenate and Print

`cat` prints the entire file to the terminal.

```bash
cat /etc/hostname           # print a file
cat file1.txt file2.txt     # print multiple files in sequence
cat -n file.txt             # show line numbers
cat -A file.txt             # show non-printable characters (useful for debugging)
```

### Creating Files with `cat`

```bash
# Create a file (Ctrl+D to end input)
cat > newfile.txt
Line 1
Line 2
^D

# Append to a file
cat >> newfile.txt
More content
^D

# Combine files
cat header.txt body.txt footer.txt > complete.txt
```

### When NOT to Use `cat`

For large files, `cat` dumps everything at once — the terminal can't scroll back far enough. Use `less` instead.

```bash
# Don't do this for large files:
cat /var/log/syslog          # might be 100MB!

# Do this instead:
less /var/log/syslog
```

---

## 4.3 `less` — The Standard File Viewer

`less` is the professional's tool for viewing any file. It loads incrementally (fast even for huge files) and lets you scroll, search, and navigate.

```bash
less /var/log/syslog
less /etc/nginx/nginx.conf
less +G /var/log/syslog      # open at the END of the file
```

### Navigation Inside `less`

| Key | Action |
|-----|--------|
| `Space` or `f` | Scroll forward one page |
| `b` | Scroll backward one page |
| `↑` / `↓` | Scroll up/down one line |
| `g` | Go to beginning of file |
| `G` | Go to end of file |
| `/<pattern>` | Search forward for pattern |
| `?<pattern>` | Search backward for pattern |
| `n` | Next search match |
| `N` | Previous search match |
| `q` | Quit |
| `h` | Help |
| `:n` | Next file (if multiple files) |

### Search in `less`

```bash
less /var/log/syslog
# Inside less, type:
/error         # search for "error" (case-sensitive)
/ERROR         # search for "ERROR"
/[Ee]rror      # regex: search for Error or error
n              # jump to next match
N              # jump to previous match
```

### View Multiple Files

```bash
less file1.txt file2.txt    # :n to go to next, :p to go to previous
```

---

## 4.4 `head` and `tail` — Read Parts of Files

### `head` — First N Lines

```bash
head file.txt               # default: first 10 lines
head -20 file.txt           # first 20 lines
head -5 /etc/passwd         # first 5 lines of passwd

# First line (useful for CSV headers)
head -1 data.csv
```

### `tail` — Last N Lines

```bash
tail file.txt               # default: last 10 lines
tail -20 file.txt           # last 20 lines
tail -100 /var/log/syslog   # last 100 lines of syslog
```

### `tail -f` — Follow File in Real Time

This is one of the most important DevOps commands. When a service is running, you can watch its log grow in real time:

```bash
tail -f /var/log/syslog             # follow system log
tail -f /var/log/nginx/access.log   # follow nginx access log
tail -f /var/log/auth.log           # watch for login attempts
```

Press `Ctrl+C` to stop following.

### `tail -f` for Multiple Files

```bash
tail -f /var/log/nginx/access.log /var/log/nginx/error.log
# Shows filename header when output comes from different files
```

### Combine `head` and `tail` — Extract Middle of File

```bash
# Get lines 100-120 (skip first 99, take next 20)
tail -n +100 file.txt | head -20
```

---

## 4.5 `more` — Basic Pager (Legacy)

`more` is an older, simpler pager. Only scrolls forward (unlike `less` which goes both ways). Use `less` instead — it's strictly better.

```bash
more /etc/passwd    # use less instead
```

---

## 4.6 `wc` — Word/Line Count

```bash
wc -l file.txt          # count lines
wc -w file.txt          # count words
wc -c file.txt          # count bytes
wc file.txt             # all three: lines words bytes filename

# Count lines in all .py files
wc -l *.py

# How many users are on this system?
wc -l /etc/passwd
```

---

## 4.7 Text Editors — Overview

| Editor | Learning Curve | Best For |
|--------|---------------|----------|
| `nano` | Very easy | Quick edits, beginners |
| `vim` | Steep | Professional use, available everywhere |
| `emacs` | Steep | Advanced users |
| `gedit`, `kate` | Easy | GUI editors (desktop only) |

> **DevOps reality:** Every server has `vi`/`vim`. If you SSH into a production server to fix a config file, you'll need vim. Learn at least the basics — it saves you in emergencies.

---

## 4.8 `nano` — The Beginner-Friendly Editor

`nano` works like a normal text editor — just type and it works.

```bash
nano filename.txt       # open (or create) a file
nano /etc/hosts         # edit a system file (may need sudo)
sudo nano /etc/nginx/nginx.conf
```

### Nano Key Bindings

The `^` symbol means `Ctrl`.

| Keys | Action |
|------|--------|
| `Ctrl+S` | Save |
| `Ctrl+O` | Write Out (save, ask for filename) |
| `Ctrl+X` | Exit (asks to save if modified) |
| `Ctrl+W` | Search (Where is) |
| `Ctrl+K` | Cut current line |
| `Ctrl+U` | Paste |
| `Ctrl+G` | Help |
| `Ctrl+/` | Go to line number |
| `Alt+U` | Undo |

### Typical nano Workflow

```bash
sudo nano /etc/hosts
# Edit the file
# Ctrl+S to save
# Ctrl+X to exit
```

---

## 4.9 `vim` — The Professional's Editor

vim stands for **Vi IMproved**. It's everywhere — any Unix-like system, Docker containers, minimal servers. Learning vim pays dividends your entire career.

### The Modal Concept

vim's most unique feature: it has **modes**. Most text editors have one mode (just type). vim has multiple:

```
NORMAL MODE (default)
    - Navigation: hjkl, w, b, gg, G
    - Deletion: dd, dw, x
    - Copy/paste: yy, p
    - Enter other modes
         ↓ i, a, o, I, A, O
INSERT MODE
    - Actually typing text
         ↑ Esc
COMMAND MODE (from normal, type :)
    - :w (save), :q (quit), :wq (save+quit)
         ↑ Enter or Esc
VISUAL MODE (from normal, type v)
    - Select text for copy/cut
         ↑ Esc
```

> **The #1 vim beginner mistake:** They type immediately and wonder why nothing works. You must press `i` first to enter INSERT mode.

### Opening and Exiting vim

```bash
vim filename.txt        # open file
vim +42 filename.txt    # open at line 42
vim +/pattern file      # open and search for pattern
```

**The Minimum You Must Know — How to Exit vim:**

```
:q         quit (fails if unsaved changes)
:q!        force quit WITHOUT saving
:w         save without quitting
:wq        save AND quit
:x         save and quit (same as :wq)
ZZ         save and quit (normal mode shortcut)
ZQ         quit without saving (normal mode shortcut)
```

This is literally a meme — people trapped in vim. Know `:q!` to escape.

### Normal Mode Navigation

```bash
# Basic movement
h   ← left
j   ↓ down
k   ↑ up
l   → right

# Word movement
w   jump to next word start
b   jump to previous word start
e   jump to end of current word

# Line movement
0   jump to start of line
$   jump to end of line
^   jump to first non-space character

# File movement
gg  go to first line
G   go to last line
:42 go to line 42
Ctrl+f   page forward
Ctrl+b   page backward
```

### Entering Insert Mode

```
i   Insert before cursor
a   Insert after cursor
o   Open new line below and insert
O   Open new line above and insert
I   Insert at start of line
A   Insert at end of line
```

### Editing in Normal Mode

```bash
x   delete character under cursor
dd  delete current line
dw  delete word from cursor
d$  delete to end of line
D   delete to end of line (same as d$)
cc  change (delete+insert) current line
cw  change word
yy  yank (copy) current line
yw  yank word
p   paste below cursor
P   paste above cursor
u   undo
Ctrl+r  redo
.   repeat last action (extremely useful)
```

### Search and Replace

```bash
/pattern       search forward
?pattern       search backward
n              next match
N              previous match

# Replace all occurrences in file
:%s/old/new/g

# Replace with confirmation
:%s/old/new/gc

# Replace in current line only
:s/old/new/g
```

### Practical vim Workflow

```bash
# Edit nginx config
sudo vim /etc/nginx/nginx.conf

# In vim:
# gg        → go to top
# /server_name  → search for server_name
# n         → find next occurrence
# i         → enter insert mode
# (make edit)
# Esc       → back to normal mode
# :wq       → save and exit

# Verify your change
cat /etc/nginx/nginx.conf | grep server_name
```

### vim Configuration (`.vimrc`)

```bash
# Create a basic .vimrc in your home directory
cat > ~/.vimrc << 'EOF'
set number          " show line numbers
set autoindent      " auto-indent new lines
set expandtab       " use spaces instead of tabs
set tabstop=4       " tab width = 4 spaces
set shiftwidth=4    " indent width = 4 spaces
set hlsearch        " highlight search results
set incsearch       " incremental search
syntax on           " syntax highlighting
set ruler           " show cursor position
EOF
```

---

## 4.10 Viewing Binary Files

```bash
xxd file.bin | head -20      # hex dump
hexdump -C file.bin | head   # hex + ASCII
strings file.bin             # extract printable strings
```

---

## 4.11 Comparing Files

```bash
diff file1.txt file2.txt     # show differences
diff -u file1.txt file2.txt  # unified format (like git diff)
diff -r dir1/ dir2/          # compare directories

vimdiff file1.txt file2.txt  # visual diff inside vim
```

---

## Summary

| Command | Purpose |
|---------|---------|
| `cat file` | Print entire file |
| `less file` | Scrollable viewer |
| `head -n N file` | First N lines |
| `tail -n N file` | Last N lines |
| `tail -f file` | Follow file in real time |
| `nano file` | Easy GUI-like editor |
| `vim file` | Professional modal editor |
| `wc -l file` | Count lines |
| `diff f1 f2` | Compare files |

---

## Knowledge Check

1. What is the difference between `cat` and `less`?
2. How do you watch a log file update in real time?
3. In vim, what key sequence do you press to exit without saving?
4. What does `tail -n +100` do?
5. How do you search for the word "ERROR" inside `less`?

---

## Hands-On Exercise

```bash
# 1. Create a test file with 100 lines
seq 1 100 > /tmp/hundred.txt

# 2. View it with less (practice navigation)
less /tmp/hundred.txt
# Press G to go to end, gg to go to start, q to quit

# 3. See last 10 lines
tail /tmp/hundred.txt

# 4. See lines 45-55
tail -n +45 /tmp/hundred.txt | head -11

# 5. Simulate a live log (run in one terminal)
while true; do echo "$(date): Log entry $$"; sleep 1; done >> /tmp/live.log &
# In another terminal (or same terminal), follow it:
tail -f /tmp/live.log
# Press Ctrl+C to stop following
# Kill the background process:
kill %1

# 6. Practice vim (do this step by step)
vim /tmp/vim-practice.txt
# Type i to enter INSERT mode
# Type some text
# Press Esc to return to NORMAL mode
# Type :wq to save and quit

# 7. nano practice
nano /tmp/nano-practice.txt
# Type some text
# Ctrl+S to save
# Ctrl+X to exit
```

**Challenge:** Open `/var/log/syslog` (or `/var/log/kern.log`) in `less`, search for "error" (case-insensitive with `/\cerror`), and count how many matches exist.

---

## Further Reading

- `vimtutor` — run this command for an interactive vim tutorial (30 mins)
- `man less` — all less options and navigation
- [Vim Adventures](https://vim-adventures.com/) — learn vim through a game

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="03-file-and-directory-commands.md">← Previous: File & Directory Commands</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="05-text-processing.md">Next: Text Processing →</a>
</div>
