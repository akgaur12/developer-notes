# Chapter 05 — Remote Repositories

## Learning Objectives

By the end of this chapter you will be able to:

- Explain what a remote is and how Git stores remote information
- Add, remove, and rename remotes
- Understand the critical difference between `git fetch` and `git pull`
- Push branches and configure upstream tracking
- Use `--force-with-lease` instead of `--force` for safer force pushes
- Set up SSH key authentication for GitHub/GitLab

**Prerequisites:** Chapter 04 — Branching

---

## What Is a Remote?

A remote is simply a **named URL** — a bookmark that tells Git where another copy of the repository lives. It could be on GitHub, GitLab, Bitbucket, your company's internal server, or even another directory on your own machine.

Git stores remotes in `.git/config`:

```ini
[remote "origin"]
    url = git@github.com:user/my-repo.git
    fetch = +refs/heads/*:refs/remotes/origin/*
```

The name `origin` is just a convention — it is the default name Git assigns when you clone. You can name remotes anything, and a repository can have multiple remotes.

---

## Listing Remotes

```bash
git remote          # list remote names
git remote -v       # list remotes with their fetch and push URLs
```

Example output:

```
origin  git@github.com:user/my-repo.git (fetch)
origin  git@github.com:user/my-repo.git (push)
```

For detailed information including tracked branches:

```bash
git remote show origin
```

Output includes:
- The remote URL
- Which local branches are configured to push where
- Which remote branches are tracked by local branches
- Whether your local branches are up to date, ahead, or behind

---

## Managing Remotes

```bash
# Add a new remote
git remote add <name> <url>
git remote add upstream git@github.com:original-author/project.git

# Remove a remote
git remote remove <name>
git remote remove upstream

# Rename a remote
git remote rename <old-name> <new-name>
git remote rename origin old-origin

# Change a remote's URL
git remote set-url origin git@github.com:user/new-repo.git
```

Having multiple remotes is common in open-source workflows: `origin` for your fork, `upstream` for the original project.

---

## git clone — What It Sets Up Automatically

When you clone a repository, Git does more than copy files:

```bash
git clone git@github.com:user/my-repo.git
```

Git automatically:
1. Creates the `.git` directory and downloads all objects
2. Adds a remote named `origin` pointing to the URL you cloned from
3. Creates remote-tracking branches for every branch on the remote (`origin/main`, `origin/feature/x`, etc.)
4. Checks out the default branch (usually `main`) as a local branch
5. Configures `main` to track `origin/main` (sets the upstream)

You are immediately ready to `git fetch`, `git push`, and see tracking information.

---

## The Critical Difference: git fetch vs git pull

This is one of the most important distinctions in Git.

### git fetch

```bash
git fetch origin          # download updates from origin
git fetch --all           # download updates from all remotes
git fetch origin main     # fetch only the main branch
```

- Downloads new commits, branches, and tags from the remote
- Updates your remote-tracking branches (`origin/main`, etc.)
- **Does NOT touch your working directory or current branch**
- Always safe to run at any time, even mid-work

After `git fetch`, you can inspect what changed before integrating:

```bash
git fetch origin
git log HEAD..origin/main --oneline   # see what's new on remote main
git diff HEAD origin/main             # see the actual diff
git merge origin/main                 # integrate when ready
```

### git pull

```bash
git pull                  # fetch + merge (uses configured upstream)
git pull origin main      # fetch origin/main + merge into current branch
git pull --rebase         # fetch + rebase instead of merge
```

`git pull` is shorthand for `git fetch` followed immediately by `git merge` (or `git rebase --rebase`). It is convenient but gives you less control — integration happens immediately.

### Best Practice

**Prefer `git fetch` + explicit merge/rebase** so you can:
- Review what changed before integrating
- Choose whether to merge or rebase
- Avoid surprise merge commits on simple updates

Use `git pull` for quick personal projects; use `git fetch` in team workflows.

---

## Pushing Branches

```bash
git push <remote> <branch>
git push origin feature/login       # push feature/login to origin
```

### Setting the Upstream (-u flag)

```bash
git push -u origin main
# equivalent to:
git push --set-upstream origin main
```

The `-u` flag does two things:
1. Pushes the branch to the remote
2. Configures the local branch to track the remote branch

After setting upstream, you can use plain `git push` and `git pull` without specifying remote/branch.

### Pushing All Branches

```bash
git push --all origin    # push all local branches (use with care)
```

---

## Force Push: Danger and the Safe Alternative

### git push --force

```bash
git push --force origin main    # DANGEROUS
```

This **overwrites the remote branch** with your local version, discarding any commits on the remote that you don't have locally. If teammates have based work on those commits, this creates serious problems.

Never force push to shared branches like `main` or `develop`.

### git push --force-with-lease (Recommended)

```bash
git push --force-with-lease origin feature/my-branch
```

This is the safe force push. It fails if the remote branch has commits you haven't fetched yet — meaning someone else pushed since you last fetched. Git checks that your remote-tracking reference (`origin/feature/my-branch`) matches the actual remote before overwriting.

When to use force push at all:
- After `git rebase` on your own feature branch (before anyone else has based work on it)
- To fix a bad commit message right after pushing
- Never on `main`, `develop`, or any shared branch

---

## Tracking Branches

A tracking branch is a local branch configured to follow a remote branch.

```bash
# Set tracking for existing branch
git branch -u origin/main
git branch --set-upstream-to=origin/main

# Create a new local branch that tracks a remote branch
git switch -c feature/x origin/feature/x
```

### Tracking Status Messages

```
Your branch is up to date with 'origin/main'.
Your branch is ahead of 'origin/main' by 2 commits.
Your branch is behind 'origin/main' by 3 commits, and can be fast-forwarded.
Your branch and 'origin/main' have diverged, and have 1 and 2 different commits each.
```

| Status | Meaning | Action |
|---|---|---|
| Up to date | In sync | Nothing needed |
| Ahead by N | You have N unpushed commits | `git push` |
| Behind by N | Remote has N commits you lack | `git pull` or `git fetch` + merge |
| Diverged | Both sides have new commits | `git pull` (merge or rebase) |

---

## Pruning Stale Remote-Tracking Branches

When a branch is deleted on the remote (e.g., after a PR merge), your local remote-tracking reference persists until pruned.

```bash
git fetch --prune              # fetch and remove stale remote-tracking branches
git remote prune origin        # prune without fetching
```

You can also configure automatic pruning:

```bash
git config --global fetch.prune true    # always prune on fetch
```

After pruning, `git branch -r` will no longer show deleted remote branches.

---

## HTTPS vs SSH Remotes

### HTTPS

```
https://github.com/user/repo.git
```

- Prompts for username and password (or token) each time
- Use a credential manager to cache credentials
- Works from most networks without special configuration

### SSH

```
git@github.com:user/repo.git
```

- Authenticates with an SSH key pair — no password prompts after setup
- Preferred for daily development work
- Requires SSH access on port 22 (sometimes blocked on corporate networks)

### Switch an Existing Remote from HTTPS to SSH

```bash
git remote set-url origin git@github.com:user/repo.git
```

---

## Setting Up SSH Keys

### Step 1: Generate an SSH Key Pair

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
# Accept the default location (~/.ssh/id_ed25519)
# Set a passphrase for extra security
```

Ed25519 is the modern recommended algorithm. Use `rsa -b 4096` only if the service doesn't support Ed25519.

### Step 2: Add the Public Key to GitHub/GitLab

```bash
cat ~/.ssh/id_ed25519.pub
# Copy the output
```

- GitHub: Settings → SSH and GPG keys → New SSH key → Paste
- GitLab: Preferences → SSH Keys → Add key → Paste

### Step 3: Test the Connection

```bash
ssh -T git@github.com
# Expected: Hi username! You've successfully authenticated...

ssh -T git@gitlab.com
# Expected: Welcome to GitLab, @username!
```

### Step 4: Add Key to SSH Agent (Optional but Convenient)

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

---

## Typical Team Workflow

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  1. git clone <repo>       Clone the repository         │
│         ↓                                               │
│  2. git switch -c          Create a feature branch      │
│     feature/my-work                                     │
│         ↓                                               │
│  3. (edit files)           Write code                   │
│     git add / git commit                                │
│         ↓                                               │
│  4. git fetch origin       Check for new changes        │
│     git rebase origin/main Stay up to date              │
│         ↓                                               │
│  5. git push -u origin     Push feature branch          │
│     feature/my-work                                     │
│         ↓                                               │
│  6. Open Pull Request      Request review on GitHub     │
│         ↓                                               │
│  7. PR merged              Remote merges branch         │
│     git switch main                                     │
│     git pull               Sync local main              │
│     git branch -d          Clean up local branch        │
│     feature/my-work                                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Summary

- A remote is a named URL — just a bookmark stored in `.git/config`
- `git fetch` downloads changes safely without touching your working tree; `git pull` fetches and immediately merges
- Use `git push -u` once to set upstream tracking; then plain `git push`/`git pull` work
- Use `--force-with-lease` instead of `--force` for safer force pushes
- SSH keys are preferred for daily development — set them up once, use forever
- Prune stale remote-tracking references with `git fetch --prune`

---

## Knowledge Check

1. You run `git fetch origin`. Your coworker just pushed a hotfix to `main`. What exactly happened to your local repository, and what did NOT happen?
2. What does `git push -u origin feature/login` do differently from `git push origin feature/login`?
3. You want to force push a rebased feature branch. Why should you use `--force-with-lease` instead of `--force`?
4. After a PR is merged and the remote branch is deleted, you run `git branch -r` and still see `origin/feature/old-branch`. How do you remove it?

---

## Hands-On Exercise

```bash
# Prerequisites: a GitHub/GitLab account and SSH key configured

# 1. Create a new repo on GitHub (via UI), then clone it
git clone git@github.com:<your-username>/remote-lab.git
cd remote-lab

# 2. Inspect what clone set up
git remote -v
git branch -vv
git remote show origin

# 3. Add a second remote (e.g., a bare local repo)
git init --bare /tmp/backup-remote
git remote add backup /tmp/backup-remote
git remote -v

# 4. Create a feature branch, commit, and push
git switch -c feature/demo
echo "Hello from feature" > demo.txt
git add demo.txt && git commit -m "Add demo file"
git push -u origin feature/demo

# 5. Simulate a remote update and fetch it
# (On GitHub: edit README.md via the web UI)
git fetch origin
git log HEAD..origin/main --oneline
git merge origin/main

# 6. Inspect tracking info
git branch -vv

# 7. Prune (delete the remote branch via GitHub UI first)
git fetch --prune
git branch -r
```

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./04-branching.md">← Previous: Branching</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./06-merging.md">Next: Merging →</a>
</div>
