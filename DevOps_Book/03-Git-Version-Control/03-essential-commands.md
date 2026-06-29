# Chapter 03 — Essential Git Commands

## Learning Objectives

By the end of this chapter, you will be able to:

- Initialize a new repository and understand what `git init` creates
- Clone remote repositories with fine-grained control (shallow, single-branch, specific branch)
- Read `git status` output fluently, including the short format
- Stage changes precisely using `git add` with file paths, directories, and interactive mode
- Interpret `git diff` in multiple modes (unstaged, staged, between commits)
- Craft commits with `git commit`, including amending the last commit
- Navigate and filter `git log` with a wide range of flags
- Inspect individual commits and objects with `git show`
- Remove and rename tracked files properly with `git rm` and `git mv`
- Write effective `.gitignore` files and debug ignored files with `git check-ignore`
- List tracked files with `git ls-files`

---

## 3.1 git init

`git init` creates a new, empty Git repository in the current directory (or a specified directory).

```bash
# Initialize a repo in the current directory
git init

# Initialize a repo in a new directory
git init my-project

# Initialize a bare repository (no working tree — used for servers/remotes)
git init --bare project.git
```

What `git init` creates:

```
Initialized empty Git repository in /home/user/my-project/.git/
```

The `.git` directory is created with the structure described in Chapter 02. The working directory itself is not touched. At this point there are no commits, no branches (technically), and no tracked files. The first branch (`main`) will be created when you make the first commit.

**Bare repositories** — A bare repo has no working tree. It contains only the contents of `.git/`. These are used as shared remotes (e.g., what you push to on a server). You cannot `git add` or edit files in a bare repo directly.

---

## 3.2 git clone

`git clone` creates a local copy of an existing repository, including all commits, branches, and tags.

```bash
# Clone from GitHub (HTTPS)
git clone https://github.com/username/repo.git

# Clone from GitHub (SSH — requires SSH key setup)
git clone git@github.com:username/repo.git

# Clone into a custom directory name
git clone https://github.com/username/repo.git my-local-name

# Clone a specific branch
git clone --branch develop https://github.com/username/repo.git

# Clone only that branch (do not download other branches)
git clone --branch develop --single-branch https://github.com/username/repo.git

# Shallow clone: only the latest commit (fast, minimal download)
git clone --depth 1 https://github.com/username/repo.git

# Shallow clone of a specific branch
git clone --depth 1 --branch main https://github.com/username/repo.git
```

### Shallow Clones vs. Full Clones

| | Full Clone | Shallow Clone (`--depth 1`) |
|---|---|---|
| Downloads | Entire history | Only the latest snapshot |
| Disk usage | Full | Minimal |
| Use case | Development, contribution | CI builds, deployments |
| Can push back? | Yes | Limited (need to unshallow first) |
| `git log` depth | Complete | Limited to fetched commits |

After cloning, Git automatically:
- Creates a remote named `origin` pointing to the source URL
- Creates a local branch tracking the default remote branch
- Sets `HEAD` to the default branch

```bash
# Verify the remote URL after cloning
git remote -v
# origin  https://github.com/username/repo.git (fetch)
# origin  https://github.com/username/repo.git (push)
```

---

## 3.3 git status

`git status` shows the state of the working tree and staging area relative to the last commit.

```bash
git status
```

### Reading Full Status Output

```
On branch main
Your branch is up to date with 'origin/main'.

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   src/auth.go          ← Staged: will go into next commit
        new file:   src/token.go         ← Staged: new file

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   README.md            ← Modified but NOT staged

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        config/local.yaml                ← Git has never seen this file
```

### Short Status Format

```bash
git status -s
# or
git status --short
```

Output format: two columns. The **left column** is the staging area status; the **right column** is the working tree status.

```
M  src/auth.go       ← Left M: staged modification
A  src/token.go      ← Left A: staged new file (Added)
 M README.md         ← Right M: unstaged modification
?? config/local.yaml ← ??: untracked file
```

| Code | Meaning |
|---|---|
| `M` | Modified |
| `A` | Added (new file, staged) |
| `D` | Deleted |
| `R` | Renamed |
| `C` | Copied |
| `U` | Updated but unmerged (conflict) |
| `?` | Untracked |
| `!` | Ignored |

Left column = staging area state. Right column = working tree state.

---

## 3.4 git add

`git add` moves changes from the working tree into the staging area.

```bash
# Stage a specific file
git add README.md

# Stage all files in a specific directory
git add src/

# Stage all changes in the current directory (and subdirectories)
git add .

# Stage all tracked modified files AND new files everywhere in the repo
git add -A
# (equivalent to: git add --all)

# Stage only modifications and deletions — NOT new untracked files
git add -u
```

### Interactive Staging with -p (Patch Mode)

One of Git's most powerful features: stage individual *hunks* (sections) of a file rather than the whole file.

```bash
git add -p README.md
# or
git add --patch README.md
```

Git shows each changed hunk one at a time and asks what to do:

```diff
diff --git a/README.md b/README.md
index a1b2c3..d4e5f6 100644
--- a/README.md
+++ b/README.md
@@ -10,3 +10,6 @@
 ## Installation
+
+Run `npm install` to install dependencies.
+Then run `npm start` to launch the server.

Stage this hunk [y,n,q,a,d,s,?]?
```

| Key | Action |
|---|---|
| `y` | Stage this hunk |
| `n` | Do not stage this hunk |
| `s` | Split the hunk into smaller pieces |
| `e` | Manually edit the hunk |
| `q` | Quit; do not stage this or any remaining hunks |
| `a` | Stage this and all remaining hunks in the file |
| `d` | Do not stage this or any remaining hunks in the file |
| `?` | Print help |

This allows you to make multiple unrelated changes to a file and commit them as separate, logical commits — a practice that keeps history clean and meaningful.

---

## 3.5 git diff

`git diff` shows the changes between two states. The specific comparison depends on the flags used.

### Working Tree vs. Staging Area (default)

Shows what you have changed but have not yet staged:

```bash
git diff
```

```diff
diff --git a/README.md b/README.md
index a1b2c3..d4e5f6 100644
--- a/README.md
+++ b/README.md
@@ -5,6 +5,8 @@ # My Project
 
 ## Overview
 
+This project implements a REST API for user management.
+
 ## Installation
```

### Staged Changes vs. Last Commit

Shows what is staged and will be included in the next commit:

```bash
git diff --staged
# or
git diff --cached   (identical — two names for the same flag)
```

### Between Two Commits

```bash
# Diff between two specific commits
git diff abc123 def456

# Diff between current state and a specific commit
git diff HEAD~2

# Diff between two branches
git diff main..feature-login

# Changes in feature-login that are not in main (three-dot)
git diff main...feature-login
```

### Useful diff Options

```bash
# Show only filenames that changed (not the actual diff)
git diff --name-only

# Show filenames with change summary (lines added/removed)
git diff --stat

# Show word-level changes instead of line-level
git diff --word-diff

# Ignore whitespace changes
git diff -w
# or
git diff --ignore-all-space
```

### Reading a Diff

```diff
diff --git a/src/main.go b/src/main.go   ← files being compared
index 4b8a7c1..9d3f2e4 100644            ← blob hashes + permissions
--- a/src/main.go                         ← "before" file (a/)
+++ b/src/main.go                         ← "after" file (b/)
@@ -15,7 +15,9 @@ func main() {          ← hunk header: old start,count new start,count
     fmt.Println("Starting server")
-    port := 8080                         ← removed line (red)
+    port := getPort()                    ← added line (green)
+    log.Printf("Using port %d", port)   ← added line (green)
     server.Listen(port)
```

---

## 3.6 git commit

`git commit` creates a new commit from everything in the staging area.

```bash
# Open the configured editor for the commit message
git commit

# Provide the message inline
git commit -m "Add user authentication module"

# Multi-line message inline
git commit -m "Add user authentication module

Implements JWT-based auth with refresh token rotation.
Access tokens expire in 15 minutes; refresh tokens in 7 days.

Closes #42"
```

### Skipping git add for Tracked Files

```bash
# Stage all modifications to already-tracked files AND commit
git commit -am "Fix typo in error message"
```

The `-a` flag stages modifications and deletions to files Git already tracks. It does NOT stage new untracked files.

### Amending the Last Commit

```bash
# Change the commit message of the last commit
git commit --amend -m "Better commit message"

# Add a forgotten file to the last commit (without changing the message)
git add forgotten-file.go
git commit --amend --no-edit
```

> **Warning:** `--amend` rewrites history. The commit gets a new SHA-1. If you have already pushed the original commit to a shared remote, amending it and force-pushing will cause problems for anyone who has pulled that commit. Only amend commits that exist only on your local machine.

### Commit Message Best Practices

A well-formed commit message follows this structure:

```
Short summary in imperative mood (50 chars or less)

More detailed explanation if necessary. Wrap at 72 characters.
Explain WHY the change was made, not just what changed (the diff
already shows what changed).

- Bullet points are fine for lists
- Use present tense in the subject line: "Fix bug" not "Fixed bug"

Closes #123
```

---

## 3.7 git log

`git log` shows the commit history. With the right flags, it becomes an extremely powerful investigation tool.

```bash
# Basic log (newest first)
git log
```

### Essential Log Flags

```bash
# One commit per line (hash + message)
git log --oneline

# Show the graph structure (branches/merges)
git log --oneline --graph

# Show all branches, not just the current one
git log --oneline --graph --all

# Show full diff for each commit
git log -p

# Show summary of changed files for each commit
git log --stat

# Show only file names changed in each commit
git log --name-only
```

### Filtering the Log

```bash
# Filter by author
git log --author="Jane Doe"
git log --author="jane"    # partial match works

# Filter by date range
git log --since="2024-01-01"
git log --until="2024-12-31"
git log --since="2 weeks ago"
git log --after="yesterday"

# Filter by commit message keyword
git log --grep="authentication"
git log --grep="fix" -i    # -i = case-insensitive

# Filter by code change (pickaxe search) — find commits that added/removed a string
git log -S "password_hash"       # string search
git log -G "def.*auth"           # regex search

# Show only last N commits
git log -5
git log -n 10

# Filter by file path
git log -- src/auth.go
git log -- "*.go"

# Show commits in a range
git log main..feature-branch   # commits in feature-branch not in main
git log v1.0..v2.0             # commits between two tags
```

### Custom Log Format

```bash
# Completely custom format
git log --format="%h %an %ar %s"
# Output:
# 7ec0a5f Jane Doe 3 days ago Add user auth
# 4b8e2c1 Bob Smith 5 days ago Fix DB connection pool

# Useful format variables:
# %h   = abbreviated commit hash
# %H   = full commit hash
# %an  = author name
# %ae  = author email
# %ar  = author date, relative (e.g., "3 days ago")
# %ai  = author date, ISO 8601
# %s   = subject (first line of commit message)
# %b   = body (rest of commit message)

# Pretty preset formats
git log --pretty=oneline
git log --pretty=short
git log --pretty=full
git log --pretty=fuller
```

### The Ultimate Log Alias

Many developers add this to their global config:

```bash
git config --global alias.lg "log --oneline --graph --all --decorate"

# Now you can use:
git lg
```

---

## 3.8 git show

`git show` displays information about a specific Git object — most commonly a commit.

```bash
# Show the most recent commit (diff + metadata)
git show

# Show a specific commit
git show 7ec0a5f

# Show a commit by tag
git show v1.0.0

# Show what was changed in a specific file in a commit
git show 7ec0a5f:src/auth.go

# Show only the file contents at a specific commit (no diff metadata)
git show 7ec0a5f:README.md

# Show only the commit metadata without the diff
git show --no-patch 7ec0a5f

# Show only the names of changed files
git show --name-only 7ec0a5f
```

---

## 3.9 git rm

`git rm` removes a file from both the working tree AND the staging area (i.e., it removes the file from disk and tells Git to stop tracking it).

```bash
# Remove a file from disk and from Git tracking
git rm obsolete-file.txt
# You still need to commit this change

# Remove a directory recursively
git rm -r old-directory/

# Stop tracking a file but keep it on disk
# (useful when you forgot to add something to .gitignore)
git rm --cached config/secrets.yaml

# Force removal even if there are staged changes
git rm -f temp-file.txt
```

> **Note:** `git rm --cached` is critical for removing accidentally committed files (like `.env` or credentials). After running it, add the file to `.gitignore` and commit. The file will remain in older commits' history — see Chapter 12 for removing files from history entirely.

---

## 3.10 git mv

`git mv` renames or moves a file in a way that Git can track.

```bash
# Rename a file
git mv old-name.go new-name.go

# Move a file to a different directory
git mv src/helper.go lib/helper.go

# Rename a directory
git mv old-dir/ new-dir/
```

Under the hood, `git mv old new` is equivalent to:

```bash
mv old new
git rm old
git add new
```

Git does not actually store rename events in its object model — renames are detected heuristically when you run `git log --follow` or `git diff -M`.

```bash
# Follow a file's history across renames
git log --follow src/auth.go
```

---

## 3.11 .gitignore

`.gitignore` is a text file at the root of your repository (or in subdirectories) that tells Git which files to completely ignore — they will not appear in `git status`, `git add .`, etc.

### Pattern Syntax

```gitignore
# Lines starting with # are comments

# Ignore a specific file
secrets.yaml

# Ignore all files with a specific extension
*.log
*.tmp
*.pyc

# Ignore a specific directory (trailing slash means directory)
node_modules/
dist/
build/
.cache/

# Ignore files in a specific directory
logs/*.log

# Ignore all .txt files recursively
**/*.txt

# Negate a pattern (do NOT ignore this file, even if a previous rule would)
!important-log.log

# Ignore a file only in the root (leading slash anchors to root)
/config.local.yaml

# Ignore all files in any directory named 'temp'
**/temp/
```

### Pattern Rules Summary

| Pattern | Matches |
|---|---|
| `*.log` | Any file ending in `.log` anywhere |
| `build/` | The `build/` directory (anywhere) |
| `/build/` | Only the top-level `build/` directory |
| `**/logs/` | Any directory named `logs` at any depth |
| `doc/*.txt` | `.txt` files directly in `doc/` (not subdirs) |
| `doc/**/*.txt` | `.txt` files anywhere under `doc/` |
| `!README.md` | Override: do NOT ignore `README.md` |

### Global .gitignore

For OS-generated and editor-generated files that should be ignored everywhere (not just one project), use a global gitignore:

```bash
git config --global core.excludesFile ~/.gitignore_global
```

Example `~/.gitignore_global`:

```gitignore
# macOS
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes

# Windows
Thumbs.db
ehthumbs.db
Desktop.ini

# VS Code
.vscode/

# JetBrains IDEs
.idea/
*.iml

# Vim
*.swp
*.swo
*~
```

### Debugging Ignored Files

```bash
# Check why a specific file is being ignored
git check-ignore -v config/secrets.yaml
# .gitignore:3:*.yaml     config/secrets.yaml
# Output: <source-file>:<line-number>:<pattern>  <matched-file>

# Check if a file is ignored (exit code 0 = ignored, 1 = not ignored)
git check-ignore config/secrets.yaml

# Show all ignored files
git status --ignored
```

---

## 3.12 git ls-files

`git ls-files` lists files known to Git — either tracked files, staged files, or ignored files.

```bash
# List all tracked files
git ls-files

# List tracked files in a subdirectory
git ls-files src/

# List all ignored files
git ls-files --ignored --exclude-standard

# List untracked files
git ls-files --others --exclude-standard

# List files with their stage status (useful during merges)
git ls-files -s
```

---

## 3.13 The Practical Edit → Stage → Commit Workflow

This is the cycle you will repeat thousands of times. Understanding it deeply prevents most common Git mistakes.

### Scenario: Adding a new feature

```bash
# 1. Check your starting state
git status
# On branch main — clean working tree

# 2. Make your changes
vim src/auth/login.go
vim src/auth/logout.go
vim tests/auth_test.go
vim README.md    ← unrelated doc fix

# 3. See what changed
git diff
# Shows all changes across all 4 files

# 4. Stage the related changes (login feature) separately from the docs fix
git add src/auth/login.go src/auth/logout.go tests/auth_test.go

# 5. Verify what is staged vs what is not
git status
# Staged: login.go, logout.go, auth_test.go
# Not staged: README.md

git diff --staged
# Shows only the staged changes (the auth feature)

# 6. Commit the feature
git commit -m "Add login and logout endpoints with tests

Implements session-based authentication using Redis.
Logout invalidates the session token immediately.

Closes #78"

# 7. Now stage and commit the unrelated docs fix
git add README.md
git commit -m "Update README with authentication setup instructions"

# 8. Verify the history looks clean
git log --oneline
# abc123 Update README with authentication setup instructions
# def456 Add login and logout endpoints with tests
# ...
```

### Using -p for even finer control

```bash
# You changed a file for two different reasons in the same edit session
# Use -p to commit only the first set of changes
git add -p src/config.go

# Git shows each hunk; you choose y/n for each
# This lets you split one file's changes into two separate commits
```

---

## Command Quick Reference

| Command | Description |
|---|---|
| `git init` | Initialize a new repository |
| `git init --bare` | Initialize a bare repository (for servers) |
| `git clone <url>` | Clone a remote repository |
| `git clone --depth 1 <url>` | Shallow clone (latest snapshot only) |
| `git status` | Show working tree and staging area state |
| `git status -s` | Short status format |
| `git add <file>` | Stage a specific file |
| `git add .` | Stage all changes in current directory |
| `git add -A` | Stage all changes everywhere |
| `git add -p` | Interactive patch staging |
| `git diff` | Unstaged changes |
| `git diff --staged` | Staged changes vs. last commit |
| `git diff <a> <b>` | Changes between two commits/branches |
| `git diff --stat` | Summary of changed files |
| `git commit -m "msg"` | Commit with inline message |
| `git commit -am "msg"` | Stage tracked files + commit |
| `git commit --amend` | Rewrite last commit |
| `git log` | Show commit history |
| `git log --oneline --graph --all` | Visual branch graph |
| `git log -p` | Show diff in each commit |
| `git log --author="name"` | Filter by author |
| `git log -S "string"` | Find commits that changed a string |
| `git show <hash>` | Inspect a specific commit |
| `git rm <file>` | Remove file from disk and tracking |
| `git rm --cached <file>` | Stop tracking; keep on disk |
| `git mv <old> <new>` | Rename or move a tracked file |
| `git ls-files` | List tracked files |
| `git check-ignore -v <file>` | Debug why a file is ignored |

---

## Knowledge Check

1. What is the difference between `git add .` and `git add -A`?
2. You staged three files. You now realize one of them should not be in this commit. How do you unstage it without losing your changes?
3. What does `git diff` show vs. `git diff --staged`? When would each be empty?
4. You run `git commit -am "fix typo"`. A new file `todo.txt` exists in the directory. Is it included in the commit? Why or why not?
5. Why is `git commit --amend` dangerous on commits that have already been pushed?
6. You used `git log -S "API_KEY"` and found a commit that accidentally added a secret. What would your next steps be?
7. What is the difference between `git rm` and simply deleting a file with `rm`?
8. Write a `.gitignore` pattern that ignores all `*.log` files everywhere EXCEPT `important.log` at the root.

---

## Hands-On Exercise

### Exercise 3.1: Build a Multi-Commit Repository

1. Create and initialize a new repository.
2. Create three files: `app.py`, `tests.py`, and `README.md`.
3. Stage and commit `app.py` and `tests.py` as one commit: "Add application and test stubs"
4. Stage and commit `README.md` separately: "Add README"
5. Modify `app.py` — add a function. Stage only part of it using `git add -p`. Commit that partial change.
6. Commit the rest of `app.py` as a separate commit.
7. Run `git log --oneline --stat` and verify you have 4 commits, each logical and separate.
8. Run `git show HEAD~2` to inspect the second-to-last commit.
9. Create a `.gitignore` that ignores `*.pyc` and `__pycache__/`. Verify with `git check-ignore -v`.
10. Run `git ls-files` to see all tracked files.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./02-git-internals.md">← Previous: Git Internals</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./04-branching.md">Next: Branching →</a>
</div>
