# Chapter 11 — Pipes, Redirects & Environment

## Learning Objectives

By the end of this chapter, you will:
- Master I/O redirection: stdin, stdout, stderr
- Build powerful command pipelines with `|`
- Use `tee`, `xargs`, and process substitution
- Manage environment variables and the PATH
- Create and use aliases and shell functions
- Understand and customize dotfiles

## Prerequisites

- Chapters 01–10

---

## 11.1 The Three Streams: stdin, stdout, stderr

Every Linux process has three standard file descriptors:

```
File Descriptor 0 = stdin   (standard input)  ← keyboard by default
File Descriptor 1 = stdout  (standard output) → terminal by default
File Descriptor 2 = stderr  (standard error)  → terminal by default
```

```
           stdin (0)
               ↓
┌─────────────────────────┐
│        Program          │ → stdout (1) → terminal / file
│     (ls, grep, etc.)    │ → stderr (2) → terminal / file
└─────────────────────────┘
```

---

## 11.2 Output Redirection

```bash
# Redirect stdout to file
ls -la > output.txt          # overwrite
ls -la >> output.txt         # append

# Redirect stderr to file
command 2> errors.txt

# Redirect both stdout and stderr to same file
command > all.log 2>&1       # traditional
command &> all.log           # bash shorthand

# Redirect stdout and stderr to different files
command > output.txt 2> errors.txt

# Discard output
command > /dev/null           # discard stdout
command 2> /dev/null          # discard stderr
command &> /dev/null          # discard everything

# Redirect stdout to stderr (useful in scripts)
echo "This is an error" >&2
```

### Understanding `2>&1`

`2>&1` means "redirect file descriptor 2 (stderr) to wherever fd 1 (stdout) currently points":

```bash
# CORRECT order
command > file.txt 2>&1
# 1. stdout → file.txt
# 2. stderr → wherever stdout goes → file.txt

# WRONG order
command 2>&1 > file.txt
# 1. stderr → wherever stdout goes → terminal (not file!)
# 2. stdout → file.txt
```

---

## 11.3 Input Redirection

```bash
# Redirect stdin from file
command < input.txt
sort < names.txt
wc -l < /var/log/syslog

# Here Document (heredoc) — multi-line stdin
cat << EOF
Line 1
Line 2
Variable: $USER
EOF

# Here String — single string as stdin
grep "pattern" <<< "This is a string to search"
wc -w <<< "count these words"
```

---

## 11.4 Pipes — The Power of Composition

A pipe `|` connects stdout of one command to stdin of the next:

```bash
command1 | command2 | command3
```

### Building Pipelines

```bash
# Simple pipeline
ls -la | grep ".txt"

# Multi-step pipeline
cat /var/log/auth.log \
  | grep "Failed password" \
  | awk '{print $11}' \
  | sort \
  | uniq -c \
  | sort -rn \
  | head -5

# Pipeline with error handling
ps aux | grep nginx | grep -v grep

# Count lines matching a pattern
grep -c "ERROR" /var/log/app.log
cat /var/log/app.log | grep "ERROR" | wc -l   # same result, pipeline style
```

### `tee` — Split Pipeline Output

`tee` writes to both a file and passes output to the next command:

```bash
command | tee file.txt | next_command
command | tee -a file.txt | next_command    # append mode

# Log AND process
cat /var/log/app.log | tee /tmp/app_copy.log | grep "ERROR"

# Log to file and view on screen simultaneously
./deploy.sh 2>&1 | tee deploy.log
```

---

## 11.5 `xargs` — Build Commands from Input

`xargs` takes lines from stdin and passes them as arguments to a command:

```bash
# Delete files found by find
find . -name "*.tmp" | xargs rm
find . -name "*.tmp" -delete    # equivalent

# Process each file
find . -name "*.log" | xargs wc -l     # line count of each log

# xargs with placeholder
find . -name "*.bak" | xargs -I{} mv {} /backup/
cat servers.txt | xargs -I{} ssh {} "uptime"

# Parallel execution
find . -name "*.py" | xargs -P4 -I{} python3 -c "import ast; ast.parse(open('{}').read())"
# -P4 = run 4 processes in parallel

# Handle filenames with spaces (use -0 with find -print0)
find . -name "*.txt" -print0 | xargs -0 grep "pattern"
```

---

## 11.6 Process Substitution

Process substitution treats a command's output as if it were a file:

```bash
# <(command) — treats output as input file
diff <(sort file1.txt) <(sort file2.txt)
comm <(sort users.txt) <(sort admins.txt)

# Compare two remote hosts' configs
diff <(ssh server1 cat /etc/nginx/nginx.conf) <(ssh server2 cat /etc/nginx/nginx.conf)

# >(command) — treats input as a file going to command
command > >(tee logfile.txt)
./script.sh > >(tee script.log) 2> >(tee script.err >&2)
```

---

## 11.7 Environment Variables

Environment variables are key-value pairs passed to every process:

```bash
# View all environment variables
env
printenv
printenv PATH          # view specific variable

# Set variable (current session only)
export MY_VAR="hello"

# Unset variable
unset MY_VAR

# Temporary variable (for one command only)
MY_VAR=hello ./script.sh

# Use variable
echo $MY_VAR
echo ${MY_VAR}         # explicit (better in scripts)
```

### Important Environment Variables

| Variable | Purpose |
|----------|---------|
| `PATH` | Directories to search for executables |
| `HOME` | User's home directory |
| `USER` / `LOGNAME` | Current username |
| `SHELL` | Current shell |
| `EDITOR` | Default text editor |
| `TERM` | Terminal type |
| `LANG` / `LC_ALL` | Locale and character encoding |
| `HISTFILE` | History file location |
| `HISTSIZE` | Commands to remember in memory |
| `PS1` | Shell prompt string |
| `PYTHONPATH` | Python module search path |
| `JAVA_HOME` | Java installation directory |

### The `PATH` Variable

`PATH` tells the shell where to find executable commands:

```bash
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games

# Add a directory to PATH
export PATH="$HOME/.local/bin:$PATH"        # prepend (higher priority)
export PATH="$PATH:/opt/custom/bin"          # append (lower priority)

# Verify a command location
which python3            # /usr/bin/python3
which kubectl            # /usr/local/bin/kubectl
type kubectl             # shows type (built-in, alias, or file)

# PATH search order: first match wins
# If you have two 'python' binaries, the one in the first PATH dir wins
```

### Persisting Environment Variables

```bash
# For a single user: add to ~/.bashrc
echo 'export JAVA_HOME="/opt/java-17"' >> ~/.bashrc
echo 'export PATH="$JAVA_HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc    # reload

# For all users: /etc/environment or /etc/profile.d/
echo 'JAVA_HOME="/opt/java-17"' | sudo tee -a /etc/environment
sudo cat > /etc/profile.d/java.sh << 'EOF'
export JAVA_HOME="/opt/java-17"
export PATH="$JAVA_HOME/bin:$PATH"
EOF
```

---

## 11.8 Aliases and Shell Functions

### Aliases

Aliases are command shortcuts:

```bash
# Define alias (current session)
alias ll='ls -la'
alias la='ls -la'
alias gs='git status'
alias gc='git commit'
alias dc='docker-compose'
alias k='kubectl'
alias ..='cd ..'
alias ...='cd ../..'
alias grep='grep --color=auto'
alias df='df -h'
alias du='du -h'
alias free='free -h'
alias cp='cp -iv'    # interactive + verbose
alias mv='mv -iv'    # interactive + verbose

# List all aliases
alias

# Remove alias
unalias ll

# Temporarily bypass alias (use backslash)
\ls                  # runs actual ls, not the alias
```

### Persisting Aliases in `.bashrc`

```bash
cat >> ~/.bashrc << 'EOF'

# ─── DevOps Aliases ───────────────────────────────────────────
alias ll='ls -laF --color=auto'
alias l='ls -CF'
alias ..='cd ..'
alias ...='cd ../..'
alias grep='grep --color=auto'
alias df='df -h'
alias du='du -sh'
alias free='free -h'

# Git shortcuts
alias gs='git status'
alias ga='git add'
alias gc='git commit'
alias gp='git push'
alias gl='git log --oneline --graph --decorate'

# Docker shortcuts
alias dc='docker-compose'
alias dps='docker ps'
alias di='docker images'

# Kubernetes shortcuts
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
EOF

source ~/.bashrc
```

### Shell Functions (More Powerful than Aliases)

```bash
# Add to ~/.bashrc

# cd + ls: list directory after entering it
cdl() {
    cd "$1" && ls -la
}

# Create and enter directory
mkcd() {
    mkdir -p "$1" && cd "$1"
}

# Find and kill process by name
fkill() {
    local pid
    pid=$(pgrep -f "$1")
    if [[ -z "$pid" ]]; then
        echo "No process matching: $1"
    else
        echo "Killing: $pid ($1)"
        kill "$pid"
    fi
}

# Backup a file with timestamp
backup() {
    cp "$1" "${1}.bak.$(date +%Y%m%d_%H%M%S)"
}

# Quickly edit and reload bashrc
reload() {
    ${EDITOR:-vim} ~/.bashrc
    source ~/.bashrc
    echo "~/.bashrc reloaded"
}

# Extract any archive
extract() {
    case "$1" in
        *.tar.bz2)  tar xjf "$1"    ;;
        *.tar.gz)   tar xzf "$1"    ;;
        *.tar.xz)   tar xJf "$1"    ;;
        *.bz2)      bunzip2 "$1"     ;;
        *.gz)       gunzip "$1"      ;;
        *.tar)      tar xf "$1"     ;;
        *.zip)      unzip "$1"      ;;
        *.7z)       7z x "$1"       ;;
        *)          echo "Unknown format: $1" ;;
    esac
}
```

---

## 11.9 Dotfiles — Your Configuration Files

Dotfiles are hidden config files in your home directory (start with `.`):

```bash
ls -la ~ | grep "^\."
# .bashrc        bash config
# .bash_profile  bash login config
# .bash_history  command history
# .vimrc         vim config
# .gitconfig     git config
# .ssh/          SSH keys and config
# .profile       POSIX shell config
```

### Important Dotfiles

**`~/.bashrc`** — loaded for every interactive bash session:
```bash
# Prompt customization, aliases, functions, environment vars
PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
```

**`~/.bash_profile`** — loaded for login shells (SSH sessions):
```bash
# Usually just sources .bashrc:
[[ -f ~/.bashrc ]] && source ~/.bashrc
```

**`~/.vimrc`** — vim configuration (from Chapter 04)

**`~/.gitconfig`** — git configuration:
```bash
git config --global user.name "Akash Gaur"
git config --global user.email "akash@example.com"
git config --global core.editor vim
git config --global alias.st status
cat ~/.gitconfig
```

**`~/.ssh/config`** — SSH shortcuts (from Chapter 08)

### Managing Dotfiles with Git

A popular DevOps practice: store dotfiles in git so they're version-controlled and reproducible:

```bash
mkdir ~/dotfiles
cp ~/.bashrc ~/dotfiles/
cp ~/.vimrc ~/dotfiles/
cd ~/dotfiles
git init
git add .
git commit -m "Initial dotfiles"
# Push to GitHub for backup and reuse on new servers
```

---

## 11.10 Command Substitution

Execute a command and use its output as a value:

```bash
# $() syntax (recommended)
today=$(date +%Y-%m-%d)
hostname=$(hostname -f)
user_count=$(wc -l < /etc/passwd)

# Backtick syntax (legacy — avoid)
today=`date +%Y-%m-%d`   # same but less readable

# Nested substitution
echo "Kernel: $(uname -r)"
echo "Python: $(python3 --version 2>&1 | awk '{print $2}')"
echo "Free RAM: $(free -m | awk 'NR==2{print $4}') MB"

# In strings
backup_file="backup_${today}.tar.gz"
echo "Creating $backup_file"
```

---

## Summary

```
Redirection:
  command > file     stdout to file (overwrite)
  command >> file    stdout to file (append)
  command 2> file    stderr to file
  command &> file    stdout+stderr to file
  command < file     stdin from file

Pipes:
  cmd1 | cmd2        stdout of cmd1 → stdin of cmd2
  cmd1 | tee f | cmd2  split: file AND next command

Environment:
  export VAR=value   set and export variable
  $PATH              executable search path
  alias x='...'      command shortcut
```

---

## Knowledge Check

1. What is the difference between `>` and `>>`?
2. What does `2>&1` mean?
3. How would you both display output on screen AND save it to a file?
4. What does `export` do that just setting `VAR=value` doesn't?
5. What is the difference between an alias and a shell function?

---

## Hands-On Exercise

```bash
# 1. Redirect practice
ls /etc 2>/dev/null | head -10          # suppress errors
ls /nonexistent 2>/tmp/err.txt           # redirect error
ls /etc /nonexistent > /tmp/out.txt 2>&1 # combine both

# 2. Pipeline building
# Build step by step — add one piece at a time
ps aux                                        # start
ps aux | awk 'NR>1 {print $1, $11}'          # extract user + command
ps aux | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn  # count by user

# 3. Set environment variables
export MY_APP_PORT=8080
export MY_APP_ENV=development
env | grep MY_APP    # verify

# 4. Add to .bashrc
cat >> ~/.bashrc << 'EOF'
alias ll='ls -laF'
alias ..='cd ..'
cdl() { cd "$1" && ls -la; }
EOF
source ~/.bashrc
ll              # test alias
cdl /etc        # test function

# 5. Process substitution
diff <(ls /bin) <(ls /usr/bin) | head -20   # compare two dir listings

# 6. tee usage
df -h | tee /tmp/disk-report.txt
cat /tmp/disk-report.txt
```

**Challenge:** Write a one-liner that: finds all `.log` files in `/var/log`, gets their sizes, and outputs a sorted report showing total size per file, largest first.

---

## Further Reading

- `man bash` — search for "REDIRECTION"
- [Bash Redirection Tutorial](https://www.gnu.org/savannah-checkouts/gnu/bash/manual/bash.html#Redirections)

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="10-users-and-system-administration.md">← Previous: Users & System Administration</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="12-advanced-concepts.md">Next: Advanced Linux Concepts →</a>
</div>
