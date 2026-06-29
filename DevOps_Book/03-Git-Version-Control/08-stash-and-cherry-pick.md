# Chapter 08 — Stash & Cherry-pick

## Learning Objectives

By the end of this chapter you will be able to:

- Use `git stash` to temporarily shelve in-progress changes and restore them later
- Manage a stack of multiple stashes
- Use `git cherry-pick` to apply specific commits from one branch to another
- Identify the correct use cases for each command — and their pitfalls

**Prerequisites:** Chapter 07 — Rebasing

---

## Part 1: Git Stash

### What `git stash` Does

`git stash` saves your **uncommitted changes** (both staged and unstaged) to an internal stack, then **restores your working tree to a clean state** (matching the last commit). This lets you switch context — change branches, pull updates, apply a hotfix — without committing half-finished work.

Think of it as a clipboard for work-in-progress.

---

### Basic Stash Commands

#### Stash your changes

```bash
git stash              # stash tracked modified files (staged + unstaged)
git stash push         # identical to the above (explicit form)
```

#### Stash with a description

```bash
git stash push -m "WIP: half-finished login validation"
```

Always add a message when stashing — the default `WIP on branch: <hash> message` is hard to identify later.

#### Include untracked and ignored files

```bash
git stash push -u      # include untracked files (new files not yet git-added)
git stash push -a      # include untracked AND ignored files (e.g. build output)
```

---

### Viewing and Applying Stashes

#### List all stashes

```bash
git stash list
```

Output:

```
stash@{0}: On feature/login: WIP: half-finished login validation
stash@{1}: On main: quick experiment
stash@{2}: WIP on main: 3f9a12b Initial setup
```

Stashes are numbered with `{0}` being the most recent.

#### Apply and remove the most recent stash

```bash
git stash pop
```

Applies `stash@{0}` and removes it from the stash list. If there is a conflict, the stash is **not** removed — resolve the conflict first, then run `git stash drop`.

#### Apply a specific stash (keep it in the list)

```bash
git stash apply stash@{2}
```

Use `apply` when you want to apply a stash to multiple branches.

#### Delete stashes

```bash
git stash drop stash@{1}   # delete one specific stash
git stash clear            # delete ALL stashes (irreversible)
```

---

### Inspecting Stash Contents

```bash
git stash show              # summary of files changed in stash@{0}
git stash show -p           # full diff of stash@{0}
git stash show stash@{2}    # summary for a specific stash
git stash show -p stash@{2} # full diff for a specific stash
```

---

### Creating a Branch from a Stash

If your stash now conflicts with the current branch (because the branch has diverged), create a new branch from the stash instead:

```bash
git stash branch <branchname> stash@{0}
```

This:
1. Creates a new branch at the commit where the stash was made
2. Applies the stash to that branch
3. Drops the stash if successful

---

### Managing Multiple Stashes

The stash is a **stack** — last in, first out. When working with multiple stashes:

- Always use `-m` to label stashes with context
- Use `git stash list` often to keep track
- Avoid letting stashes accumulate — apply or drop regularly

---

### Real-World Scenario: Urgent Bug While Mid-Feature

```bash
# You're working on a feature
echo "half-done feature" > feature.js

# Urgent bug report comes in — you need to switch to main
git stash push -m "WIP: feature work half done"

# Switch context and fix the bug
git checkout main
git checkout -b hotfix/crash-on-login
# ... fix the bug, commit it ...
git checkout main
git merge hotfix/crash-on-login

# Return to your feature
git checkout feature/my-branch
git stash pop
# Your half-done changes are back, exactly as you left them
```

---

## Part 2: Git Cherry-pick

### What Cherry-pick Does

`git cherry-pick` takes the **changes introduced by a specific commit** and applies them to your current branch as a new commit. It uses the same diff but produces a **new commit with a new hash** — the commit exists on both branches independently.

---

### Basic Cherry-pick Commands

#### Apply a single commit

```bash
git cherry-pick <hash>
```

#### Apply multiple specific commits

```bash
git cherry-pick abc123 def456 789ghi
```

Applied in left-to-right order.

#### Apply a range of commits

```bash
git cherry-pick A..B       # commits after A up to and including B (A excluded)
git cherry-pick A^..B      # commits from A up to and including B (A included)
```

---

### Conflict Resolution During Cherry-pick

If the cherry-picked commit conflicts with your current branch:

```bash
# 1. Resolve conflicts manually in the affected files
# 2. Stage the resolved files
git add <resolved-file>
# 3. Continue the cherry-pick
git cherry-pick --continue
```

To abandon entirely:

```bash
git cherry-pick --abort
```

---

### Useful Cherry-pick Flags

#### `--no-commit` (`-n`): Apply without committing

```bash
git cherry-pick -n abc123
```

Applies the changes to your working tree and index but does **not** create a commit. Useful when you want to combine changes from multiple commits before committing, or when you want to review/modify before committing.

#### `-x`: Record the source commit

```bash
git cherry-pick -x abc123
```

Appends `(cherry picked from commit abc123)` to the commit message. Highly recommended when backporting fixes — makes traceability easy.

---

### Key Use Cases

#### 1. Backporting a Bug Fix

A critical fix was made on `main` but also needs to go to an older release branch:

```bash
git log main --oneline     # find the fix commit hash: abc123
git checkout release/1.4
git cherry-pick -x abc123
```

The fix is now on `release/1.4` with a traceable note in the commit message.

#### 2. Hotfix to Multiple Release Branches

```bash
git checkout release/2.0 && git cherry-pick -x abc123
git checkout release/1.9 && git cherry-pick -x abc123
git checkout release/1.8 && git cherry-pick -x abc123
```

#### 3. Rescuing a Commit on the Wrong Branch

You accidentally committed to `main` instead of `feature/my-branch`:

```bash
# Note the hash of the misplaced commit
git log --oneline    # hash: bad1234

# Apply it to the correct branch
git checkout feature/my-branch
git cherry-pick bad1234

# Remove it from main
git checkout main
git reset --hard HEAD~1   # only if not pushed; otherwise use git revert
```

---

### The Duplicate Commit Danger

Cherry-pick creates **independent copies** of a commit. If both branches are later merged together, Git may apply the same diff twice — causing duplicate changes or confusing conflicts.

To mitigate:

- Drop the original commit from the wrong branch after cherry-picking (when safe)
- Use `-x` so it's obvious the commit was cherry-picked
- Be aware when cherry-picking from branches that will eventually merge back

---

## Stash vs Cherry-pick — Comparison Table

| Aspect | `git stash` | `git cherry-pick` |
|---|---|---|
| Purpose | Temporarily save uncommitted changes | Apply a committed change to another branch |
| Operates on | Uncommitted working tree changes | Existing commits |
| Creates a new commit | No | Yes (new hash) |
| Typical use | Context switching mid-task | Backporting, hotfixing, rescuing commits |
| Affects history | No | Yes — adds a commit to current branch |
| Risk | Stash conflicts on apply | Duplicate commits if branches merge |

---

## Summary

- `git stash` saves uncommitted changes to a stack so you can switch contexts cleanly
- Use `-m` to label stashes; `-u` to include untracked files
- `pop` applies and removes the top stash; `apply` keeps it in the list
- `git stash branch` creates a branch from a stash, avoiding conflicts
- `git cherry-pick <hash>` applies a commit's diff to the current branch as a new commit
- Use `-x` when backporting to record the source commit
- Use `--no-commit` when you want to batch multiple cherry-picks before committing
- Main cherry-pick use cases: backporting fixes, hotfixing multiple branches, rescuing misplaced commits

---

## Knowledge Check

1. What is the difference between `git stash pop` and `git stash apply`?
2. You stash your changes, but when you try to `git stash pop` later there are conflicts. What happens to the stash entry?
3. You have `stash@{0}`, `stash@{1}`, `stash@{2}` in your list. Which is the most recent?
4. What does `git cherry-pick -x` add to the commit message and why is it useful?
5. Why can cherry-picking cause problems if both branches are eventually merged?

---

## Hands-On Exercise

### Part A: Stash Workflow

```bash
# Setup
git init stash-practice && cd stash-practice
git commit --allow-empty -m "Initial commit"

# Simulate mid-feature work
echo "half-done feature" > feature.js
git add feature.js

# An urgent issue requires you to switch context
git stash push -m "WIP: feature.js half done"
git stash list   # verify it's there

# Switch to main, apply a "fix"
echo "bugfix" > fix.js
git add fix.js && git commit -m "hotfix: critical bug"

# Restore your feature work
git stash pop
git status   # feature.js should be back as staged
```

### Part B: Cherry-pick Workflow

```bash
# Setup
git init cherry-practice && cd cherry-practice
git commit --allow-empty -m "Initial commit"

# Create a commit on main with a "bug fix"
echo "bugfix code" > fix.js
git add . && git commit -m "fix: resolve null pointer in auth"
git log --oneline   # note the hash, e.g. abc1234

# Create a release branch that needs the same fix
git checkout -b release/1.0 HEAD~1   # branch before the fix
git log --oneline   # fix is not here

# Cherry-pick the fix onto the release branch
git cherry-pick -x abc1234   # use your actual hash
git log --oneline   # fix is now here, with cherry-pick note
```

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="07-rebasing.md">← Previous: Rebasing</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="09-undoing-changes.md">Next: Undoing Changes →</a>
</div>
