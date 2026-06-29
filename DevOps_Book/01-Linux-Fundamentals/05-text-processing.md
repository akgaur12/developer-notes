# Chapter 05 — Text Processing

## Learning Objectives

By the end of this chapter, you will:
- Search files and command output with `grep`
- Extract and transform data with `awk`
- Find and replace text with `sed`
- Slice columns with `cut`
- Sort and deduplicate with `sort` and `uniq`
- Count and measure with `wc`
- Build powerful text processing pipelines

## Prerequisites

- Chapter 04 — Text Viewing & Editing
- Understanding of pipes `|` (covered in Chapter 11, but we use them here — they pass output of one command to input of next)

---

## 5.1 Why Text Processing Is a Core DevOps Skill

In DevOps, you spend enormous time processing text:
- **Parsing logs** to find errors across millions of lines
- **Transforming data** from one format to another in scripts
- **Extracting values** from config files or API responses
- **Summarizing** metrics and counts from log files
- **Filtering** relevant records from large datasets

These commands — `grep`, `awk`, `sed` — are the Swiss Army knife of Linux text processing. They're used daily in scripts, pipelines, and one-liners.

---

## 5.2 `grep` — Global Regular Expression Print

`grep` searches for patterns in text. It's the most-used text tool in DevOps.

### Basic Syntax

```bash
grep "pattern" file
grep "pattern" file1 file2
command | grep "pattern"
```

### Basic Usage

```bash
grep "error" /var/log/syslog        # find lines containing "error"
grep "Failed" /var/log/auth.log     # find failed logins
grep "nginx" /etc/hosts             # find nginx in hosts file
ps aux | grep "python"              # find python processes
```

### Essential `grep` Flags

| Flag | Meaning |
|------|---------|
| `-i` | Case-insensitive |
| `-v` | Invert — lines that do NOT match |
| `-n` | Show line numbers |
| `-c` | Count matching lines (not show them) |
| `-l` | Show only filenames that match |
| `-L` | Show filenames that do NOT match |
| `-r` | Recursive search in directories |
| `-R` | Recursive, follow symlinks |
| `-w` | Match whole words only |
| `-x` | Match whole line only |
| `-A N` | Show N lines After match |
| `-B N` | Show N lines Before match |
| `-C N` | Show N lines of Context (before + after) |
| `-E` | Extended regex (same as `egrep`) |
| `-P` | Perl-compatible regex |
| `--color` | Highlight matches |
| `-o` | Print only matched part, not whole line |

### Practical grep Examples

```bash
# Case-insensitive search
grep -i "error" /var/log/syslog

# Show line numbers
grep -n "server_name" /etc/nginx/nginx.conf

# Show context around matches (3 lines before and after)
grep -C 3 "FATAL" /var/log/app.log

# Count how many errors
grep -c "ERROR" /var/log/app.log

# Search recursively in all config files
grep -r "max_connections" /etc/postgresql/

# Find files containing "password" (security audit)
grep -rl "password" /etc/ 2>/dev/null

# Lines NOT containing "DEBUG"
grep -v "DEBUG" /var/log/app.log

# Find processes (exclude the grep process itself)
ps aux | grep "nginx" | grep -v "grep"

# Search for whole word "log" (not "syslog" or "login")
grep -w "log" /etc/syslog.conf
```

### `grep` with Regular Expressions

```bash
# Basic regex in grep (anchors, character classes)
grep "^ERROR" logfile.txt       # lines STARTING with ERROR
grep "error$" logfile.txt       # lines ENDING with error
grep "^$" logfile.txt           # empty lines
grep "[0-9]\{3\}" logfile.txt   # exactly 3 digits

# Extended regex with -E (or egrep)
grep -E "error|warning|fatal" /var/log/app.log    # OR
grep -E "^(ERROR|WARN)" logfile.txt
grep -E "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}" logfile.txt  # IP addresses
grep -E "https?://" logfile.txt   # http or https URLs
```

### Real DevOps Use Cases

```bash
# Find all SSH login failures
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn | head

# Find all HTTP 500 errors in nginx log
grep " 500 " /var/log/nginx/access.log

# Count requests per status code
grep -oE " [0-9]{3} " /var/log/nginx/access.log | sort | uniq -c

# Find config files modified in the last 24 hours
find /etc -newer /etc/hostname -type f | xargs grep -l "changed"

# Grep with timestamp context
grep -A 5 "OutOfMemoryError" /var/log/app.log | head -30
```

---

## 5.3 `awk` — Pattern Scanning and Processing Language

`awk` is a **mini programming language** for processing columnar text data. It processes files line by line, splitting each line into fields.

### How awk Thinks

```
Input line:  "John   25   Engineer   5000"
awk splits:   $1=$John  $2=25  $3=Engineer  $4=5000  $NF=last field
```

### Basic Syntax

```bash
awk 'pattern { action }' file
awk '{ print $1 }' file          # print first field of every line
awk '{ print $1, $3 }' file      # print fields 1 and 3
```

### Built-in Variables

| Variable | Meaning |
|----------|---------|
| `$0` | Entire current line |
| `$1, $2, ...` | Field 1, 2, etc. |
| `$NF` | Last field |
| `NR` | Number of current record (line number) |
| `NF` | Number of fields in current line |
| `FS` | Field separator (default: whitespace) |
| `OFS` | Output field separator |
| `RS` | Record separator (default: newline) |

### Essential awk Examples

```bash
# Print specific columns
ls -la | awk '{ print $9, $5 }'         # name and size
ps aux | awk '{ print $1, $11 }'        # user and command

# Custom field separator (for CSV, /etc/passwd)
awk -F: '{ print $1 }' /etc/passwd      # print just usernames
awk -F: '{ print $1, $6 }' /etc/passwd  # username and home dir
awk -F, '{ print $1, $2 }' data.csv     # first two CSV columns

# Print lines matching a pattern
awk '/error/ { print }' logfile.txt
awk '/ERROR|WARN/ { print NR, $0 }' logfile.txt   # with line numbers

# Conditional logic
awk '$3 > 1000 { print $1, $3 }' data.txt         # where column 3 > 1000
awk 'NR > 5 && NR < 20 { print }' file.txt        # lines 6-19
awk '$1 == "root" { print }' /etc/passwd           # root entries

# Math operations
awk '{ sum += $4 } END { print "Total:", sum }' data.txt
awk '{ count++ } END { print "Lines:", count }' file.txt
awk 'NR == 1 { next } { sum += $2 } END { print sum/(NR-1) }' data.csv  # average, skip header

# BEGIN and END blocks
awk 'BEGIN { print "Start" } { print NR, $0 } END { print "Done" }' file.txt
```

### Real DevOps Use Cases

```bash
# Extract IP addresses from nginx access log
awk '{ print $1 }' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Calculate average response time from log
# Assume last field is response time in ms
awk '{ total += $NF; count++ } END { print "Average:", total/count "ms" }' /var/log/app.log

# Find top memory-consuming processes
ps aux | awk 'NR > 1 { print $4, $11 }' | sort -rn | head -10

# Extract usernames who logged in
last | awk '{ print $1 }' | sort | uniq -c | sort -rn

# Parse /etc/passwd for non-system users (UID >= 1000)
awk -F: '$3 >= 1000 { print $1, $3, $6 }' /etc/passwd

# Count HTTP methods in access log
awk '{ print $6 }' /var/log/nginx/access.log | tr -d '"' | sort | uniq -c

# Print lines where disk usage exceeds 80%
df -h | awk '$5 > 80 { print "ALERT:", $1, $5 }'
```

### awk Scripts (Multi-line)

For complex logic, write awk scripts:

```bash
cat > analyze.awk << 'EOF'
BEGIN {
    errors = 0
    warnings = 0
    total = 0
}

/ERROR/ { errors++ }
/WARN/  { warnings++ }
        { total++ }

END {
    print "Total lines:", total
    print "Errors:", errors
    print "Warnings:", warnings
    printf "Error rate: %.2f%%\n", (errors/total)*100
}
EOF

awk -f analyze.awk /var/log/app.log
```

---

## 5.4 `sed` — Stream Editor

`sed` is a stream editor — it reads input, applies transformations, and writes output. It's perfect for find-and-replace, line deletion, and insertion in scripts.

### Basic Syntax

```bash
sed 'command' file
sed 's/old/new/' file           # substitute first occurrence per line
sed 's/old/new/g' file          # substitute ALL occurrences (g = global)
sed -i 's/old/new/g' file       # in-place edit (modify file directly)
sed -i.bak 's/old/new/g' file   # in-place with backup (.bak file created)
```

### The Substitute Command `s/`

```bash
# Basic substitution
sed 's/hello/Hello/' file.txt           # first match per line
sed 's/http/https/g' urls.txt           # all occurrences per line
sed 's/  */ /g' file.txt                # collapse multiple spaces to one
sed 's/[[:space:]]*$//' file.txt        # remove trailing whitespace
sed 's/^[[:space:]]*//' file.txt        # remove leading whitespace

# Case-insensitive substitution
sed 's/error/ERROR/gi' log.txt

# Use different delimiter when pattern has /
sed 's|/old/path|/new/path|g' file.txt
sed 's:/old/path:/new/path:g' file.txt

# Backreferences (capture groups)
sed 's/\(first\) \(second\)/\2 \1/' file.txt  # swap words
sed -E 's/(first) (second)/\2 \1/' file.txt   # extended regex (cleaner)
```

### Deleting and Printing Lines

```bash
sed '5d' file.txt               # delete line 5
sed '5,10d' file.txt            # delete lines 5-10
sed '/pattern/d' file.txt       # delete lines matching pattern
sed '/^#/d' config.txt          # delete comment lines
sed '/^$/d' file.txt            # delete empty lines

sed -n '5p' file.txt            # print only line 5 (-n suppresses default print)
sed -n '5,10p' file.txt         # print lines 5-10
sed -n '/START/,/END/p' file    # print between markers
```

### Inserting and Appending

```bash
sed '5i\New line before line 5' file.txt    # insert before line 5
sed '5a\New line after line 5' file.txt     # append after line 5
sed '/pattern/a\New line after match' file.txt
```

### Multiple sed Commands

```bash
# Apply multiple substitutions
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt

# Or with semicolons
sed 's/foo/bar/g; s/baz/qux/g' file.txt
```

### Real DevOps Use Cases

```bash
# Update a config value in place
sed -i 's/^max_connections = .*/max_connections = 200/' /etc/postgresql/postgresql.conf

# Change port in nginx config
sed -i 's/listen 80/listen 8080/' /etc/nginx/sites-available/default

# Remove comments and empty lines from config
sed '/^#/d; /^$/d' /etc/nginx/nginx.conf

# Add http:// to all plain domains in a list
sed 's|^|http://|' domains.txt

# Replace environment variable in a template
sed "s/{{APP_PORT}}/$APP_PORT/g" config.template > config.final

# Extract version from a file
sed -n 's/^version = "\(.*\)"/\1/p' pyproject.toml

# Backup and modify in one step
sed -i.bak 's/debug = true/debug = false/g' app.config
```

---

## 5.5 `cut` — Extract Fields/Columns

`cut` extracts portions of lines — by character position or field delimiter.

```bash
cut -d: -f1 /etc/passwd          # field 1 (delimiter = colon)
cut -d: -f1,3 /etc/passwd        # fields 1 and 3
cut -d, -f2 data.csv             # second column of CSV
cut -c1-10 file.txt              # characters 1-10 of each line
cut -c1,5,10 file.txt            # characters at positions 1, 5, 10

# Examples
echo "a:b:c:d" | cut -d: -f2    # prints: b
echo "2026-06-23" | cut -d- -f1 # prints: 2026 (year)
```

---

## 5.6 `sort` — Sort Lines

```bash
sort file.txt               # alphabetical sort
sort -r file.txt            # reverse sort
sort -n file.txt            # numeric sort
sort -nr file.txt           # numeric reverse
sort -k2 file.txt           # sort by second field
sort -k2n file.txt          # sort by second field numerically
sort -t: -k3n /etc/passwd   # sort by UID (field 3, delimiter :)
sort -u file.txt            # sort and remove duplicates
sort -h file.txt            # human-readable sort (1K, 2M, 3G)
```

```bash
# Sort processes by memory usage
ps aux | sort -k4rn | head -10

# Sort IP addresses (weird but works with -V)
sort -V ip_list.txt

# Sort by multiple keys
sort -k1,1 -k2,2n file.txt    # sort by field 1, then field 2 numerically
```

---

## 5.7 `uniq` — Remove/Count Duplicates

`uniq` works on **adjacent** duplicates, so almost always used after `sort`:

```bash
sort file.txt | uniq          # remove duplicates
sort file.txt | uniq -c       # count occurrences (most useful!)
sort file.txt | uniq -d       # show only duplicates
sort file.txt | uniq -u       # show only lines that appear once
```

```bash
# Top 10 IP addresses hitting your server
awk '{ print $1 }' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Count unique values in column 2
awk '{ print $2 }' data.txt | sort | uniq -c | sort -rn
```

---

## 5.8 Building Pipelines — The Real Power

The magic of these tools is combining them with pipes (`|`):

```bash
# Pipeline: read | filter | transform | sort | limit
cat /var/log/auth.log \
  | grep "Failed password" \
  | awk '{ print $11 }' \
  | sort \
  | uniq -c \
  | sort -rn \
  | head -5
# Result: Top 5 IPs brute-forcing SSH

# Count HTTP status codes in nginx log
awk '{ print $9 }' /var/log/nginx/access.log \
  | sort \
  | uniq -c \
  | sort -rn
# Result:
# 15234  200
#  1423  404
#   234  301
#    43  500

# Find most active users
last | awk '{ print $1 }' | grep -v "^$" | sort | uniq -c | sort -rn | head

# Disk usage report: top 10 largest directories
du -sh /var/* 2>/dev/null | sort -rh | head -10

# Extract all unique domains from a log file
grep -oE '[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' logfile.txt | sort | uniq
```

---

## 5.9 Other Useful Text Tools

### `tr` — Translate Characters

```bash
echo "hello WORLD" | tr '[:upper:]' '[:lower:]'   # lowercase
echo "hello" | tr 'a-z' 'A-Z'                     # uppercase
echo "a:b:c" | tr ':' '\n'                         # replace : with newline
echo "hello   world" | tr -s ' '                   # squeeze spaces
echo "hello" | tr -d 'l'                           # delete characters
```

### `tee` — Write to File AND Pass to Next Command

```bash
cat /var/log/syslog | tee /tmp/syslog-copy.txt | grep "error"
# Both: writes to file AND filters for "error"

# Append
command | tee -a logfile.txt | next_command
```

### `xargs` — Build and Execute Commands from Input

```bash
# Pass stdin as arguments
find . -name "*.log" | xargs grep "ERROR"
find . -name "*.tmp" | xargs rm
cat servers.txt | xargs -I{} ssh {} "uptime"     # SSH to each server

# With null delimiter (safe for filenames with spaces)
find . -name "*.txt" -print0 | xargs -0 cat
```

---

## Summary

| Command | Primary Use |
|---------|-------------|
| `grep "pattern" file` | Find lines matching pattern |
| `grep -r "pattern" dir/` | Recursive search |
| `awk '{ print $2 }' file` | Extract columns |
| `awk -F: '{ print $1 }' file` | Custom delimiter column extract |
| `sed 's/old/new/g' file` | Find and replace |
| `sed -i 's/old/new/g' file` | In-place replacement |
| `cut -d: -f1 file` | Cut field by delimiter |
| `sort -rn file` | Numeric reverse sort |
| `uniq -c` | Count unique occurrences |

---

## Knowledge Check

1. How do you search for "error" case-insensitively and show 3 lines of context?
2. What does `awk -F: '{ print $1, $3 }' /etc/passwd` do?
3. How do you replace all occurrences of "foo" with "bar" in a file, in-place?
4. What is the difference between `sort | uniq` and just `sort -u`?
5. Write a one-liner to find the 5 most common IP addresses in an nginx access log.

---

## Hands-On Exercise

```bash
# Setup: create sample data
cat > /tmp/access.log << 'EOF'
192.168.1.1 - - [23/Jun/2026:10:00:01] "GET /index.html HTTP/1.1" 200 1234
10.0.0.5 - - [23/Jun/2026:10:00:02] "POST /api/login HTTP/1.1" 401 56
192.168.1.1 - - [23/Jun/2026:10:00:03] "GET /about.html HTTP/1.1" 200 2345
172.16.0.1 - - [23/Jun/2026:10:00:04] "GET /missing.html HTTP/1.1" 404 78
192.168.1.1 - - [23/Jun/2026:10:00:05] "GET /index.html HTTP/1.1" 200 1234
10.0.0.5 - - [23/Jun/2026:10:00:06] "POST /api/login HTTP/1.1" 401 56
10.0.0.5 - - [23/Jun/2026:10:00:07] "POST /api/login HTTP/1.1" 401 56
EOF

# Exercise 1: Find all 401 lines
grep " 401 " /tmp/access.log

# Exercise 2: Count how many 401 responses
grep -c " 401 " /tmp/access.log

# Exercise 3: Extract just the IP addresses
awk '{ print $1 }' /tmp/access.log

# Exercise 4: Count requests per IP
awk '{ print $1 }' /tmp/access.log | sort | uniq -c | sort -rn

# Exercise 5: Extract all HTTP status codes and count them
awk '{ print $9 }' /tmp/access.log | sort | uniq -c

# Exercise 6: Use sed to anonymize IP addresses
sed 's/^[0-9.]*/HIDDEN/' /tmp/access.log

# Exercise 7: Find lines with GET requests only
grep '"GET ' /tmp/access.log

# Exercise 8: Extract request paths only
awk '{ print $7 }' /tmp/access.log | sort | uniq -c | sort -rn
```

**Challenge:** Write a pipeline that outputs: the IP that made the most requests, how many requests it made, and all the URLs it requested.

---

## Further Reading

- `man grep`, `man awk`, `man sed` — the definitive references
- [awk Tutorial](https://www.grymoire.com/Unix/Awk.html) — comprehensive awk guide
- [sed Tutorial](https://www.grymoire.com/Unix/Sed.html) — comprehensive sed guide

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="04-text-viewing-and-editing.md">← Previous: Text Viewing & Editing</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="06-permissions-and-ownership.md">Next: Permissions & Ownership →</a>
</div>
