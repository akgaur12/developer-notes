# Chapter 09 — Undoing Changes

## Learning Objectives

By the end of this chapter you will be able to:

- Choose the correct undo mechanism based on whether changes are staged, committed, or pushed
- Use `git restore` to discard working tree changes and unstage files
- Use `git reset` with `--soft`, `--mixed`, and `--hard` to move HEAD
- Use `git revert` to safely undo changes on shared branches
- Use `git commit --amend` to fix the most recent commit
- Use `git reflog` to recover from seemingly irreversible mistakes

**Prerequisites:** Chapter 08 — Stash & Cherry-pick

---

## Overview

Git provides several undo mechanisms. The right one depends on:

1. **Where the change lives** — working tree, staging index, or a commit
2. **Whether the commit has been pushed** — pushed commits on shared branches must not be rewritten
3. **How far back** you need to go — last commit, several commits, or an old commit in the middle of history

The table at the end of this chapter summarises all options side by side.

---

## `git restore` — Discard Changes in Files (Git 2.23+)

`git restore` is the modern, focused command for discarding file changes. It does **not** move HEAD or touch history.

### Discard unstaged changes in a file

```bash
git restore <file>
```

**DESTRUCTIVE** — the working-tree changes are gone permanently (not stashed, not recoverable from reflog).

```bash
git restore .              # discard all unstaged changes in the repo
git restore src/           # discard changes in a directory
```

### Unstage a file (keep the changes)

```bash
git restore --staged <file>
```

Moves the file from the index back to the working tree. The changes are preserved — they're just no longer staged.

```bash
git restore --staged .     # unstage everything
```

### Restore a file to a specific commit's version

```bash
git restore --source=<hash> <file>
git restore --source=HEAD~2 config.yml
```

The file is written to your working tree at the specified version. Use `git add` to stage it if you want to commit the restoration.

---

## `git reset` — Move HEAD (and Optionally the Index / Working Tree)

`git reset` moves the **branch pointer (HEAD)** to a target commit. What happens to the index and working tree depends on the mode flag.

### The Three Trees Mental Model

Git manages three "trees" (snapshots of your files):

| Tree | What it holds |
|---|---|
| **HEAD** | The last commit on the current branch |
| **Index (Staging Area)** | What will go into the next commit |
| **Working Tree** | Your actual files on disk |

`git reset` always moves HEAD. The flag controls how far the reset propagates:

### `--soft`: Move HEAD only

```bash
git reset --soft <hash>
git reset --soft HEAD~1    # undo last commit
```

- HEAD moves to `<hash>`
- Index stays as it was (still staged)
- Working tree stays as it was
- **Effect:** the commit is undone but all its changes remain staged, ready to re-commit

### `--mixed` (default): Move HEAD + reset index

```bash
git reset <hash>            # --mixed is the default
git reset HEAD~1
```

- HEAD moves to `<hash>`
- Index is reset to match `<hash>`
- Working tree stays as it was
- **Effect:** the commit is undone; changes are back in the working tree as unstaged modifications

### `--hard`: Move HEAD + reset index + reset working tree

```bash
git reset --hard <hash>
git reset --hard HEAD~1
```

- HEAD moves to `<hash>`
- Index is reset to match `<hash>`
- Working tree is reset to match `<hash>`
- **Effect: DESTRUCTIVE** — all uncommitted changes are lost; the working tree matches `<hash>` exactly

### Safety Rule for `git reset`

> **NEVER reset commits that have been pushed to a shared branch.**

Resetting rewrites history. Teammates who have pulled the original commits will have orphaned history when you force-push. If you must rewrite pushed history, coordinate with the team and use `--force-with-lease` (safer than `--force` — fails if the remote has changed since your last fetch).

---

## `git revert` — Safe Undo for Shared Branches

`git revert` creates a **new commit** that applies the exact inverse of a target commit's diff. It does not rewrite history — it adds to it. This makes it the correct tool when commits have already been pushed.

### Revert a single commit

```bash
git revert <hash>
```

Git opens your editor to confirm the revert commit message, then creates the new commit.

### Revert without immediately committing

```bash
git revert --no-commit <hash>
```

Applies the inverse changes to the working tree and index without creating a commit. Useful for batching multiple reverts into one commit:

```bash
git revert --no-commit abc123
git revert --no-commit def456
git commit -m "revert: roll back auth changes"
```

### Reverting a merge commit

Reverting a merge requires specifying which parent to treat as the mainline:

```bash
git revert -m 1 <merge-commit-hash>
```

`-m 1` means "treat parent 1 as mainline" — typically parent 1 is the branch that was merged into (e.g., `main`). The revert undoes everything that was introduced by the merged branch.

---

## `git commit --amend` — Fix the Most Recent Commit

`--amend` replaces the most recent commit with a new one. The working tree changes are merged into the previous commit, creating a fresh commit object (new hash).

### Fix the commit message

```bash
git commit --amend -m "fix: correct the login error message"
```

### Add a forgotten file to the last commit

```bash
git add forgotten-file.js
git commit --amend --no-edit    # --no-edit keeps the existing message
```

### Safety Rule for `--amend`

`--amend` rewrites the last commit — it produces a new hash. If you have already pushed this commit, you would need to force-push to update the remote.

> Only amend commits that have **not yet been pushed** to a shared branch.

If already pushed and you must amend, use `--force-with-lease`:

```bash
git push --force-with-lease origin feature/my-branch
```

---

## `git reflog` — The Safety Net

The reflog records **every movement of HEAD** — every commit, checkout, reset, rebase, merge, and amend. It is local to your machine and is the last resort for recovering "lost" work.

### View the reflog

```bash
git reflog
```

Output example:

```
7a3f9bc (HEAD -> main) HEAD@{0}: commit: fix: auth null check
2d1a8ee HEAD@{1}: reset: moving to HEAD~1
9c4b1f2 HEAD@{2}: commit: feat: add login form
3e0d7a1 HEAD@{3}: checkout: moving from feature to main
```

### View the reflog for a specific branch

```bash
git reflog show feature/my-branch
```

### Recovering a Lost Commit

Scenario: you ran `git reset --hard` and lost commits you needed.

```bash
# Find the hash in the reflog
git reflog
# Identify the commit before the reset, e.g. 9c4b1f2

# Option 1: create a new branch at that point
git checkout -b recovery 9c4b1f2

# Option 2: hard reset current branch back to it
git reset --hard 9c4b1f2
```

### Reflog Expiry

Reflog entries are not permanent:

- Reachable commits: expire after **90 days** by default
- Unreachable commits (orphaned): expire after **30 days** by default

> "Nothing is truly lost until the reflog expires." — if you act promptly, you can recover almost anything in Git.

---

## Decision Flowchart — Which Undo to Use?

```
Did you commit the change?
├── No (uncommitted)
│   ├── Is it staged? → git restore --staged <file>  (unstage, keep changes)
│   └── Not staged?   → git restore <file>           (DESTRUCTIVE, discard changes)
│
└── Yes (committed)
    ├── Was it pushed to a shared branch?
    │   ├── No (local only)
    │   │   ├── Fix the message or add a file? → git commit --amend
    │   │   └── Undo 1+ commits?
    │   │       ├── Keep changes staged?      → git reset --soft HEAD~N
    │   │       ├── Keep changes unstaged?    → git reset --mixed HEAD~N
    │   │       └── Discard all changes?      → git reset --hard HEAD~N (DESTRUCTIVE)
    │   │
    │   └── Yes (pushed / shared)
    │       └── → git revert <hash>   (safe, adds new commit, no history rewrite)
    │
    └── Can't find the commit?
        └── → git reflog  (find the hash, then checkout or reset)
```

---

## Summary Comparison Table

| Command | Moves HEAD | Rewrites History | Destroys Changes | Safe After Push |
|---|---|---|---|---|
| `git restore <file>` | No | No | Yes (working tree) | N/A |
| `git restore --staged` | No | No | No | N/A |
| `git reset --soft` | Yes | Yes (local) | No | No |
| `git reset --mixed` | Yes | Yes (local) | No | No |
| `git reset --hard` | Yes | Yes (local) | Yes (all) | No |
| `git revert` | No (adds commit) | No | No | **Yes** |
| `git commit --amend` | Yes (replaces) | Yes (local) | No | No |
| `git reflog` + reset | Yes | Yes (local) | Depends on mode | No |

---

## Knowledge Check

1. What is the difference between `git restore <file>` and `git restore --staged <file>`?
2. You ran `git reset --hard HEAD~3` but realise you needed one of those commits. How do you recover it?
3. When should you use `git revert` instead of `git reset`?
4. What does `git commit --amend --no-edit` do?
5. How long does Git keep unreachable commits in the reflog by default?
6. You need to undo a commit that was merged into `main` 2 weeks ago and has been deployed. What is the safest command?

---

## Hands-On Exercise

**Goal:** Practice each undo mechanism safely in an isolated test repo.

```bash
# Setup
git init undo-practice && cd undo-practice
echo "line 1" > file.txt && git add . && git commit -m "first commit"
echo "line 2" >> file.txt && git add . && git commit -m "second commit"
echo "line 3" >> file.txt && git add . && git commit -m "third commit"

# --- Exercise 1: git restore ---
echo "accidental change" >> file.txt   # unstaged change
git restore file.txt                    # discard it
git diff                                # should show nothing

# --- Exercise 2: git restore --staged ---
echo "staged change" >> file.txt
git add file.txt
git restore --staged file.txt          # unstage (keep changes)
git status                              # should show unstaged modification

# --- Exercise 3: git reset --soft ---
git add . && git commit -m "temp commit"
git reset --soft HEAD~1                 # undo commit, keep staged
git status                              # changes still staged

# --- Exercise 4: git reset --mixed ---
git commit -m "temp commit 2"
git reset HEAD~1                        # undo commit, keep unstaged
git status                              # changes unstaged

# --- Exercise 5: git reset --hard (careful!) ---
git add . && git commit -m "temp commit 3"
git reset --hard HEAD~1                 # undo commit, lose changes
git status                              # clean working tree

# --- Exercise 6: git revert ---
git log --oneline                       # pick a commit hash, e.g. abc1234
git revert abc1234                      # creates a new inverse commit
git log --oneline                       # original + revert commit both visible

# --- Exercise 7: git reflog ---
git reflog                              # find the hash of the reset commit from Exercise 5
# checkout the lost commit to a new branch to inspect it
git checkout -b recovered-branch <hash-from-reflog>
git log --oneline
```

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="08-stash-and-cherry-pick.md">← Previous: Stash & Cherry-pick</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="10-branching-strategies.md">Next: Branching Strategies →</a>
</div>
