# Chapter 01 — Introduction to Git & Version Control

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain what version control is and why it is essential in software development
- Describe the evolution from local VCS to distributed VCS
- Summarize the origin and design goals of Git
- Compare Git to other version control systems (SVN, Mercurial)
- Identify the three states of a file in Git's model
- Install Git on Ubuntu/Debian, macOS, and Windows
- Configure Git with your identity, preferred editor, and other global settings
- Use `git help`, `git --version`, and `git config --list`

---

## 1.1 What Is Version Control and Why It Matters

Version control is a system that records changes to a file or set of files over time so that you can recall specific versions later. Without version control, developers typically resort to strategies like copying entire folders (`project_v1/`, `project_final/`, `project_final_REAL/`) — an approach that is error-prone, consumes storage, and makes collaboration nearly impossible.

Version control solves three fundamental problems:

1. **History** — Every change is recorded. You can see who changed what, when, and why. You can rewind to any previous state.
2. **Collaboration** — Multiple developers can work on the same codebase simultaneously without overwriting each other's work.
3. **Recovery** — If you introduce a bug or accidentally delete a file, you can restore a known-good state instantly.

### The Evolution of Version Control Systems

Version control systems have evolved through three generations:

#### Generation 1: Local VCS
Early version control systems kept patch sets (the differences between files) in a local database. The most well-known example is **RCS (Revision Control System)**. The fatal limitation: all history is on a single machine, and collaboration requires manually exchanging files.

```
Developer's Machine
┌────────────────────────────┐
│  Working Files             │
│  ┌──────────────────────┐  │
│  │  Version Database    │  │
│  │  v1, v2, v3, ...     │  │
│  └──────────────────────┘  │
└────────────────────────────┘
```

#### Generation 2: Centralized VCS (CVCS)
Systems like **CVS** and **SVN (Subversion)** introduced a single central server that holds all versioned files. Clients check out files from this central place. This enables collaboration but introduces a critical single point of failure — if the central server goes down, nobody can save versioned changes or see history.

```
         Central Server
         ┌───────────┐
         │ Repository│
         └─────┬─────┘
    ┌──────────┼──────────┐
    ▼          ▼          ▼
Developer A  Developer B  Developer C
```

#### Generation 3: Distributed VCS (DVCS)
Systems like **Git** and **Mercurial** give every client a full mirror of the repository, including its complete history. There is no single point of failure. You can commit, branch, and view history entirely offline. When you push to a shared server, it is simply one node among many — not a privileged central authority.

```
         Remote Server
         ┌───────────┐
         │ Repository│ (full history)
         └─────┬─────┘
    ┌──────────┼──────────┐
    ▼          ▼          ▼
Developer A  Developer B  Developer C
(full repo)  (full repo)  (full repo)
```

---

## 1.2 Git's Origin Story

Git was created by **Linus Torvalds** in **April 2005**. The story begins with the Linux kernel project, which at the time was using a proprietary DVCS called **BitKeeper**. When the free-of-charge status of BitKeeper was revoked due to a licensing dispute, the Linux kernel community needed a replacement — fast.

Linus Torvalds had specific, non-negotiable requirements for the new tool:

- **Speed** — Must handle the Linux kernel's scale (thousands of files, hundreds of contributors)
- **Simple design** — No unnecessary complexity
- **Strong support for non-linear development** — Thousands of parallel branches
- **Fully distributed** — No reliance on a central server
- **Integrity** — Cryptographic verification of history (SHA-1 hashing of every object)

In roughly **10 days**, Torvalds wrote the first version of Git. It was self-hosting by April 7, 2005 (Git's first commit was made using Git itself). By June 2005, the Linux kernel was already being maintained with Git.

Junio Hamano took over as the primary maintainer in July 2005, a role he continues to hold today.

> "I'm an egotistical bastard, and I name all my projects after myself. First 'Linux', now 'Git'."
> — Linus Torvalds. ('Git' is British slang for an unpleasant person.)

---

## 1.3 Git vs. Other Version Control Systems

| Feature | Git | SVN (Subversion) | Mercurial |
|---|---|---|---|
| Architecture | Distributed | Centralized | Distributed |
| Offline commits | Yes | No | Yes |
| Branching cost | Near-zero (pointer) | Expensive (copy) | Lightweight |
| History storage | Snapshots | Deltas | Changesets |
| Rename tracking | Heuristic | Explicit | Heuristic |
| Learning curve | Steep but powerful | Gentler | Moderate |
| Industry adoption | Dominant | Declining | Niche |
| Binary files | Poor (use Git LFS) | Acceptable | Acceptable |
| Atomic commits | Yes (local) | Yes (server) | Yes (local) |
| Rewriting history | Yes (rebase, amend) | Very limited | Limited |
| Performance (large repos) | Excellent | Moderate | Good |
| Ecosystem/tooling | Vast (GitHub, GitLab, etc.) | Moderate | Limited |

**Key insight:** SVN has a single, authoritative server. Git has no inherent "central" server — GitHub and GitLab are convenient remotes, not requirements.

---

## 1.4 Key Git Concepts

Before writing a single command, you need a clear mental model of how Git tracks your work.

### The Three Areas

Every file in a Git project lives in one of three areas at any given moment:

```
┌──────────────────────────────────────────────────────────┐
│                     Your Project                         │
│                                                          │
│  ┌─────────────┐   git add   ┌─────────────┐            │
│  │             │ ──────────► │             │            │
│  │  Working    │             │   Staging   │            │
│  │    Tree     │             │  Area (Index│            │
│  │             │ ◄────────── │             │            │
│  └─────────────┘  git restore└──────┬──────┘            │
│                                     │ git commit         │
│                                     ▼                    │
│                             ┌─────────────┐             │
│                             │    .git     │             │
│                             │  Repository │             │
│                             │  (History)  │             │
│                             └─────────────┘             │
└──────────────────────────────────────────────────────────┘
```

**Working Tree (Working Directory)**
This is the directory on your filesystem where you edit files. Git is aware of these files but does not automatically record changes to them.

**Staging Area (Index)**
A preparatory area where you assemble the exact set of changes you want to include in your next commit. Think of it as a draft commit. You use `git add` to move changes here.

**Repository (.git directory)**
The Git database, stored in the hidden `.git` folder at the root of your project. When you run `git commit`, Git takes a snapshot of everything in the staging area and permanently stores it here.

### The Three States of a File

A tracked file in Git is always in one of three states:

| State | Meaning | How to advance |
|---|---|---|
| **Modified** | You have changed the file in the working tree but not yet staged it | Run `git add <file>` |
| **Staged** | You have marked the modified file to go into the next commit | Run `git commit` |
| **Committed** | The data is safely stored in the local repository | Push to share it |

There is a fourth conceptual state: **Untracked** — files in your working directory that Git has never seen before. They are neither modified nor staged from Git's perspective. Use `git add` to begin tracking them.

---

## 1.5 Installing Git

### Ubuntu / Debian (APT)

```bash
# Update package list
sudo apt update

# Install Git
sudo apt install git -y

# Verify installation
git --version
# git version 2.43.0
```

### macOS (Homebrew)

```bash
# Install Homebrew if not present
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Git
brew install git

# Verify
git --version
# git version 2.44.0
```

> macOS ships with an Apple-bundled Git from Xcode Command Line Tools, but the Homebrew version is more current. Both work, but Homebrew's version is preferred for development.

### Windows

1. Download the installer from [https://git-scm.com/download/win](https://git-scm.com/download/win)
2. Run the installer. Accept defaults, but consider these choices:
   - **Default editor**: Choose VS Code or Notepad++ over the default Vim if you are not comfortable with Vim
   - **Line ending conversions**: Choose "Checkout Windows-style, commit Unix-style line endings"
   - **Terminal emulator**: Choose "Use Windows' default console window" or "Use MinTTY"
3. Verify in Git Bash or PowerShell:

```powershell
git --version
# git version 2.44.0.windows.1
```

---

## 1.6 Essential First-Time Setup

After installing Git, the very first thing you must do is tell Git who you are. This identity is embedded into every commit you make. There is no way to commit without it.

### Setting Your Identity

```bash
# Set your name (use your real name or team handle)
git config --global user.name "Jane Doe"

# Set your email (should match your GitHub/GitLab account)
git config --global user.email "jane.doe@example.com"
```

### Setting Your Default Editor

Git opens a text editor when you need to write a commit message without `-m`, during interactive rebases, and for merge conflict resolution messages.

```bash
# VS Code (opens a new tab; waits for the tab to close)
git config --global core.editor "code --wait"

# Vim
git config --global core.editor "vim"

# Nano (friendlier for beginners)
git config --global core.editor "nano"

# Notepad++ (Windows)
git config --global core.editor "'C:/Program Files/Notepad++/notepad++.exe' -multiInst -notabbar -nosession -noPlugin"
```

### Setting the Default Branch Name

Since Git 2.28, you can configure the default branch name for new repositories (the industry has moved from `master` to `main`):

```bash
git config --global init.defaultBranch main
```

### Configuration Scopes

Git has three levels of configuration, each overriding the previous:

| Scope | Flag | Location | Applies to |
|---|---|---|---|
| **System** | `--system` | `/etc/gitconfig` | All users on the machine |
| **Global** | `--global` | `~/.gitconfig` or `~/.config/git/config` | All repos for your user |
| **Local** | `--local` | `.git/config` (inside repo) | Only that specific repository |

Example: you might set `user.email` globally to your personal email, then override it locally inside a work project's repo:

```bash
# Inside a work project directory
git config --local user.email "jane.doe@company.com"
```

### Anatomy of ~/.gitconfig

After running the setup commands above, your `~/.gitconfig` will look like this:

```ini
[user]
    name = Jane Doe
    email = jane.doe@example.com

[core]
    editor = code --wait

[init]
    defaultBranch = main
```

You can edit this file directly with a text editor — it is just an INI-format text file. Each section in square brackets (`[user]`, `[core]`) groups related settings.

### Useful Additional Configuration

```bash
# Colorize output (highly recommended)
git config --global color.ui auto

# Set the default push behavior (push only the current branch)
git config --global push.default current

# Automatically set up tracking for new branches on push
git config --global push.autoSetupRemote true

# Use a credential helper to avoid re-entering passwords
# macOS
git config --global credential.helper osxkeychain
# Windows
git config --global credential.helper manager
# Linux (stores in memory for 15 minutes)
git config --global credential.helper cache
```

---

## 1.7 Verifying Your Setup

### Check the Git Version

```bash
git --version
# git version 2.43.0
```

### List All Configuration

```bash
# Show all configuration with their source files
git config --list --show-origin

# Show only global config
git config --global --list

# Get a specific value
git config user.email
```

### Getting Help

Git has comprehensive built-in documentation:

```bash
# Full man page for a command
git help commit
git help config

# Shorter summary (flags and options)
git commit -h
git clone -h

# Open the HTML version of the manual in a browser
git help --web log
```

---

## Summary

| Concept | Key Takeaway |
|---|---|
| Version control | Records history, enables collaboration, allows recovery |
| Local VCS | Single machine, no collaboration |
| Centralized VCS | Single server, single point of failure (SVN) |
| Distributed VCS | Every clone is a full repository (Git, Mercurial) |
| Git's origin | Created by Linus Torvalds in 2005 for the Linux kernel |
| Working tree | Where you edit files |
| Staging area (index) | Where you prepare the next commit |
| Repository (.git) | Where Git permanently stores snapshots |
| Three file states | Modified → Staged → Committed |
| `--global` config | Applies to all repos for the current user |
| `--local` config | Overrides global for a specific repo |

---

## Knowledge Check

1. What problem does a centralized VCS introduce that a distributed VCS solves?
2. Why did Linus Torvalds create Git in 2005?
3. What is the difference between the working tree and the staging area?
4. A file is in the "modified" state. What command moves it to the "staged" state?
5. You want your work email to be used only in one specific repository without affecting your global config. What command do you run?
6. What does `git config --list --show-origin` do that `git config --list` does not?

---

## Hands-On Exercise

### Exercise 1.1: Install and Configure Git

1. Install Git on your system using the instructions for your OS.
2. Verify the installation with `git --version`.
3. Configure your name and email globally.
4. Set your preferred editor.
5. Set the default branch name to `main`.
6. Run `git config --list` and verify all settings are correct.
7. Open `~/.gitconfig` in a text editor and inspect its contents.

### Exercise 1.2: Explore Git Help

1. Run `git help` to see the list of common commands.
2. Run `git help config` to open the full config manual.
3. Run `git commit -h` to see a quick summary of commit flags.
4. Find the flag for `git log` that shows the patch (diff) for each commit. (Hint: look in `git log -h`)

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./00-index.md">← Previous: Index</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./02-git-internals.md">Next: Git Internals →</a>
</div>
