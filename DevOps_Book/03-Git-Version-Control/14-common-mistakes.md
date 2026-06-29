# Chapter 14 — Common Git Mistakes & Fixes

## Learning Objectives

By the end of this chapter, you will be able to:

- Recognize the most common Git mistakes by their symptoms
- Apply the correct recovery procedure for each mistake
- Understand which mistakes are recoverable and which are not
- Use `git reflog` and related tools as an emergency recovery toolkit

**Prerequisites:** Chapter 13 — Git Best Practices

---

## Overview

Mistakes in Git are normal — even experienced developers make them. What separates confident Git users from anxious ones is knowing how to recover. Git's design is intentionally safe: most operations are reversible, and very little data is truly lost unless you explicitly ask Git to delete it.

This chapter is structured as:

> **Mistake → Symptom → Cause → Fix**

Work through it as a reference. You do not need to memorize every fix — knowing that a fix *exists* and roughly how to find it is enough. When a real mistake happens, come back here.

---

## Mistake 1 — Committed to the Wrong Branch

### Symptom

Your latest commit landed on `main` but it should have been on `feature/xyz`.

### Cause

Forgot to switch branches before starting work, or switched to the wrong branch.

### Fix — Work not yet committed

```bash
# Stash your current (unstaged) work
git stash

# Switch to the right branch
git switch feature/xyz

# Reapply your work
git stash pop
```

### Fix — Work already committed

```bash
# Step 1: Copy the commit to the correct branch
git switch feature/xyz
git cherry-pick main   # cherry-pick the tip of main

# Step 2: Remove the commit from the wrong branch
git switch main
git reset HEAD~1       # soft reset by default — keeps your changes staged
```

Use `--soft` (default) to keep changes staged, `--mixed` to keep them unstaged, or `--hard` to discard entirely (dangerous).

---

## Mistake 2 — Pushed and Forgot to Pull First

### Symptom

```
! [rejected]  main -> main (fetch first)
error: failed to push some refs to 'origin'
hint: Updates were rejected because the remote contains work that you do not have locally.
```

### Cause

A teammate pushed commits to the remote after your last `git pull`. Your local and remote histories have diverged.

### Fix

```bash
# Rebase your commits on top of the remote's commits (preferred — keeps linear history)
git pull --rebase origin main

# Resolve any conflicts that arise, then:
git push origin main
```

Avoid `git pull` without `--rebase` if you want to keep history clean — a plain pull creates an unnecessary merge commit.

---

## Mistake 3 — Accidentally Committed a Secret / API Key

### Symptom

An API key, password, or private key appears in git history.

### Cause

`.env` or credentials file was not in `.gitignore`, or was explicitly `git add`-ed.

### Fix

> **CRITICAL: Rotate the secret FIRST before anything else. Assume it is compromised.**

```bash
# Step 1: Rotate/revoke the exposed credential in your provider dashboard

# Step 2: Remove the file from all history
pip install git-filter-repo
git filter-repo --path .env --invert-paths

# Step 3: Force push all branches and tags
git push --force --all
git push --force --tags

# Step 4: Notify all collaborators — they must re-clone
# Step 5: Notify your security team if the secret had production access
```

See Chapter 13 for prevention strategies.

---

## Mistake 4 — Made Commits in Detached HEAD State

### Symptom

```
HEAD detached at abc1234
```

You checked out a specific commit or tag directly, then made new commits. These commits exist but are not on any branch — Git will garbage-collect them eventually.

### Fix

```bash
# While still in detached HEAD state — create a new branch right here
git branch rescue-branch

# Or combined:
git switch -c rescue-branch

# Now your commits are safe on rescue-branch
# Merge into your target branch when ready
git switch main
git merge rescue-branch
```

---

## Mistake 5 — Accidentally Deleted a Branch

### Symptom

```bash
git branch -D feature/important-work
# Branch is gone and you realize it had unmerged commits
```

### Fix

Git keeps references to commits in `reflog` for 90 days by default. The commits still exist — only the branch label was deleted.

```bash
# Step 1: Find the last commit that was on the deleted branch
git reflog | grep "feature/important-work"
# Or just scroll through git reflog to find recent commits

# You will see entries like:
# abc1234 HEAD@{3}: commit: feat: add important feature

# Step 2: Recreate the branch at that commit
git checkout -b feature/important-work abc1234
```

---

## Mistake 6 — `git reset --hard` Lost Uncommitted Work

### Symptom

You ran `git reset --hard` and your working directory changes are gone.

### Cause

`--hard` resets both the index (staging area) and the working tree. Changes that were not committed and not stashed are overwritten.

### Reality

If the work was **never committed and never stashed**, it is usually **unrecoverable**. Git only tracks committed objects and stashed changes in its object database.

### Partial Recovery Attempt

```bash
# Git's "lost and found" for orphaned objects
git fsck --lost-found

# Check the output in .git/lost-found/other/
# These are blob objects that exist but aren't reachable from any reference
ls .git/lost-found/other/
git show .git/lost-found/other/<blob-hash>
```

This is not guaranteed to find your work, especially for recently created files.

### Prevention

Always `git stash` or `git commit` before running any `reset --hard`.

```bash
# Safe habit before any destructive operation
git stash push -m "safety stash before reset"
git reset --hard origin/main
```

---

## Mistake 7 — Merge Conflict Made Worse (Corrupted File)

### Symptom

You started resolving a merge conflict but made it worse. The file now has leftover conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) or broken code.

### Fix — If Still Mid-Merge

```bash
# Abort the entire merge and go back to the pre-merge state
git merge --abort
```

### Fix — Accept One Side Entirely

```bash
# Keep your version (current branch)
git checkout --ours -- path/to/file

# Keep their version (incoming branch)
git checkout --theirs -- path/to/file

# Stage the resolved file
git add path/to/file
git merge --continue
```

### Fix — Restore to Pre-Conflict State for One File

```bash
# Get the file from the commit before the merge started
git show HEAD:path/to/file > path/to/file
```

---

## Mistake 8 — 20 WIP Commits Cluttering History Before a PR

### Symptom

Your branch has commits like:
```
wip
wip2
fix typo
fix
actually fix
ok now it works
remove debug
```

### Fix

Use interactive rebase to squash them into meaningful commits before opening a PR:

```bash
# Squash the last 20 commits interactively
git rebase -i HEAD~20
```

In the editor, change `pick` to `squash` (or `s`) for commits you want to fold into the one above, or `reword` (`r`) to keep as a separate commit but change the message:

```
pick abc1234 feat(auth): add OAuth2 login
squash def5678 wip
squash ghi9012 fix
squash jkl3456 fix typo
reword mno7890 fix(auth): handle token expiry edge case
```

After saving, Git opens an editor for each merged commit message.

---

## Mistake 9 — Pushed Directly to Main (Bypassed Branch Protection)

### Symptom

You pushed a broken commit directly to `main` instead of going through a PR.

### Fix

Do **not** `git reset --hard` on a shared branch — this rewrites shared history and breaks teammates' clones.

Instead, create a revert commit:

```bash
# Revert the bad commit (creates a new commit that undoes it)
git revert HEAD

# Push the revert commit
git push origin main
```

If multiple commits need reverting:

```bash
git revert HEAD~3..HEAD   # revert last 3 commits
```

Then open a proper PR to re-introduce the changes the right way.

---

## Mistake 10 — Wrong Commit Message (Already Pushed)

### Fix — Not Yet Pushed

```bash
git commit --amend -m "fix(auth): correct error message for invalid token"
```

### Fix — Already Pushed to a Personal/Feature Branch

```bash
# Use interactive rebase to reword a past commit
git rebase -i HEAD~5
# Change 'pick' to 'reword' for the target commit

# Force push is required after rewriting history
git push --force-with-lease origin feature/my-branch
```

Use `--force-with-lease` instead of `--force` — it will refuse to push if someone else has pushed to the branch since your last fetch, preventing accidental overwrites.

### Already Pushed to a Shared Branch

**Do not rewrite shared history.** The wrong message is a minor cosmetic issue — it is not worth the coordination cost of a force push on a shared branch. Accept the bad message and move on.

---

## Mistake 11 — .gitignore Not Ignoring a File

### Symptom

You added a file to `.gitignore` but `git status` still shows it as modified/untracked.

### Cause

The file was already committed to the repository. `.gitignore` only affects **untracked** files. If a file is already tracked, `.gitignore` entries for it are silently ignored.

### Fix

```bash
# Step 1: Remove the file from the index (Git's tracking), keep local file
git rm --cached path/to/file

# For a directory
git rm -r --cached path/to/dir/

# Step 2: Commit the removal
git commit -m "chore: stop tracking .env file"

# Now .gitignore will work going forward
```

---

## Mistake 12 — Cloned with HTTPS, Want SSH

### Symptom

Every `git push` or `git pull` prompts for a username and password (or token), because you cloned with an HTTPS URL.

### Fix

```bash
# Check current remote URL
git remote -v

# Update to SSH
git remote set-url origin git@github.com:username/repo.git

# Verify
git remote -v
```

Now all operations use your SSH key for authentication.

---

## Mistake 13 — Accidentally Staged Everything Including Junk Files

### Symptom

You ran `git add .` or `git add -A` and now your staging area includes generated files, editor temporaries, and other things you did not mean to commit.

### Fix

```bash
# Unstage everything (keeps all changes in working directory)
git restore --staged .

# Now add files selectively
git add src/feature.py
git add tests/test_feature.py

# Or use patch mode to stage parts of files
git add -p
```

---

## Mistake 14 — Rebased a Shared Branch (Broke Teammates)

### Symptom

After you force-pushed a rebased branch, teammates see:

```
Your branch and 'origin/feature/shared-work' have diverged,
and have 5 and 3 different commits each, respectively.
```

### Cause

Rebasing rewrites commit hashes. A force push replaces the remote's history. Anyone who already has the old commits now has a diverged history.

### Fix — For Each Affected Teammate

```bash
# Do NOT try to merge or pull normally — it will create a mess
git fetch origin
git reset --hard origin/feature/shared-work
```

This discards their local commits on the branch and replaces them with the remote's (now rebased) version. **They will lose any uncommitted work on this branch** — warn them before they do this.

### Prevention

- Never rebase branches that multiple people are actively working on
- Only rebase your own private feature branches before opening a PR
- If you must rebase a shared branch, coordinate with the team, do it when no one is actively working on it, and send a clear message before force-pushing

---

## Emergency Recovery Cheat Sheet

When something goes wrong and you are not sure what happened, work through this list:

```bash
# 1. See everything that happened (commits, checkouts, resets, merges)
git reflog

# 2. See all commits across all branches (including orphaned ones)
git log --all --oneline --graph

# 3. Restore a specific file from any point in history
git checkout <commit-hash> -- path/to/file

# 4. See exactly what a specific commit changed
git show <commit-hash>

# 5. Find commits that changed a specific file
git log --all --oneline -- path/to/file

# 6. Search commit messages for a keyword
git log --all --oneline --grep="keyword"

# 7. Find orphaned objects (last resort for lost uncommitted work)
git fsck --lost-found

# 8. See what is in a stash without applying it
git stash show -p stash@{0}
```

**The golden rule:** before doing anything unfamiliar, run `git status` and `git log --oneline -5` to understand where you are. And always check `git reflog` first when something seems lost.

---

## Summary

| Mistake | Quick Fix |
|---------|-----------|
| Wrong branch | `cherry-pick` + `reset HEAD~1` |
| Rejected push | `git pull --rebase` |
| Committed secret | Rotate → `filter-repo` → force push |
| Detached HEAD commits | `git switch -c new-branch` |
| Deleted branch | `git reflog` → `git checkout -b <branch> <hash>` |
| `reset --hard` lost work | `git fsck --lost-found` (partial recovery only) |
| Corrupted merge conflict | `git merge --abort` or `checkout --ours/--theirs` |
| Messy WIP commits | `git rebase -i HEAD~N` |
| Direct push to main | `git revert HEAD` |
| Bad commit message | `--amend` (not pushed) / `rebase -i reword` (pushed to own branch) |
| .gitignore not working | `git rm --cached` → commit removal |
| HTTPS vs SSH | `git remote set-url origin git@...` |
| Staged junk files | `git restore --staged .` |
| Rebased shared branch | Coordinate → `git reset --hard origin/<branch>` |

---

## Knowledge Check

1. What is the safest way to undo a bad commit on a shared branch?
2. What is the first thing you must do when you discover a committed secret?
3. What command would you use to find a deleted branch's last commit hash?
4. Why does `.gitignore` sometimes not work for a file that is listed in it?
5. What does `--force-with-lease` do differently from `--force`?
6. What does `git merge --abort` do, and when can you use it?
7. What git command is your first tool when something unexpectedly disappears?

---

## Hands-On Exercise

### Deliberately Trigger and Recover from 3 Mistakes

Set up a safe test repository:

```bash
mkdir git-mistakes-lab && cd git-mistakes-lab
git init
git commit --allow-empty -m "chore: initial commit"
```

**Scenario A — Wrong branch commit**

```bash
# Make a commit on main by accident
echo "feature code" > feature.py
git add feature.py
git commit -m "feat: add feature (on wrong branch!)"

# Recover: move it to the right branch
git branch feature/correct-branch
git reset HEAD~1
git switch feature/correct-branch
git add feature.py
git commit -m "feat: add feature (now on right branch)"
```

**Scenario B — Deleted branch recovery**

```bash
git switch -c feature/delete-me
echo "important work" > work.py
git add work.py
git commit -m "feat: important work"

git switch main
git branch -D feature/delete-me   # oops

# Recover
git reflog | head -10
# find the hash of "feat: important work"
git checkout -b feature/delete-me <hash>
```

**Scenario C — Messy WIP commits cleanup**

```bash
git switch -c feature/cleanup-demo
for i in 1 2 3 4 5; do
  echo "change $i" >> file.txt
  git add file.txt
  git commit -m "wip $i"
done

# Clean up with interactive rebase
git rebase -i HEAD~5
# Squash all into one commit with a proper message
```

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./13-best-practices.md">← Previous: Best Practices</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./15-projects.md">Next: Hands-On Projects →</a>
</div>
