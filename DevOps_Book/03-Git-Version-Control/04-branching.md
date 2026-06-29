# Chapter 04 — Branching

## Learning Objectives

By the end of this chapter you will be able to:

- Explain what a branch is at the internal Git level
- Create, rename, and delete branches with confidence
- Switch between branches using both `git checkout` and the modern `git switch`
- Understand what HEAD is and what "detached HEAD" means
- Visualize branch divergence with `git log --graph`
- Follow standard branch naming conventions used in team workflows

**Prerequisites:** Chapter 03 — Essential Commands

---

## What Is a Branch?

A Git branch is nothing more than a **41-byte file** that holds the SHA-1 hash of a single commit. That's it. When you create a new branch, Git writes one tiny file inside `.git/refs/heads/`. There is no copying of files, no duplication of history — just a pointer.

```
.git/refs/heads/main     → abc1234...
.git/refs/heads/feature  → abc1234...   (same commit right after branching)
```

This is why branching in Git is nearly instantaneous, regardless of repository size. Compare this to older VCS tools (SVN, Perforce) where branching could involve copying entire directory trees.

As you make new commits on a branch, the pointer advances automatically to the latest commit.

---

## Listing Branches

```bash
git branch           # list local branches
git branch -a        # list all branches (local + remote-tracking)
git branch -r        # list remote-tracking branches only
git branch -v        # verbose: show last commit on each branch
git branch -vv       # very verbose: also show upstream tracking info
```

Example output of `git branch -vv`:

```
* main        abc1234 [origin/main] Initial commit
  feature/ui  def5678 [origin/feature/ui: ahead 2] Add login form
  bugfix/nav  ghi9012 Add responsive nav
```

The `*` marks the currently checked-out branch. `ahead 2` means you have 2 local commits not yet pushed.

---

## Creating Branches

```bash
git branch <name>          # create branch at current HEAD (does NOT switch to it)
git branch hotfix/login    # example
```

---

## Deleting Branches

```bash
git branch -d <name>    # delete merged branch (safe: refuses if unmerged)
git branch -D <name>    # force delete (even if unmerged — use with care)
```

Git's `-d` flag is your safety net. If you try to delete a branch whose commits haven't been merged into the current branch, Git will warn you and refuse. Use `-D` only when you deliberately want to discard unmerged work.

---

## Renaming Branches

```bash
git branch -m <old-name> <new-name>    # rename any branch
git branch -m <new-name>               # rename the current branch
```

If the branch has already been pushed to a remote, you'll need to push the renamed branch and delete the old remote branch:

```bash
git push origin <new-name>
git push origin --delete <old-name>
```

---

## Switching Branches

### The Classic Way: `git checkout`

```bash
git checkout main              # switch to main
git checkout -b feature/login  # create AND switch in one step
```

`git checkout` is a Swiss army knife command — it also checks out individual files, detaches HEAD onto commits, and more. This overloading causes confusion for beginners.

### The Modern Way: `git switch` (Recommended)

Introduced in Git 2.23, `git switch` does one thing: change branches.

```bash
git switch main                 # switch to existing branch
git switch -c feature/login     # create AND switch (equivalent to checkout -b)
git switch -                    # switch back to the previous branch
```

`git switch` is more explicit and harder to misuse. Prefer it in all new scripts and workflows.

---

## The HEAD Pointer

`HEAD` is a special pointer that always tells Git **which branch you are on** (or which commit, in detached HEAD mode). It is stored in `.git/HEAD`.

```
# Normal state — HEAD points to a branch
$ cat .git/HEAD
ref: refs/heads/main

# After git switch feature/ui
$ cat .git/HEAD
ref: refs/heads/feature/ui
```

When you make a new commit, two things happen:
1. A new commit object is written
2. The branch pointer (e.g., `main`) advances to the new commit
3. HEAD still points to `main`, so it effectively advances too

---

## Detached HEAD

Detached HEAD means `HEAD` points **directly to a commit hash** rather than to a branch name.

```
# Normal:   HEAD → main → commit abc1234
# Detached: HEAD → commit abc1234
```

### When Does It Happen?

- `git checkout <commit-hash>` — checking out an old commit
- `git checkout <tag>` — checking out a tag
- During rebase and bisect operations internally

```bash
git checkout abc1234    # HEAD is now detached
# Git will tell you:
# You are in 'detached HEAD' state.
```

### Why It Matters

Any commits you make in detached HEAD state are not attached to a branch. If you switch away without saving them to a branch, they become unreachable and will eventually be garbage-collected.

### How to Get Out

```bash
# Option 1: Create a new branch here and switch to it
git switch -c my-new-branch

# Option 2: Discard any new commits and return to main
git switch main
```

---

## Remote-Tracking Branches

When you clone or fetch from a remote, Git creates **remote-tracking branches** — read-only local copies of what the remote looks like.

```
origin/main      # where origin's main was, last time you fetched
origin/feature/x # where origin's feature/x was, last time you fetched
```

These are listed under `git branch -r`. You cannot commit to them directly. They update only when you run `git fetch` or `git pull`.

---

## Visualizing the Branch Graph

```bash
git log --oneline --graph --all
```

This command draws an ASCII graph of the entire commit DAG (Directed Acyclic Graph), showing all branches and where they diverge.

Example output:

```
* f3a1b2c (HEAD -> feature/login) Add password validation
* e2d0a1b Add login form HTML
| * 9c8b7a6 (main) Fix typo in README
|/
* 4a3b2c1 Initial project setup
```

This tells you:
- Both `feature/login` and `main` diverged from commit `4a3b2c1`
- `main` has one additional commit (the README fix)
- `feature/login` has two commits ahead

### Branch Divergence Diagram

```
        A---B---C   feature/login
       /
  D---E             main (HEAD)
```

- `D` and `E` are on `main`
- At `E`, a new branch `feature/login` was created
- `A`, `B`, `C` are commits only on `feature/login`
- `main` has not advanced since the branch was created

---

## Deleting Branches: Summary

| Command | Behavior |
|---|---|
| `git branch -d feature` | Safe delete — refuses if unmerged |
| `git branch -D feature` | Force delete — no safety check |
| `git push origin --delete feature` | Delete branch on remote |

**Tip:** After a pull request is merged, always delete the feature branch both locally and on the remote to keep the repository clean.

---

## Branch Naming Conventions

Consistent naming makes it immediately obvious what each branch is for.

| Prefix | Purpose | Example |
|---|---|---|
| `feature/` | New functionality | `feature/user-authentication` |
| `bugfix/` | Non-critical bug fix | `bugfix/navbar-overflow` |
| `hotfix/` | Urgent production fix | `hotfix/payment-null-check` |
| `release/` | Release preparation | `release/2.4.0` |
| `chore/` | Non-code tasks (deps, CI) | `chore/update-node-18` |

Rules of thumb:
- Use lowercase and hyphens, never spaces or underscores
- Keep names short but descriptive
- Include a ticket/issue number if your team uses one: `feature/PROJ-123-login-flow`

---

## Summary

- A branch is a 41-byte pointer — it is cheap to create and cheap to delete
- `git switch` is the modern, preferred command for changing branches
- HEAD tracks your current branch; detached HEAD means HEAD points directly at a commit
- Remote-tracking branches (`origin/main`) are read-only local snapshots
- `git log --oneline --graph --all` is your best friend for visualizing branch structure
- Use naming conventions (`feature/`, `bugfix/`, etc.) from day one

---

## Knowledge Check

1. If a branch is just a pointer, why is `git branch -d` sometimes safer than deleting the pointer file directly?
2. You run `git checkout v1.0.0` and Git warns you about detached HEAD. What happened, and how do you safely create a new branch from here?
3. What is the difference between `origin/main` and `main`?
4. Why might a team enforce `git merge --no-ff` for all feature branches?

---

## Hands-On Exercise

```bash
# 1. Initialize a repo and make an initial commit
git init branch-lab && cd branch-lab
echo "# Branch Lab" > README.md
git add README.md && git commit -m "Initial commit"

# 2. Create two feature branches
git switch -c feature/header
echo "<header>Hello</header>" > header.html
git add header.html && git commit -m "Add header"

git switch main
git switch -c feature/footer
echo "<footer>Bye</footer>" > footer.html
git add footer.html && git commit -m "Add footer"

# 3. Visualize the divergence
git log --oneline --graph --all

# 4. Experience detached HEAD
git checkout HEAD~1
git log --oneline
git switch -c recovery-branch   # save work then return
git switch main

# 5. Clean up
git branch -d feature/header
git branch -d feature/footer
git branch -d recovery-branch
```

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./03-essential-commands.md">← Previous: Essential Commands</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./05-remote-repositories.md">Next: Remote Repositories →</a>
</div>
