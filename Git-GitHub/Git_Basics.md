# 📘 Git Basics & Commands (With Rebase, Merge & Cherry-Pick)


## 🌱 What is Git?
Git is a **distributed version control system (DVCS)** used to track changes in source code, collaborate with teams, and manage project history efficiently.

---

## 🧠 Core Git Concepts

| Term | Meaning |
|----|----|
Repository | Project tracked by Git |
Working Directory | Your local files |
Staging Area | Area to prepare changes |
Commit | Snapshot of changes |
Branch | Parallel line of development |
HEAD | Pointer to current branch |
Remote | Remote repository (GitHub/GitLab) |

---


## 🔄 Git Workflow


```
┌─────────────────────────────────────────── LOCAL REPO ─────────────────────────────────────────┐───── REMOTE REPO ───┐
│                                                                                                │                     |
│  ┌──────────┐   ┌──────────────┐  ┌──────────────────┐ ┌──────────────┐  ┌──────────────────┐  │   ┌────────────┐    │
│  │  Stash   │   │  Working     │  │  Index /         │ │  Local       │  │  Remote-tracking │  │   │   Remote   │    │
│  │  Area    │   │  Tree        │  │  Staging Area    │ │  Branch      │  │  Ref             │  │   │   Branch   │    │
│  └──────────┘   └──────────────┘  └──────────────────┘ └──────────────┘  └──────────────────┘  │   └────────────┘    │
│        |               │                   │                  │                   │            │          │          │
│        │               │ git add           │                  │                   │            │          │          │
│        │               ├──────────────────▶│                  │                   │            │          │          │
│        │               │                   │ git commit       │                   │            │          │          │
│        │               │                   ├─────────────────▶│                   │            │          │          │
│        │               │                   │                  │   git push        │            │          │          │
│        │               │                   │                  ├──────────────────────────────────────────▶│          |
│        |               │                   │                  │                   │            │          │          │
│        │               │                   │                  │                   │  git fetch │          │          │
│        │               │                   │                  │                   |◀──────────────────────┤          |
│        |               │                   │                  │                   │            │          │          │
│        │               │                   │                  │   git pull        │            │          │          │
│        |               |◀─────────────────────────────────────────────────────────────────────────────────┤          |
│        │               │                   │                  │                   │            │          │          │
│        │               │  git checkout     │                  │                   │            │          │          │
│        |               |◀─────────────────────────────────────┤                   │            │          │          │
│        |               |                   |                  |                   |            │          |          │
│        │               │  git merge/rebase │                  │                   │            │          │          │
│        |               |◀─────────────────────────────────────────────────────────┤            │          │          │
│        │               │                   │                  │                   │            │          │          │
│        │               │                   │                  │                   │            │          │          │
│        │ git stash pop │                   │                  │                   │            │          │          │
│        └──────────────▶│                   │                  │                   │            │          │          │
│        |               │                   │                  │                   │            │          │          │
│        │   git stash   │                   │                  │                   │            │          │          │
│        |◀──────────────┘                   │                  │                   │            │          │          │
│        |               |                   |                  |                   |            │          |          │
│        |               |                   |                  |                   |            │          |          │
└────────────────────────────────────────────────────────────────────────────────────────────────┘─────────────────────┘

```

### Legend
- **Stash Area**: Temporary storage for changes
- **Working Tree**: Files you edit locally
- **Index / Staging Area**: Files prepared for commit
- **Local Branch**: Commits on your branch (e.g. `main`)
- **Remote-tracking Ref**: Local reference to remote state (e.g. `origin/main`)
- **Remote Branch**: Actual branch on remote repository


---

## ⚙️ Git Configuration Levels

Git supports **three configuration levels**:

| Level | Scope | File | Desc
|----|----|----|----|
System | All users | `/etc/gitconfig` | Applies to all users on the machine |
Global | Single user | `~/.gitconfig` | Applies to all repositories of a user |
Local | Single repository | `.git/config` | Applies to only one repository |

**Priority order:**  
```
Local > Global > System
```

```bash
# System Level (Least used)
git config --system user.name "System User"

# Global Level (User-specific)
git config --global user.name "Your Name"
git config --global user.email "you@email.com"

# Local Level (Repository-specific)
git config --local user.name "Repo Specific Name"
git config --local user.email "repo@email.com"

# View all configs
git config --list

# View specific level
git config --system --list
git config --global --list
git config --local --list
```


**Initial Branch Name**

By default Git will create a branch called master when you create a new repository with git init. From Git version 2.28 onwards, you can set a different name for the initial branch. To set main as the default branch name do
```bash
git config --global init.defaultBranch main
```

---

## 📁 Repository Commands

```bash
git init
git clone <repo-url>
```

---

## 📌 Status & Inspection

```bash
git status
git log --oneline --graph --all
git diff
git diff --staged
```

---

## ➕ Add & Commit

```bash
git add .
git commit -m "Meaningful message"
git commit --amend
```

---

## 🌿 Branching

```bash
git branch
git branch feature-x
git checkout feature-x
git checkout -b feature-x
git branch -d feature-x
```

---

## 🌍 Remote Operations

```bash
git remote -v
git push origin main
git pull origin main
git fetch origin
```

---

## ⚠️ Undo & Fix

```bash
git restore file.txt
git restore --staged file.txt
git reset --soft HEAD~1
git reset --hard HEAD~1
```

---

## 🧪 Stash

```bash
git stash
git stash list
git stash apply
```

---

## 📄 .gitignore Example

```gitignore
.env
__pycache__/
node_modules/
*.log
```

---

# 🔀 Git Merge vs Git Rebase vs Git Cherry-Pick

---

## 1️⃣ Git Merge

### What it does
- Combines branches
- Preserves full history
- Creates a **merge commit**

### Diagram (Merge)

```
main:     A───B───────E
              \     /
feature:       C───D
```

### Command
```bash
git checkout main
git merge feature
```

### Pros
- Safe
- Keeps history intact
- Best for shared branches

### Cons
- Commit history can get messy

---

## 2️⃣ Git Rebase

### What it does
- Rewrites commit history
- Moves feature commits on top of main
- No merge commit

### Diagram (Rebase)

Before:
```
main:     A───B
              \
feature:       C───D
```

After:
```
main:     A───B───C'───D'
```

### Command
```bash
git checkout feature
git rebase main
```

### Pros
- Clean, linear history
- Easy to read logs

### Cons
- Dangerous on shared branches
- Rewrites commit history

### ⚠️ Golden Rule
> Never rebase a branch that others are using.

### How to Rebase when Local Branch is Behind

Someone pushed changes before you, so your local branch is behind the remote branch.
If you want your commit to appear after their changes, you should rebase your commit on top of the latest remote branch.

#### 
```bash
# 1️⃣ Fetch the latest changes
git fetch origin

# 2️⃣ Rebase your commit on top of the latest branch
# (assuming branch name is A)

git rebase origin/A


# This will:
# - First apply their commits
# - Then apply your commit on top

# So history becomes:`
# their commit
# their commit
# your commit


# 3️⃣ If there are conflicts
# Git will stop and ask you to fix them.

# After fixing:
git add .
git rebase --continue

# 4️⃣ Push your changes
git push origin A
```


## 3️⃣ Git Cherry-Pick

### What it does
- Applies **a specific commit** from one branch to another
- Does NOT merge the full branch
- Creates a new commit with a new hash

### When to use
- Hotfix from feature → main
- Pick one bug fix without other changes
- Backport fixes to release branches

### Diagram (Cherry-Pick)

```
feature:   A───B───C
                 |
main:      D───E───C'
```

Only commit `C` is copied to `main` as `C'`.

### Command
```bash
git checkout main
git cherry-pick <commit-hash>
```

Cherry-pick multiple commits:
```bash
git cherry-pick <hash1> <hash2>
```

### Abort cherry-pick (on conflict):
```bash
git cherry-pick --abort
```

### Pros
- Very precise
- No unwanted commits
- Great for production hotfixes

### Cons
- Duplicate commits
- Can cause conflicts
- History divergence if overused

---

## 🆚 Merge vs Rebase vs Cherry-Pick

| Feature | Merge | Rebase | Cherry-Pick |
|------|------|------|-----------|
History | Preserved | Rewritten | Partially copied |
Commit Graph | Messy | Clean | Divergent |
Use Case | Combine branches | Clean local history | Pick specific commit |
Safe for shared branch | ✅ Yes | ❌ No | ⚠️ Careful |

---

## 🧠 Interview One-Liners

- **Merge**: combine full branch safely
- **Rebase**: rewrite history for clean commits
- **Cherry-pick**: copy a specific commit

---

## ⭐ Daily Git Commands

```bash
git status
git add .
git commit -m "msg"
git pull
git push
```

---


### ✍️ Commit Message Convention

| Change Type / Purpose | Commit Type | When to Use |
| :--- | :--- | :--- |
| New functionality / feature | `feat` | Adding a new feature, API, endpoint, or capability |
| Bug fix | `fix` | Fixing incorrect behavior, crashes, or logic errors |
| Cleanup / logs / config changes | `chore` | Non-functional changes (configs, logs, deps, tooling) |
| Restructure code | `refactor` | Code changes that improve structure without changing behavior |
| Docs only | `docs` | README, comments, API docs, markdown files |
| Formatting / linting | `style` | Code style changes (spacing, formatting, lint fixes) |
| Tests | `test` | Adding or updating unit/integration tests |
| Faster / optimized | `perf` | Performance improvements |
| Build system changes | `build` | Docker, dependencies, build scripts |
| CI pipeline changes | `ci` | GitHub Actions, GitLab CI, Jenkins, etc. |
| Updates (general) | `update` ⚠️ | Version bumps or content updates (use sparingly) |
| Dependency updates | `deps` | Updating libraries or packages |
| Reverts a commit | `revert` | Reverting a previous commit |
| Security fixes | `security` | Vulnerability fixes, auth/security improvements |
| Configuration changes | `config` | App/runtime config changes (if you separate from chore) |
| Hotfix (urgent prod fix) | `hotfix` | Critical production fixes |
| Release-related changes | `release` | Versioning, changelog, tagging |
| Initial commit / scaffolding | `init` | Project bootstrap or first commit |
| Experimental / spikes | `experiment` | PoCs, spikes, exploratory code |

> [!TIP]
> **Why use conventions?** Standardized commit prefixes make the project history readable, searchable, and compatible with automated tools like changelog generators and semantic versioning.


---


## 📧 Rewriting Author/Committer Email in Git History

Use `git filter-repo` to replace commits authored or committed by an old email with a new one. It's faster and safer than the older `git filter-branch`.

### How it works

1. `git filter-repo` walks through every commit in your history
2. For each commit, it checks if the author/committer email matches `old_email`
3. If it matches, it rewrites that commit with the corrected `new_email`
4. All commits get new hashes (since hash changes when content changes), but dates and messages stay the same

### Step 1 — Check existing authors

```bash
# View authors in recent history
git log --format="%an <%ae>"

# View all unique authors across all branches
git log --all --format="%an <%ae>" | sort -u
```

### Step 2 — Rewrite history with mailmap

```bash
# <<'EOF' ... EOF is a bash heredoc — passes multi-line text directly into the command via stdin
git filter-repo --force --mailmap /dev/stdin <<'EOF'
new_name <new_email> old_name <old_email>
EOF
```

> [!NOTE]
> `git filter-repo` removes the `origin` remote by default as a safety measure to prevent accidentally force-pushing rewritten history to the wrong remote.

### Step 3 — Re-add remote and verify

```bash
# Check remote is gone
git remote -v

# Re-add it
git remote add origin <repository-url>

# Optionally review what refs changed
cat .git/filter-repo/changed-refs
```

### Step 4 — Force push rewritten history

Since `git filter-repo` removes the `origin` remote and you re-added it manually, Git doesn't know which remote branch to track yet. Use `--set-upstream` (or `-u`) to link your local branch to the remote on the first push.

```bash
# Push all branches with upstream tracking set
git push --force --set-upstream origin --all
git push --force --tags

# Or push a specific branch and set upstream at the same time
git push --force -u origin main
```

> [!WARNING]
> Force-pushing rewrites shared history. Coordinate with your team before doing this on shared branches.

---

### 🔧 About `git filter-repo`

A tool to rewrite Git history. It's faster and safer than the older `git filter-branch`. It walks through every commit and lets you transform things like file content, commit messages, or author info.

| Concept | Explanation |
| --- | --- |
| `git filter-repo` | Rewrites Git history by walking every commit and applying transformations |
| `--mailmap` | Tells filter-repo to use a mailmap file for remapping author/committer identities |
| `--force` | Required if the repo wasn't freshly cloned (overrides the safety check) |
| `<<'EOF' ... EOF` | Bash heredoc — passes multi-line text as stdin without creating a temp file |

---

## 🕐 Committing with a Custom Date (Backdating)

Git records two timestamps per commit: the **author date** (when the change was made) and the **committer date** (when the commit was created). Both can be overridden using environment variables before running `git commit`.

### Why use this

- Preserve the original date when migrating or replaying commits
- Match commit timestamps to when work was actually done
- Reconstruct history from notes/logs after the fact

### Syntax

```bash
git add <file> && \
GIT_AUTHOR_DATE="<YYYY-MM-DDTHH:MM:SS>" \
GIT_COMMITTER_DATE="<YYYY-MM-DDTHH:MM:SS>" \
git commit -m "<type>: <message>"
```

### How backdating works

| Variable | Controls |
| --- | --- |
| `GIT_AUTHOR_DATE` | The date shown in `git log` as "Author Date" — when the change was made |
| `GIT_COMMITTER_DATE` | The date shown as "Commit Date" — when the commit object was created |

- Both are set as **shell environment variables** prefixed before `git commit`, so they apply only to that single command
- The `&&` chains `git add` first; the `\` at the end of each line continues the command across lines
- Format: ISO 8601 — `YYYY-MM-DDTHH:MM:SS` or with timezone like `2026-06-18T22:05:13+05:30`

### Verify the date was applied

```bash
git log --format="%H %ai %ci %s" -1
#                     ^^^ ^^^
#                     |   committer date
#                     author date
```

> [!NOTE]
> These variables only affect the current `git commit` call. They do not persist to future commits.

---

## ✅ Rule of Thumb

> Merge shared branches,  
> Rebase local branches,  
> Cherry-pick only when you need a specific commit.
