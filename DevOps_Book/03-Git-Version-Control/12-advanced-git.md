# Chapter 12 — Advanced Git

## Learning Objectives

By the end of this chapter you will be able to:

- Write and deploy client-side and server-side Git hooks to automate quality gates
- Share hooks across a team using a repository-tracked directory and Husky
- Use `git bisect` (manually and automated) to pinpoint the commit that introduced a bug
- Manage multiple concurrent branches with `git worktree` without stashing
- Integrate external repositories as submodules and keep them updated
- Use `git blame` to trace the origin of any line of code
- Use `git grep` to efficiently search the working tree and historical commits

**Prerequisites:** Chapter 11 — GitHub & GitLab Workflows

---

## Git Hooks

### What Are Hooks?

Git hooks are scripts that Git executes automatically at specific points in its lifecycle. They live in `.git/hooks/` and can be written in any language — bash, Python, Node.js, Ruby — as long as the file is executable.

Hooks come in two families:

- **Client-side** — run on the developer's machine during local operations (commit, push, checkout)
- **Server-side** — run on the remote repository server during receive operations (used to enforce policies on the server)

Git ships sample hooks in `.git/hooks/` with a `.sample` extension. Remove the extension and make the file executable to activate a hook.

### Client-Side Hooks

| Hook | Triggered | Common Uses |
|---|---|---|
| `pre-commit` | Before the commit is saved | Run linter, check for secrets, run fast unit tests |
| `prepare-commit-msg` | After default message is created, before editor opens | Prepend branch name/ticket to message |
| `commit-msg` | After user writes message, before commit is finalised | Validate commit message format |
| `post-commit` | After commit is saved | Notifications, IDE integrations |
| `pre-push` | Before `git push` sends data | Run full test suite, block push on failure |
| `post-checkout` | After `git checkout` | Auto-install dependencies if package.json changed |
| `pre-rebase` | Before a rebase starts | Warn user, prevent rebasing shared branches |

### Making a Hook Executable

```bash
chmod +x .git/hooks/pre-commit
```

Hooks that are not executable are silently ignored — a very common source of confusion.

### Example: pre-commit — Catch Debug Statements and Run a Linter

```bash
#!/usr/bin/env bash
# .git/hooks/pre-commit

set -e   # exit immediately on any error

echo ">>> pre-commit: checking for debug statements..."

# Block commits containing leftover debug output
FORBIDDEN_PATTERNS="console\.log\|pdb\.set_trace\|binding\.pry\|debugger"
if git diff --cached | grep -qE "^\+.*($FORBIDDEN_PATTERNS)"; then
  echo "ERROR: Commit contains debug statements. Remove them before committing."
  exit 1
fi

echo ">>> pre-commit: running linter..."
if command -v eslint &> /dev/null; then
  git diff --cached --name-only --diff-filter=ACM | \
    grep '\.js$\|\.ts$' | \
    xargs -r eslint --max-warnings=0
fi

echo ">>> pre-commit: running shellcheck on staged shell scripts..."
git diff --cached --name-only --diff-filter=ACM | \
  grep '\.sh$' | \
  xargs -r shellcheck

echo ">>> pre-commit: all checks passed."
```

### Example: commit-msg — Enforce Conventional Commits

```bash
#!/usr/bin/env bash
# .git/hooks/commit-msg
# Conventional Commits format: type(scope): description
# Types: feat, fix, docs, style, refactor, test, chore, perf, ci, build, revert

COMMIT_MSG_FILE="$1"
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Allow merge commits and revert commits to bypass the check
if echo "$COMMIT_MSG" | grep -qE "^(Merge|Revert)"; then
  exit 0
fi

PATTERN="^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\(.+\))?: .{1,72}$"

if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
  echo ""
  echo "ERROR: Commit message does not follow Conventional Commits format."
  echo ""
  echo "Expected format: type(scope): description"
  echo ""
  echo "Valid types: feat, fix, docs, style, refactor, test, chore, perf, ci, build, revert"
  echo ""
  echo "Examples:"
  echo "  feat(auth): add OAuth login support"
  echo "  fix(payment): prevent null pointer on empty cart"
  echo "  chore: upgrade Node.js to v20"
  echo ""
  echo "Your message was: $COMMIT_MSG"
  exit 1
fi
```

### Example: pre-push — Block Push if Tests Fail

```bash
#!/usr/bin/env bash
# .git/hooks/pre-push

set -e

echo ">>> pre-push: running test suite before pushing..."
npm test

echo ">>> pre-push: tests passed, proceeding with push."
```

### Sharing Hooks with the Team

The `.git/` directory is not tracked by Git, so hooks are not automatically shared when teammates clone the repository. Two solutions:

**Option 1: Tracked `hooks/` directory with a setup script**

```bash
# Project structure:
# hooks/
#   pre-commit
#   commit-msg
#   pre-push
# scripts/
#   install-hooks.sh

# scripts/install-hooks.sh
#!/usr/bin/env bash
HOOKS_DIR="$(git rev-parse --show-toplevel)/hooks"
GIT_HOOKS_DIR="$(git rev-parse --show-toplevel)/.git/hooks"

for hook in "$HOOKS_DIR"/*; do
  hook_name=$(basename "$hook")
  ln -sf "$HOOKS_DIR/$hook_name" "$GIT_HOOKS_DIR/$hook_name"
  chmod +x "$GIT_HOOKS_DIR/$hook_name"
  echo "Installed hook: $hook_name"
done
echo "All hooks installed."
```

Add `npm run prepare` or `make install-hooks` to onboarding instructions.

**Option 2: Husky (Node.js projects)**

Husky manages hooks automatically via `package.json`. After `npm install`, hooks are installed into `.husky/` and Git is configured to use that directory.

```bash
# Install
npm install --save-dev husky
npx husky init

# Add a pre-commit hook
echo "npx lint-staged" > .husky/pre-commit

# Add a commit-msg hook (with commitlint)
npm install --save-dev @commitlint/cli @commitlint/config-conventional
echo "npx --no -- commitlint --edit \$1" > .husky/commit-msg
```

```json
// package.json
{
  "scripts": {
    "prepare": "husky"
  },
  "lint-staged": {
    "*.{js,ts}": ["eslint --fix", "prettier --write"],
    "*.py": ["black", "flake8"]
  }
}
```

### Bypassing Hooks

In a genuine emergency, you can skip client-side hooks:

```bash
git commit --no-verify -m "emergency: hotfix for production outage"
git push --no-verify
```

Document when this is acceptable in your team's contributing guide. It should be rare and always followed by a follow-up commit that fixes whatever the hook would have caught.

---

## Git Bisect

### The Problem

A bug exists in the current version of the codebase. You know it did not exist in the version tagged `v1.0.0`, but 200 commits have been made since then. Reviewing all 200 commits manually is impractical.

`git bisect` performs a **binary search** through commit history. At each step it checks out a midpoint commit, you test whether the bug exists, and you tell Git `good` or `bad`. After `log₂(200) ≈ 8` steps, the first bad commit is identified.

### Manual Bisect Walkthrough

```bash
# 1. Start a bisect session
git bisect start

# 2. Tell Git the current state is broken
git bisect bad

# 3. Tell Git the last known good state (tag, branch name, or hash)
git bisect good v1.0.0

# Git checks out the midpoint commit and says:
# "Bisecting: 100 revisions left to test after this (roughly 7 steps)"

# 4. Test: does the bug exist at this commit?
#    If yes:
git bisect bad
#    If no:
git bisect good

# Git narrows the range and checks out the next midpoint.
# Repeat until Git prints:
# "<hash> is the first bad commit"

# 5. Inspect the offending commit
git show <hash>

# 6. End the bisect session (returns you to original branch)
git bisect reset
```

### Automated Bisect with a Test Script

When the bug is programmatically detectable, bisect can run fully automatically. Write a script that exits `0` for a good state and `1` (or any non-zero) for a bad state.

```bash
#!/usr/bin/env bash
# test-for-regression.sh — returns 0 (good) or 1 (bad)
npm test -- --testNamePattern="payment total calculation" --silent
# npm test exits non-zero if the test fails
```

```bash
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
git bisect run ./test-for-regression.sh
# Git will run the script at each midpoint automatically
```

### Real Example: Finding a Performance Regression

```bash
# Setup: a benchmark shows response time regressed from 50ms to 800ms
# somewhere in the last 200 commits

git bisect start
git bisect bad HEAD
git bisect good v3.1.0

# Automated test script:
cat > /tmp/perf-check.sh << 'EOF'
#!/usr/bin/env bash
npm run build --silent 2>/dev/null || exit 125  # 125 = skip this commit
RESPONSE_TIME=$(curl -s -o /dev/null -w "%{time_total}" http://localhost:3000/api/users)
# Fail (bad) if response time exceeds 200ms
python3 -c "exit(0 if float('$RESPONSE_TIME') < 0.2 else 1)"
EOF
chmod +x /tmp/perf-check.sh

git bisect run /tmp/perf-check.sh
```

Exit code `125` tells `git bisect` to skip a commit (e.g., because it fails to build) without marking it good or bad.

---

## Git Worktrees

### The Problem

You are in the middle of a feature branch with uncommitted work. A production bug arrives. You either have to stash your work, switch branches, fix the bug, push, then switch back and pop the stash — or you open a second terminal, clone the entire repo again (wasting disk space), and work there.

### The Solution: Worktrees

A **worktree** is an additional checked-out working directory linked to the same repository. Multiple worktrees share the same `.git` directory (objects, refs), so no extra disk space is used for the repository data itself. Each worktree is on a different branch.

### Commands

```bash
# Create a new worktree for a hotfix branch
# (the new directory is created at ../hotfix-branch)
git worktree add ../hotfix-branch hotfix/v1.2

# You can also create the branch at the same time
git worktree add -b hotfix/v1.2 ../hotfix-branch main

# List all worktrees
git worktree list
# /home/alice/myproject           abc1234 [feature/dark-mode]
# /home/alice/hotfix-branch       def5678 [hotfix/v1.2]

# Work in the hotfix worktree (separate terminal)
cd ../hotfix-branch
# ... fix the bug, commit, push ...

# Remove the worktree when done
git worktree remove ../hotfix-branch

# Prune stale worktree metadata (if you deleted the directory manually)
git worktree prune
```

### Use Cases

| Use Case | How Worktree Helps |
|---|---|
| Reviewing a PR while working on a feature | Check out PR branch in a separate worktree |
| Running a long test suite on main while developing | Keep main and feature in separate worktrees |
| Responding to a production hotfix mid-feature | Hotfix in a new worktree; no stashing needed |
| Comparing behaviour between two branches | Run both simultaneously on different ports |

---

## Git Submodules

### What Are Submodules?

A submodule is a **Git repository nested inside another Git repository**, pinned to a specific commit. The parent repository stores a reference (the commit hash) to the submodule, not a copy of its files.

Common use cases:
- Shared internal libraries across multiple services
- Vendored third-party code you need to modify
- Separating a plugin from its host application

### Adding a Submodule

```bash
git submodule add https://github.com/example/shared-lib.git vendor/shared-lib
# Creates: vendor/shared-lib/  (the submodule checkout)
#          .gitmodules          (configuration file)

# .gitmodules content:
# [submodule "vendor/shared-lib"]
#     path = vendor/shared-lib
#     url = https://github.com/example/shared-lib.git

git commit -m "chore: add shared-lib as submodule"
```

### Cloning a Repo That Contains Submodules

```bash
# Option 1: clone and initialise in one step (recommended)
git clone --recurse-submodules https://github.com/your-org/main-repo.git

# Option 2: clone first, then initialise submodules
git clone https://github.com/your-org/main-repo.git
cd main-repo
git submodule init
git submodule update --init --recursive   # --recursive handles nested submodules
```

### Updating a Submodule

A submodule is pinned to a specific commit. To move it to a newer commit:

```bash
cd vendor/shared-lib
git pull origin main       # fetch latest commits in the submodule repo

cd ../..                   # back to parent repo
git add vendor/shared-lib  # stage the new commit pointer
git commit -m "chore: update shared-lib to latest"
git push
```

Teammates must run `git submodule update --init --recursive` after pulling the parent repo to get the updated submodule.

### Removing a Submodule

```bash
# 1. Remove the submodule entry from .gitmodules
git config -f .gitmodules --remove-section submodule.vendor/shared-lib

# 2. Remove the submodule from .git/config
git config --remove-section submodule.vendor/shared-lib

# 3. Stage the .gitmodules change
git add .gitmodules

# 4. Remove the submodule from the index (not the working tree yet)
git rm --cached vendor/shared-lib

# 5. Delete the directory
rm -rf vendor/shared-lib

# 6. Clean up .git internals
rm -rf .git/modules/vendor/shared-lib

git commit -m "chore: remove shared-lib submodule"
```

### Dangers and Alternatives

**Common pitfalls:**
- **Forgetting to push submodule changes** before pushing the parent — teammates get a broken build because the parent points to a commit that does not exist on the remote
- **`git pull` does not automatically update submodules** — always run `git submodule update` after pulling
- **Detached HEAD in submodules** — submodules check out a specific commit, not a branch, so you must explicitly work on a branch when making changes inside a submodule

**Alternatives:**

| Alternative | How It Works | Best When |
|---|---|---|
| `git subtree` | Merges subproject history inline into the parent repo | You want simpler workflows without submodule complexity |
| Package manager (`npm`, `pip`, `Maven`) | Declares versioned dependencies externally | Dependencies are versioned and published |
| Monorepo tooling (Nx, Turborepo, Bazel) | All code in one repo, tooling handles build graphs | You control all the code |

---

## git blame

`git blame` shows **who last modified each line** of a file, in which commit, and when.

### Basic Usage

```bash
git blame src/auth/LoginService.java
```

Output format:
```
^abc1234 (Alice Smith  2024-03-10 14:22:01 +0000  42) public User login(String email, String password) {
 def5678 (Bob Jones    2024-05-01 09:15:33 +0000  43)     User user = userRepo.findByEmail(email);
 def5678 (Bob Jones    2024-05-01 09:15:33 +0000  44)     if (user == null) throw new NotFoundException();
```

### Useful Flags

```bash
# Blame a specific line range (lines 10–20)
git blame -L 10,20 src/auth/LoginService.java

# Ignore whitespace-only changes (formatting commits)
git blame -w src/auth/LoginService.java

# Detect lines moved or copied within the same file
git blame -M src/auth/LoginService.java

# Detect lines moved or copied from other files in the same commit
git blame -C src/auth/LoginService.java

# Show the commit in short format for readability
git blame --abbrev=7 src/auth/LoginService.java
```

### Investigative Workflow

```bash
# 1. Find the suspicious line
git blame -w src/payment/Calculator.java | grep "tax rate"
# Output: a1b2c3d4 (Carol Chen 2024-06-15 ...) return amount * 0.15;  // tax rate

# 2. Inspect the full commit to understand context
git show a1b2c3d4

# 3. If still unclear, blame the file at that commit to see surrounding lines
git blame a1b2c3d4 -- src/payment/Calculator.java
```

`git blame` answers "who" and "when"; `git show` answers "why" (via the commit message and diff).

---

## git grep

`git grep` searches the **working tree** (or any commit) for a pattern. It honours `.gitignore`, is parallelised internally, and is consistently faster than running `grep -r` on a large repository.

### Basic Usage

```bash
# Search the working tree
git grep "getUserById"

# Show line numbers
git grep -n "getUserById"

# Case-insensitive search
git grep -i "todo"

# Restrict to specific file types
git grep -n "TODO" -- "*.py"

# Search only within a subdirectory
git grep "paymentGateway" -- src/payment/
```

### Searching Historical Commits

```bash
# Search in a specific commit
git grep "deprecated_function" v1.0.0

# Search across all commits on main (can be slow on large repos)
git grep "deprecated_function" $(git rev-list main)

# Find which commit introduced a string (combine with rev-list)
git log -S "deprecated_function" --oneline
# (git log -S is "pickaxe search" — finds commits that added or removed the string)
```

### Practical Examples

```bash
# Find all TODO comments with their file and line number
git grep -n "TODO\|FIXME\|HACK" -- "*.py" "*.js"

# Find all uses of a function across the codebase
git grep -n "calculateTax(" -- "*.java"

# Find hardcoded IP addresses (potential security issue)
git grep -nE "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}"

# Count occurrences per file
git grep -c "console.log" -- "*.js"
```

---

## Summary

- **Git hooks** are scripts that automate quality gates at specific points in the Git lifecycle; share them via a tracked `hooks/` directory or Husky
- **`git bisect`** performs binary search across commit history to find the commit that introduced a bug; `git bisect run` automates it with a test script
- **`git worktree`** provides multiple simultaneous working directories on different branches without stashing or re-cloning
- **Git submodules** embed one repository inside another at a pinned commit; they are powerful but operationally complex — consider package managers or `git subtree` as alternatives
- **`git blame`** traces the origin of any line; combine with `git show` to understand the why behind a change
- **`git grep`** searches the working tree (or any commit) efficiently, faster than system `grep` on large repositories

---

## Knowledge Check

1. What exit code does a hook script need to return to abort the Git operation?
2. In `git bisect`, what does exit code `125` tell Git when used with `git bisect run`?
3. You have two worktrees open. Can they both be on the same branch? Why or why not?
4. A colleague clones your repo and the `vendor/lib` directory is empty. What command do they need to run?
5. How does `git blame -w` differ from `git blame` with no flags?
6. What is the difference between `git grep` and `git log -S`?

---

## Hands-On Exercises

### Exercise A: Write a pre-commit Hook for Conventional Commits

```bash
# Create a test repo
mkdir hooks-demo && cd hooks-demo
git init

# Write the commit-msg hook from the chapter
cat > .git/hooks/commit-msg << 'EOF'
#!/usr/bin/env bash
COMMIT_MSG=$(cat "$1")
PATTERN="^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\(.+\))?: .{1,72}$"
if echo "$COMMIT_MSG" | grep -qE "^(Merge|Revert)"; then exit 0; fi
if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
  echo "ERROR: Commit message must follow Conventional Commits."
  echo "Example: feat(auth): add OAuth login"
  exit 1
fi
EOF
chmod +x .git/hooks/commit-msg

# Test: this should fail
touch file.txt && git add file.txt
git commit -m "added a file"       # should be rejected

# Test: this should pass
git commit -m "chore: add placeholder file"
```

### Exercise B: Use git bisect on a Sample Repository

```bash
# Set up a repo with a known regression
mkdir bisect-demo && cd bisect-demo
git init

for i in $(seq 1 20); do
  echo "iteration $i" > output.txt
  git add output.txt
  if [ "$i" -eq 12 ]; then
    # Introduce a "bug" at commit 12
    echo "BUG" >> output.txt
    git commit -m "commit $i (contains bug)"
  else
    git commit -m "commit $i"
  fi
done

# Run bisect to find commit 12
git bisect start
git bisect bad HEAD
git bisect good HEAD~15   # commit 5 was good

# Automated: test script checks for "BUG" in output.txt
cat > /tmp/test.sh << 'EOF'
#!/usr/bin/env bash
grep -q "BUG" output.txt && exit 1 || exit 0
EOF
chmod +x /tmp/test.sh

git bisect run /tmp/test.sh
git bisect reset

# Verify: the found commit should be commit 12
```

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./11-github-gitlab-workflows.md">← Previous: GitHub & GitLab Workflows</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./13-best-practices.md">Next: Best Practices →</a>
</div>
