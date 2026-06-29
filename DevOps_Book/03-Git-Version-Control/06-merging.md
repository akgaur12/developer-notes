# Chapter 06 — Merging

## Learning Objectives

By the end of this chapter you will be able to:

- Explain the difference between fast-forward and three-way merges
- Execute merges and interpret merge commits in the log
- Understand what merge conflicts are and how to resolve them manually
- Configure VS Code as a merge tool
- Use squash merges to produce a clean main branch history
- Choose the right merge strategy for your team's workflow

**Prerequisites:** Chapter 05 — Remote Repositories

---

## What Does Merge Do?

`git merge` incorporates the changes from one branch into another. The key property of merging is that it **preserves history** — the original commits on both branches remain exactly as they were, and Git records a permanent record of how they were combined.

This is the fundamental trade-off between merging and rebasing: merging is safe and honest about what happened; the log can become complex, but it is always accurate.

---

## Fast-Forward Merge

A fast-forward merge happens when the current branch has not diverged from the branch being merged in. In other words, every commit on the current branch is already an ancestor of the target branch.

### Condition

No new commits on `main` since `feature` branched off.

### Before

```
main:    A---B
                \
feature:         C---D   (HEAD)
```

Wait — actually in a fast-forward setup there is no divergence at all:

```
main (HEAD):  A---B
                   \
feature:            C---D
```

`main` points to `B`, and `feature` is two commits ahead. `B` is a direct ancestor of `D`.

### After

```bash
git switch main
git merge feature
```

```
main (HEAD):  A---B---C---D
feature:                  D   (still points here)
```

Git simply **moves the `main` pointer forward** to `D`. No new commit is created. The branch history is linear.

Git will tell you: `Fast-forward`

---

## Three-Way Merge

A three-way merge is required when both branches have diverged — each has commits the other does not.

### Condition

Both `main` and `feature` have new commits since they shared a common ancestor.

### Before

```
         C---D   feature
        /
A---B---E         main (HEAD)
```

- `A---B` is the shared history
- `E` is a new commit on `main` (e.g., a bugfix)
- `C---D` are new commits on `feature`
- The **common ancestor** is `B`

### The Three-Way Logic

Git uses three snapshots to compute the merge result:
1. The **common ancestor** (`B`)
2. The **current branch tip** (`E` on `main`)
3. The **incoming branch tip** (`D` on `feature`)

Git applies the changes from both sides relative to the common ancestor and combines them automatically — unless both sides changed the same lines.

### After

```bash
git switch main
git merge feature
```

```
         C---D   feature
        /       \
A---B---E---------M   main (HEAD)
```

`M` is the **merge commit**. It has two parents: `E` and `D`. This is what makes it special — most commits have one parent, but a merge commit has two (or more).

Git will tell you: `Merge made by the 'ort' strategy.`

---

## Executing a Merge

Always merge **into** the destination branch by switching to it first:

```bash
git switch main           # switch to the branch you want to update
git merge feature/login   # pull in the feature branch commits
```

### Merge Commits in the Log

```bash
git log --oneline --graph
```

```
*   a7f3c2e (HEAD -> main) Merge branch 'feature/login'
|\
| * 3b2e1d0 Add password validation
| * 9c1a4f2 Add login form
* | 7f8e0b1 Fix typo in README
|/
* 4d2c1a0 Initial project setup
```

The `|\` and `|/` characters show where the branch diverged and where it was merged back.

---

## --no-ff: Forcing a Merge Commit

```bash
git merge --no-ff feature/login
```

Even if a fast-forward is possible, `--no-ff` forces Git to create a merge commit. The branch history stays visible in the log.

### Why Teams Use --no-ff

Without `--no-ff`, fast-forwarded feature branches leave no trace in the log — you cannot tell which commits were part of a feature. With `--no-ff`, there is always an explicit merge commit you can revert, and the feature's branch name appears in the commit message.

```bash
# Configure --no-ff as the default for all merges:
git config merge.ff false

# Or only for a specific branch via branch protection on GitHub/GitLab
```

**Rule of thumb:**
- Personal projects: fast-forward is fine, keeps history clean
- Team workflows: `--no-ff` for feature branches so the branch structure is always visible

---

## Merge Conflicts

A conflict happens when Git cannot automatically resolve differences — specifically, when **both branches modified the same lines** of the same file (or one branch deleted a file the other modified).

### What Triggers a Conflict

- Both branches edited line 42 of `app.js` with different content
- One branch renamed a file, the other modified it
- Both branches added a different function at the same location

### Conflict Markers

When a conflict occurs, Git pauses the merge and marks the conflicting sections in the file:

```
<<<<<<< HEAD
button.style.color = "blue";
=======
button.style.color = "red";
>>>>>>> feature/branding
```

- `<<<<<<< HEAD` — start of your current branch's version
- `=======` — separator between the two versions
- `>>>>>>> feature/branding` — end of the incoming branch's version

A file can have multiple conflict regions. **You must resolve every single one.**

---

## Resolving Conflicts Manually

### Step 1: Identify Conflicting Files

```bash
git status
# Both modified: src/styles.css
# Both modified: src/app.js
```

### Step 2: Edit Each Conflicted File

Open the file in your editor. Decide what the final content should be — it might be one version, the other, or a combination of both. Remove the conflict markers completely.

Before:
```
<<<<<<< HEAD
button.style.color = "blue";
=======
button.style.color = "red";
>>>>>>> feature/branding
```

After (you decided to keep both with a comment):
```
/* brand color from feature/branding, overrides dev default */
button.style.color = "red";
```

### Step 3: Stage the Resolved File

```bash
git add src/styles.css
git add src/app.js
```

### Step 4: Complete the Merge

```bash
git commit
# Git will open an editor with a pre-filled message like:
# "Merge branch 'feature/branding'"
# Save and close to finalize
```

---

## Using a Merge Tool

### Configure VS Code as Merge Tool

```bash
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
```

Then when a conflict occurs:

```bash
git mergetool
```

VS Code opens with a three-panel view:
- Left panel: your current branch version (HEAD)
- Right panel: incoming branch version
- Bottom panel: the merged result (editable)

You click "Accept Current", "Accept Incoming", "Accept Both", or edit the result panel directly. Save and close VS Code; Git detects the resolution.

Other popular merge tools: `vimdiff`, `meld`, `kdiff3`, `IntelliJ IDEA`.

---

## Aborting a Merge

If you started a merge and realize it is getting too complex:

```bash
git merge --abort
```

This returns your repository to the exact state it was in before you ran `git merge`. All partial conflict resolutions are discarded.

---

## Squash Merge

```bash
git switch main
git merge --squash feature/user-auth
git commit -m "Add user authentication feature"
```

Squash merge takes all commits from `feature/user-auth`, combines their changes into a single staged set, and lets you write one clean commit message. It does **not** create a merge commit with two parents — it creates a regular single-parent commit on `main`.

### What Gets Lost

The individual commits on `feature/user-auth` are not in `main`'s history. If you later need to see the step-by-step development, you must look at the feature branch itself (if it still exists).

### When to Use Squash

- The feature branch has messy "WIP", "fix typo", "oops" commits
- You want a clean, readable `main` history
- The feature is small and the detailed branch history has no long-term value

---

## Merge Strategy Comparison

| Strategy | Command | Merge Commit? | History | When to Use |
|---|---|---|---|---|
| Fast-forward | `git merge` | No | Linear | Personal projects, simple updates |
| Three-way | `git merge` | Yes (auto) | Shows divergence | When branches diverged |
| No fast-forward | `git merge --no-ff` | Always | Branch visible | Team feature workflows |
| Squash | `git merge --squash` | No (1 parent) | Clean main | Messy feature branches |

---

## When to Use Each Strategy

### Fast-Forward

Use when you are keeping a personal branch in sync with main and want a linear, easy-to-read history. Common for `git pull` on your own branches.

### Three-Way (Default)

Git's automatic behavior when branches have diverged. Appropriate when you want to faithfully record that two parallel workstreams existed.

### --no-ff

Enforced by many teams for all feature branches. Provides a clean "this was a feature" marker in history, makes reverting a full feature trivial (`git revert -m 1 <merge-commit-hash>`), and keeps the branch name visible in the log.

### Squash

Best for teams that value a clean, bisectable `main` history over complete branch-level detail. Often combined with pull request workflows where the PR diff serves as the historical record of the feature's development.

---

## Summary

- Fast-forward moves the branch pointer — no merge commit, linear history
- Three-way creates a merge commit with two parents, accurately recording that two histories were joined
- `--no-ff` forces a merge commit even when fast-forward is possible, preserving branch context
- Conflicts happen when both branches changed the same lines; you resolve them by editing files, removing markers, staging, and committing
- `git merge --abort` gets you out of a messy merge cleanly
- Squash merge collapses all feature commits into one — clean history, but you lose the commit-by-commit record

---

## Knowledge Check

1. You have `main` at commit `B` and `feature` at commit `D` (where `D`'s chain starts from `B`). What type of merge will Git perform, and will a merge commit be created?
2. You are in the middle of resolving a conflict and you run `git status`. The file shows "both modified". What does that mean, and what are your next steps?
3. A coworker wants to use `git merge --squash` for all feature branches. What is the main downside you should warn them about?
4. What is the purpose of `git revert -m 1 <merge-commit>`, and why does `-m 1` matter for merge commits specifically?

---

## Hands-On Exercise

```bash
# 1. Set up a conflict scenario
git init merge-lab && cd merge-lab
echo "color: blue;" > style.css
git add style.css && git commit -m "Initial styles"

# 2. Create a feature branch and modify the same line
git switch -c feature/red-theme
sed -i 's/blue/red/' style.css
git add style.css && git commit -m "Change color to red"

# 3. Switch to main and make a conflicting change
git switch main
sed -i 's/blue/green/' style.css
git add style.css && git commit -m "Change color to green"

# 4. Try to merge — this will conflict
git merge feature/red-theme
# Git reports: CONFLICT (content): Merge conflict in style.css

# 5. Inspect the conflict
cat style.css
git status

# 6. Resolve manually
# Edit style.css to your chosen resolution, remove all markers
echo "color: purple;  /* merged: combined both themes */" > style.css
git add style.css
git commit -m "Merge feature/red-theme — adopt purple as compromise"

# 7. Visualize the result
git log --oneline --graph --all

# 8. Try a squash merge
git switch -c feature/squash-demo
echo "font-size: 16px;" >> style.css && git commit -am "Set base font size"
echo "font-family: sans-serif;" >> style.css && git commit -am "Set font family"
echo "line-height: 1.5;" >> style.css && git commit -am "Set line height"

git switch main
git merge --squash feature/squash-demo
git status    # all three changes are staged but no commit yet
git commit -m "Apply typography defaults"
git log --oneline --graph --all  # one clean commit, no merge commit
```

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./05-remote-repositories.md">← Previous: Remote Repositories</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./07-rebasing.md">Next: Rebasing →</a>
</div>
