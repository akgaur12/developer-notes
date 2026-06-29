# Chapter 07 — Rebasing

## Learning Objectives

By the end of this chapter you will be able to:

- Explain what `git rebase` does at the commit-graph level
- Articulate the Golden Rule of rebasing and why violating it causes problems
- Use interactive rebase to squash, reword, reorder, edit, and drop commits
- Use `git rebase --onto` to transplant a branch to a different base
- Choose between merge and rebase based on team policy and context

**Prerequisites:** Chapter 06 — Merging

---

## What Rebase Does

Rebasing **replays your commits on top of another branch's tip**, creating brand-new commits with new hashes. Each original commit is re-applied one by one onto the new base, producing a rewritten, linear history.

### Before Rebase

```
main:    A --- B --- C
              \
feature:       D --- E
```

`feature` branched from `B`. Meanwhile `main` advanced to `C`.

### After: `git rebase main` (from feature branch)

```
main:    A --- B --- C
                      \
feature:               D' --- E'
```

`D'` and `E'` are **new commits** — same diff content as `D` and `E`, but different parent hashes. The old `D` and `E` are abandoned (and will eventually be garbage-collected).

### Running the Rebase

```bash
git checkout feature
git rebase main
```

Or in one step (Git 2.x+):

```bash
git rebase main feature
```

---

## Rebase vs Merge — Comparison Table

| Aspect | `git merge` | `git rebase` |
|---|---|---|
| History shape | Non-linear (preserves true branching) | Linear (as if work happened sequentially) |
| Merge commit | Yes — adds an extra merge commit | No — no extra commit |
| Readability | Can be noisy in `git log` | Clean, easy to follow |
| Rewrites history | No | Yes — new hashes for rebased commits |
| Safe on shared branches | Yes | **No** (see Golden Rule) |
| Conflict handling | Resolve once at merge point | Resolve at each replayed commit |

---

## The Golden Rule of Rebasing

> **NEVER rebase commits that have been pushed to a shared remote branch.**

### Why This Rule Exists

When you rebase, Git creates **new commits with new hashes**. If teammates have already pulled your original commits and based their own work on them, their history now refers to hashes that no longer exist in your rewritten branch. The result:

- Their pull/merge will fail or produce duplicate commits
- Force-pushing overwrites the remote — their local branches become orphaned
- Recovering the team's repos is painful and error-prone

### When Rebase IS Safe

- Cleaning up **local commits** before your first push
- On a **private feature branch** that only you use
- With explicit team agreement and coordination (advanced scenarios)

---

## Interactive Rebase

Interactive rebase lets you edit the commit list before replaying — reorder, combine, delete, or reword commits.

```bash
git rebase -i HEAD~N    # rewrite the last N commits
git rebase -i <hash>    # rewrite commits after <hash>
```

Git opens your editor with a list like:

```
pick a1b2c3 Add login form
pick d4e5f6 WIP: validation
pick 789abc WIP: more validation
pick 012def WIP: edge cases
pick 345678 Fix typo in login form
```

### Available Commands

| Command | What It Does |
|---|---|
| `pick` | Keep the commit as-is |
| `reword` | Keep the commit but edit its message |
| `edit` | Pause after applying — lets you amend the commit content |
| `squash` | Merge into the previous commit; opens editor to combine messages |
| `fixup` | Merge into the previous commit; **discard** this commit's message |
| `drop` | Delete the commit entirely |

### Example 1: Squash 5 WIP Commits into 1 Clean Commit

Change the list to:

```
pick a1b2c3 Add login form
squash d4e5f6 WIP: validation
squash 789abc WIP: more validation
squash 012def WIP: edge cases
fixup 345678 Fix typo in login form
```

Git will combine all five into a single commit. The `squash` lines open an editor so you can write a unified message; `fixup` discards the message silently.

### Example 2: Reordering Commits for Logical Grouping

Simply cut and paste the `pick` lines into a different order in the editor. Git replays them in the new sequence.

### Splitting a Commit

1. Mark the target commit as `edit`
2. When Git pauses, run `git reset HEAD^` — this unstages the commit's changes back to your working tree
3. Re-add and commit in smaller pieces:
   ```bash
   git add file1.js
   git commit -m "Part 1: add input validation"
   git add file2.js
   git commit -m "Part 2: add error messages"
   ```
4. Continue the rebase: `git rebase --continue`

---

## Advanced Rebase: `--onto`

Transplant a branch to a completely different base:

```bash
git rebase --onto <newbase> <upstream> <branch>
```

**Scenario:** `feature` branched off `dev`, but you want it on `main` instead.

```bash
git rebase --onto main dev feature
```

This takes commits from `feature` that are **not** in `dev` and replays them onto `main`.

---

## Mid-Rebase Controls

| Command | Purpose |
|---|---|
| `git rebase --abort` | Abandon the rebase entirely; restore original state |
| `git rebase --continue` | After resolving a conflict, continue replaying |
| `git rebase --skip` | Skip the current conflicting commit and proceed |

When a conflict occurs during rebase:

```bash
# 1. Resolve the conflict in the file
# 2. Stage the resolution
git add <file>
# 3. Continue
git rebase --continue
```

---

## `git pull --rebase`

By default, `git pull` does a merge when integrating remote changes. With `--rebase`, your local commits are replayed on top of the fetched commits instead — keeping history linear.

```bash
git pull --rebase origin main
```

Make this the default for all pulls:

```bash
git config --global pull.rebase true
```

This is a popular team setting for projects that prefer linear history.

---

## When to Use Merge vs Rebase

| Situation | Recommendation |
|---|---|
| Integrating a completed feature into `main` | Merge (preserves branch context) |
| Keeping a feature branch up to date with `main` | Rebase (keeps history clean) |
| Cleaning up local commits before PR | Interactive rebase |
| Commits already pushed to shared branch | Merge — never rebase |
| Team prefers linear history | Rebase with "no rebase after push" rule |

The most important factor is **team policy**. Agree on a convention and stick to it consistently.

---

## Summary

- Rebase replays commits onto a new base, producing new hashes and a linear history
- Merge preserves true branching; rebase rewrites history
- **Golden Rule:** never rebase commits pushed to a shared branch
- Interactive rebase (`-i`) lets you squash, reword, reorder, edit, and drop commits
- `--onto` transplants a branch to an entirely different base
- `git pull --rebase` keeps pull history linear

---

## Knowledge Check

1. What is the key difference between the commits produced by `git rebase` versus `git merge`?
2. Why does rebasing a pushed branch cause problems for teammates?
3. What is the difference between `squash` and `fixup` in interactive rebase?
4. You have 4 local WIP commits you want to clean up before opening a PR. What command do you run?
5. How would you abort a rebase that has gone wrong mid-way?

---

## Hands-On Exercise

**Goal:** Clean up a messy feature branch using interactive rebase.

```bash
# Setup: create a test repo and messy branch
git init rebase-practice && cd rebase-practice
git commit --allow-empty -m "Initial commit"
git checkout -b feature

# Make several messy commits
echo "feature code" > feature.js && git add . && git commit -m "WIP"
echo "more code" >> feature.js && git add . && git commit -m "WIP 2"
echo "fix typo" >> feature.js && git add . && git commit -m "oops"
echo "final" >> feature.js && git add . && git commit -m "done hopefully"
echo "one more fix" >> feature.js && git add . && git commit -m "really done"

# View the messy history
git log --oneline

# Now clean it up
git rebase -i HEAD~5
# In the editor:
# - Mark the first as 'pick' or 'reword' to give it a proper name
# - Mark the rest as 'fixup' to merge them silently

# Verify the result
git log --oneline
```

**Expected result:** 5 commits collapsed into 1 clean commit with a meaningful message.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="06-merging.md">← Previous: Merging</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="08-stash-and-cherry-pick.md">Next: Stash & Cherry-pick →</a>
</div>
