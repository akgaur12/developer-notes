# Chapter 15 — Hands-On Projects

## Learning Objectives

By the end of this chapter, you will have:

- Set up a professional Git repository from scratch for a real project
- Simulated a complete GitFlow lifecycle including conflicts and hotfixes
- Built a working local CI pipeline using Git hooks
- Completed an end-to-end open source contribution workflow with clean history

**Prerequisites:** Chapter 14 — Common Git Mistakes & Fixes

---

## Overview

This chapter contains four progressively harder projects. Each one builds on the skills from earlier chapters and is designed to produce something real and usable, not just an exercise you discard.

| Project | Level | Skills |
|---------|-------|--------|
| 1 — Personal Repository Setup | Beginner | init, gitignore, commits, branching, push |
| 2 — GitFlow Feature Simulation | Intermediate | GitFlow, conflicts, releases, hotfixes |
| 3 — Git Hooks CI Pipeline | Advanced | bash scripting, hooks, test automation |
| 4 — Open Source Contribution | Capstone | fork, upstream sync, clean history, PR |

Work through them in order — later projects assume the habits built in earlier ones.

---

## Project 1 — Personal Project Repository Setup

### Goal

Set up a professional Git repository from scratch for any personal project. By the end, you will have a repository that looks like it was set up by someone who knows what they are doing.

### Skills Practiced

`git init`, `.gitignore`, meaningful commits, basic branching, tagging, pushing to GitHub

### Step-by-Step

**Step 1: Initialize the repository**

```bash
mkdir my-project && cd my-project
git init
git config user.name "Your Name"
git config user.email "your@email.com"
```

**Step 2: Create a proper .gitignore**

Go to [gitignore.io](https://www.toptal.com/developers/gitignore) and generate a template for your language. For a Python project:

```bash
curl -sL "https://www.toptal.com/developers/gitignore/api/python,vscode,macos" > .gitignore
```

Verify it covers the critical patterns:

```bash
grep -E "\.env|__pycache__|\.venv|dist/" .gitignore
```

**Step 3: Create a README.md**

```markdown
# My Project

Brief description of what this project does and why it exists.

## Requirements

- Python 3.11+

## Installation

\`\`\`bash
pip install -r requirements.txt
\`\`\`

## Usage

\`\`\`bash
python main.py
\`\`\`
```

**Step 4: Make the initial commit**

```bash
git add .gitignore README.md
git commit -m "chore: initial project setup with gitignore and README"
```

**Step 5: Create a feature branch and work on it**

```bash
# Create and switch to a feature branch
git switch -c feature/add-core-logic

# Do some work
echo "def main(): pass" > main.py
git add main.py
git commit -m "feat: add main entry point"

echo "# Core module" > core.py
git add core.py
git commit -m "feat(core): scaffold core module"

# Merge back with --no-ff to preserve branch history
git switch main
git merge --no-ff feature/add-core-logic -m "feat: merge core logic feature"

# Clean up the branch
git branch -d feature/add-core-logic
```

**Step 6: Create a GitHub repository and push**

```bash
# Create repo on GitHub (via web UI or gh CLI)
gh repo create my-project --public --source=. --remote=origin

# Or if repo already exists on GitHub:
git remote add origin git@github.com:yourusername/my-project.git

# Push with upstream tracking
git push -u origin main
```

**Step 7: Tag and push a release**

```bash
# Create an annotated tag
git tag -a v0.1.0 -m "Release v0.1.0 — initial working version"

# Push the tag
git push origin v0.1.0

# Verify the tag is on GitHub
gh release list
```

### Verification Checklist

- [ ] Repository has a meaningful `.gitignore` covering language-specific artifacts
- [ ] Initial commit message follows the Conventional Commits format
- [ ] Feature branch was merged with `--no-ff` (visible in `git log --graph`)
- [ ] No `node_modules/`, `__pycache__/`, `.env`, or build artifacts in the repository
- [ ] Remote is set to SSH (not HTTPS) — verify with `git remote -v`
- [ ] Tag `v0.1.0` is visible on GitHub under Releases/Tags
- [ ] `git log --oneline` shows a clean, readable history

---

## Project 2 — GitFlow Feature Development Simulation

### Goal

Practice the full GitFlow lifecycle with a realistic scenario: two parallel features, an intentional conflict, a release, and a hotfix.

### Skills Practiced

GitFlow branching model, resolving merge conflicts, release branching, hotfix workflow

### Scenario: "DemoApp v1.0"

You are working on a small web application. Two features are being developed in parallel:
- **Feature A:** User authentication
- **Feature B:** API endpoint for user data

There is an intentional conflict in `app.py`. You will also simulate a production bug requiring a hotfix after release.

### Step-by-Step

**Step 1: Initialize the project with GitFlow branches**

```bash
mkdir demoapp && cd demoapp
git init
git commit --allow-empty -m "chore: initial commit"

# GitFlow uses main (production) and develop (integration)
git switch -c develop
echo "APP_VERSION = '0.0.0'" > app.py
echo "ROUTES = []" >> app.py
git add app.py
git commit -m "chore: initialize app.py on develop"

git push -u origin main
git push -u origin develop
```

**Step 2: Feature A — User authentication**

```bash
# Create feature branch from develop
git switch -c feature/add-auth develop

# Simulate development work
cat > app.py << 'EOF'
APP_VERSION = '0.0.0'
ROUTES = ['/login', '/logout']

def authenticate(username, password):
    """Authenticate a user against the database."""
    return username == 'admin' and password == 'secret'
EOF

git add app.py
git commit -m "feat(auth): add user authentication function"

echo "AUTH_SECRET_KEY = 'change-me-in-production'" > config.py
git add config.py
git commit -m "feat(auth): add auth configuration"
```

**Step 3: Feature B — API endpoint (started at the same time)**

```bash
# Create feature branch from develop (at the same point in history as Feature A)
git switch -c feature/add-api develop

cat > app.py << 'EOF'
APP_VERSION = '0.0.0'
ROUTES = ['/api/v1/users', '/api/v1/health']

def get_users():
    """Return all users from the database."""
    return [{'id': 1, 'name': 'Alice'}, {'id': 2, 'name': 'Bob'}]
EOF

git add app.py
git commit -m "feat(api): add /api/v1/users endpoint"

echo "API_RATE_LIMIT = 100" > api_config.py
git add api_config.py
git commit -m "feat(api): add API rate limit configuration"
```

**Step 4: Merge Feature A into develop**

```bash
git switch develop
git merge --no-ff feature/add-auth -m "feat: merge user authentication feature"
```

**Step 5: Merge Feature B into develop — resolve the conflict**

```bash
git merge --no-ff feature/add-api
# This will fail with a conflict in app.py
```

You will see:

```
Auto-merging app.py
CONFLICT (content): Merge conflict in app.py
```

Open `app.py` and resolve the conflict to include both features:

```python
APP_VERSION = '0.0.0'
ROUTES = ['/login', '/logout', '/api/v1/users', '/api/v1/health']

def authenticate(username, password):
    """Authenticate a user against the database."""
    return username == 'admin' and password == 'secret'

def get_users():
    """Return all users from the database."""
    return [{'id': 1, 'name': 'Alice'}, {'id': 2, 'name': 'Bob'}]
```

```bash
git add app.py
git merge --continue -m "feat: merge API endpoint feature (resolve routes conflict)"
```

**Step 6: Create a release branch**

```bash
# Cut release from develop
git switch -c release/1.0.0 develop

# Bump version
sed -i "s/APP_VERSION = '0.0.0'/APP_VERSION = '1.0.0'/" app.py
git add app.py
git commit -m "chore(release): bump version to 1.0.0"

# Any final release polish (update changelog, etc.)
echo "## v1.0.0" > CHANGELOG.md
echo "- feat: user authentication" >> CHANGELOG.md
echo "- feat: REST API endpoints" >> CHANGELOG.md
git add CHANGELOG.md
git commit -m "docs: add v1.0.0 changelog"
```

**Step 7: Merge release into main and tag it**

```bash
# Merge to main
git switch main
git merge --no-ff release/1.0.0 -m "chore(release): merge release/1.0.0 into main"
git tag -a v1.0.0 -m "Release v1.0.0"

# Merge back to develop to capture release commits
git switch develop
git merge --no-ff release/1.0.0 -m "chore(release): merge release/1.0.0 back to develop"

# Clean up
git branch -d release/1.0.0
```

**Step 8: Simulate a critical hotfix**

A production bug is reported: the authentication check has a security bypass.

```bash
# Hotfix branches always come from main (production)
git switch -c hotfix/fix-auth-bypass main

# Fix the bug
sed -i "s/return username == 'admin' and password == 'secret'/return username != '' and password != '' and len(password) >= 8/" app.py
git add app.py
git commit -m "fix(auth): prevent authentication bypass with empty credentials"

# Bump patch version
sed -i "s/APP_VERSION = '1.0.0'/APP_VERSION = '1.0.1'/" app.py
git add app.py
git commit -m "chore(release): bump version to 1.0.1"

# Merge to main
git switch main
git merge --no-ff hotfix/fix-auth-bypass -m "fix: merge auth bypass hotfix into main"
git tag -a v1.0.1 -m "Hotfix v1.0.1 — auth bypass fix"

# Merge to develop to keep the fix
git switch develop
git merge --no-ff hotfix/fix-auth-bypass -m "fix: merge auth bypass hotfix into develop"

# Clean up
git branch -d hotfix/fix-auth-bypass
```

**Step 9: View the full history**

```bash
git log --oneline --graph --all
```

You should see a clear branching structure: main and develop diverging, feature branches appearing and merging back, the release branch, the v1.0.0 tag, and the hotfix.

---

## Project 3 — Git Hooks CI Pipeline

### Goal

Build a local CI pipeline that runs automatically on every commit and push, catching problems before they reach the remote.

### Skills Practiced

Bash scripting, Git hooks, Conventional Commits enforcement, automated testing

### Overview

You will implement three hooks:

| Hook | Trigger | Checks |
|------|---------|--------|
| `pre-commit` | Before commit is created | Syntax errors, debug statements |
| `commit-msg` | After commit message entered | Conventional Commits format |
| `pre-push` | Before push to remote | Full test suite |

### Hook 1 — pre-commit

This hook runs before Git creates the commit. It checks Python syntax and blocks debug print statements.

Create `.git/hooks/pre-commit`:

```bash
#!/bin/bash
# pre-commit hook: syntax check and debug statement detection

set -e

echo "Running pre-commit checks..."

# ─── 1. Python syntax check ───────────────────────────────────────────────────
PYTHON_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep '\.py$')

if [ -n "$PYTHON_FILES" ]; then
  echo "Checking Python syntax..."
  for file in $PYTHON_FILES; do
    if ! python3 -m py_compile "$file" 2>&1; then
      echo "SYNTAX ERROR: $file has a syntax error. Fix it before committing."
      exit 1
    fi
  done
  echo "  Python syntax: OK"
fi

# ─── 2. Debug statement detection ─────────────────────────────────────────────
if [ -n "$PYTHON_FILES" ]; then
  echo "Checking for debug statements..."
  DEBUG_FOUND=0
  for file in $PYTHON_FILES; do
    # Check for bare print() calls (not in test files)
    if [[ "$file" != *test* ]] && grep -n "^\s*print(" "$file"; then
      echo "  WARNING: Debug print() found in $file"
      DEBUG_FOUND=1
    fi
    # Check for pdb / breakpoint calls
    if grep -n "import pdb\|pdb\.set_trace\|breakpoint()" "$file"; then
      echo "  ERROR: Debugger call found in $file"
      exit 1
    fi
  done
  if [ $DEBUG_FOUND -eq 1 ]; then
    echo "  Hint: Remove debug print() statements or add a '# noqa' comment to suppress."
    exit 1
  fi
  echo "  Debug check: OK"
fi

# ─── 3. TODO/FIXME without ticket number ──────────────────────────────────────
ALL_STAGED=$(git diff --cached --name-only --diff-filter=ACM)
if [ -n "$ALL_STAGED" ]; then
  # Allow TODO(#123) but block bare TODO or FIXME
  BARE_TODO=$(git diff --cached | grep '^\+' | grep -E '\bTODO\b(?!\s*\(#[0-9]+\))|\bFIXME\b' || true)
  if [ -n "$BARE_TODO" ]; then
    echo "  WARNING: Untracked TODO/FIXME found:"
    echo "$BARE_TODO"
    echo "  Use TODO(#issue-number) format to link to a ticket."
    exit 1
  fi
fi

echo "All pre-commit checks passed."
exit 0
```

```bash
chmod +x .git/hooks/pre-commit
```

### Hook 2 — commit-msg

This hook validates the commit message against the Conventional Commits format.

Create `.git/hooks/commit-msg`:

```bash
#!/bin/bash
# commit-msg hook: enforce Conventional Commits format

COMMIT_MSG_FILE="$1"
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Skip merge commits and revert commits
if echo "$COMMIT_MSG" | grep -qE "^Merge |^Revert "; then
  exit 0
fi

# Skip fixup! and squash! commits (used in interactive rebase)
if echo "$COMMIT_MSG" | grep -qE "^(fixup|squash)!"; then
  exit 0
fi

# Conventional Commits pattern
PATTERN="^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\(.+\))?!?: .{1,72}"

if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
  echo ""
  echo "ERROR: Commit message does not follow Conventional Commits format."
  echo ""
  echo "  Expected format:  type(scope): description"
  echo "  Your message:     $COMMIT_MSG"
  echo ""
  echo "  Valid types: feat, fix, docs, style, refactor, test, chore, perf, ci, build, revert"
  echo ""
  echo "  Examples:"
  echo "    feat(auth): add OAuth2 Google login"
  echo "    fix(api): handle null user response"
  echo "    docs(readme): update installation steps"
  echo "    chore(deps): upgrade axios to 1.6.0"
  echo ""
  exit 1
fi

# Check subject line length (50 char soft limit, 72 hard limit)
SUBJECT=$(echo "$COMMIT_MSG" | head -1)
SUBJECT_LENGTH=${#SUBJECT}

if [ $SUBJECT_LENGTH -gt 72 ]; then
  echo ""
  echo "ERROR: Commit subject line is $SUBJECT_LENGTH characters (max 72)."
  echo "  $SUBJECT"
  echo ""
  exit 1
fi

if [ $SUBJECT_LENGTH -gt 50 ]; then
  echo "WARNING: Commit subject line is $SUBJECT_LENGTH characters (recommended max 50)."
fi

exit 0
```

```bash
chmod +x .git/hooks/commit-msg
```

### Hook 3 — pre-push

This hook runs the full test suite before allowing a push. A failed test suite blocks the push.

Create `.git/hooks/pre-push`:

```bash
#!/bin/bash
# pre-push hook: run test suite before pushing

REMOTE="$1"
URL="$2"

echo "Running pre-push checks against $REMOTE ($URL)..."

# ─── Run the test suite ────────────────────────────────────────────────────────
if [ -f "pytest.ini" ] || [ -f "setup.cfg" ] || [ -f "pyproject.toml" ] || [ -d "tests/" ]; then
  echo "Detected Python project — running pytest..."
  if ! python3 -m pytest --tb=short -q; then
    echo ""
    echo "ERROR: Tests failed. Push blocked."
    echo ""
    echo "  Fix failing tests before pushing, or bypass with:"
    echo "    git push --no-verify"
    echo ""
    echo "  WARNING: --no-verify bypasses ALL hooks. Only use in genuine emergencies."
    exit 1
  fi
  echo "  Tests: PASSED"

elif [ -f "package.json" ]; then
  echo "Detected Node.js project — running npm test..."
  if ! npm test --silent; then
    echo ""
    echo "ERROR: Tests failed. Push blocked."
    exit 1
  fi
  echo "  Tests: PASSED"

else
  echo "  No test suite detected. Skipping test run."
fi

echo "All pre-push checks passed. Proceeding with push."
exit 0
```

```bash
chmod +x .git/hooks/pre-push
```

### Testing the Hooks

```bash
# Test pre-commit: add a syntax error
echo "def broken(: pass" > bad.py
git add bad.py
git commit -m "test"   # should be blocked with syntax error

# Test commit-msg: bad format
touch good.py && git add good.py
git commit -m "updated stuff"   # should be blocked
git commit -m "feat: add good feature"   # should pass

# Test pre-push bypass (emergency only)
git push --no-verify origin main
```

### Sharing Hooks with Your Team

Git hooks in `.git/hooks/` are local and not committed. To share them:

```bash
# Option 1: Store in a committed directory
mkdir -p hooks/
cp .git/hooks/pre-commit hooks/
cp .git/hooks/commit-msg hooks/
cp .git/hooks/pre-push hooks/
git add hooks/
git commit -m "chore: add shared Git hooks"

# Each developer symlinks them
ln -sf ../../hooks/pre-commit .git/hooks/pre-commit
ln -sf ../../hooks/commit-msg .git/hooks/commit-msg
ln -sf ../../hooks/pre-push .git/hooks/pre-push
```

```bash
# Option 2: Husky (Node.js projects)
npm install --save-dev husky
npx husky install
npx husky add .husky/pre-commit "npm run lint"
npx husky add .husky/commit-msg "npx commitlint --edit $1"
```

```bash
# Option 3: Configure core.hooksPath (Git 2.9+)
git config core.hooksPath hooks/
# Everyone who clones sets this once — hooks/ directory is committed
```

---

## Project 4 — Open Source Contribution Workflow

### Goal

Complete an end-to-end open source contribution, from finding a project to submitting a professional pull request with clean history.

### Skills Practiced

Forking, upstream synchronization, interactive rebase, fixup commits, PR writing

### Step-by-Step

**Step 1: Fork a repository**

Find a real project to contribute to, or use a demo repo. Fork it on GitHub:

```bash
# Fork via GitHub web UI, then clone your fork
git clone git@github.com:yourusername/target-repo.git
cd target-repo
```

Or use the `gh` CLI:

```bash
gh repo fork original-owner/repo --clone
cd repo
```

**Step 2: Set up the upstream remote**

```bash
# Add the original repo as 'upstream'
git remote add upstream git@github.com:original-owner/repo.git

# Verify you have both remotes
git remote -v
# origin    git@github.com:yourusername/repo.git (fetch)
# origin    git@github.com:yourusername/repo.git (push)
# upstream  git@github.com:original-owner/repo.git (fetch)
# upstream  git@github.com:original-owner/repo.git (push)
```

**Step 3: Create a feature branch**

```bash
# Always branch from a fresh copy of upstream/main
git fetch upstream
git switch -c feature/add-contribution upstream/main
```

**Step 4: Develop with messy WIP commits (realistic)**

```bash
# Normal development — messy commits are fine at this stage
echo "# New feature" > feature.md
git add feature.md
git commit -m "wip"

# Make changes, realize you need to fix something
echo "Details here" >> feature.md
git add feature.md
git commit -m "wip2"

# Fix a typo
sed -i "s/Detials/Details/" feature.md 2>/dev/null || true
git add feature.md
git commit -m "typo fix"

# Add the actual implementation
cat >> feature.md << 'EOF'

## Implementation

The feature works by following these steps:
1. Validate input
2. Process data
3. Return result
EOF
git add feature.md
git commit -m "add implementation section"

# Remember to add tests
echo "test: feature does what it says" > test_feature.py
git add test_feature.py
git commit -m "tests"
```

At this point you have 5 messy commits. That is fine — you will clean them up before the PR.

**Step 5: Sync with upstream before opening the PR**

Never open a PR without syncing with upstream first. The maintainer should be able to merge it without conflicts.

```bash
git fetch upstream

# Rebase your branch on top of the latest upstream/main
git rebase upstream/main

# If there are conflicts, resolve them:
# git add <resolved-file>
# git rebase --continue
```

**Step 6: Clean up history with interactive rebase**

Squash your 5 WIP commits into 1–2 meaningful commits:

```bash
git rebase -i upstream/main
```

The editor will show your 5 commits. Change the markers:

```
pick abc1234 wip
squash def5678 wip2
squash ghi9012 typo fix
squash jkl3456 add implementation section
reword mno7890 tests
```

After saving, Git opens a commit message editor for each `pick`/`reword`. Write:

```
Commit 1 subject:
docs(feature): add complete feature documentation

Commit 2 subject:
test(feature): add unit tests for feature behavior
```

**Step 7: Use fixup commits for minor corrections**

If you need to correct something in a previous commit (without `rebase -i` each time):

```bash
# Make a fix
echo "## Notes" >> feature.md
git add feature.md

# Create a fixup commit targeting the first commit
git commit --fixup <hash-of-commit-1>

# Automatically squash all fixups during rebase
git rebase -i --autosquash upstream/main
# fixup! commits are automatically reordered and marked 'fixup'
```

**Step 8: Push to your fork**

```bash
git push origin feature/add-contribution

# If you already pushed and then did a rebase, force push is required
git push --force-with-lease origin feature/add-contribution
```

**Step 9: Open the pull request**

Use the `gh` CLI with a professional PR description:

```bash
gh pr create \
  --title "docs(feature): add complete feature documentation and tests" \
  --body "$(cat << 'EOF'
## What

Added documentation for the new feature, including implementation notes and usage examples.

## Why

The feature was added in #123 but without documentation. New contributors and users
had no way to understand how to use it correctly.

## How to Test

1. Clone the repository
2. Navigate to `feature.md` and read through the documentation
3. Run `python3 -m pytest test_feature.py -v` to verify all tests pass
4. Confirm the documentation matches the actual behavior of the feature

## Checklist

- [x] Tests pass locally (`pytest test_feature.py`)
- [x] Documentation follows existing style conventions
- [x] No secrets or sensitive data included
- [x] Branch is up to date with upstream/main
- [x] Commit history is clean and follows Conventional Commits
EOF
)"
```

**Step 10: Handle review feedback**

When a reviewer requests changes, do NOT create a PR update commit like "addressed review comments". Instead, create clean commits and optionally fixup:

```bash
# Make the requested change
echo "Updated based on feedback" >> feature.md
git add feature.md

# Option A: new commit for significant changes
git commit -m "docs(feature): clarify step 2 per review feedback"

# Option B: fixup to the original commit (preferred for minor corrections)
git commit --fixup <original-commit-hash>
git rebase -i --autosquash upstream/main

# Push the updated branch
git push --force-with-lease origin feature/add-contribution
```

### Verification Checklist

- [ ] Fork is on your GitHub account and remote is named `origin`
- [ ] Original repository is added as `upstream` remote
- [ ] Feature branch was created from `upstream/main`, not `origin/main`
- [ ] Branch was rebased on `upstream/main` before opening the PR
- [ ] WIP commits were squashed into 1–2 logical, well-named commits
- [ ] Commit messages follow Conventional Commits format
- [ ] PR description includes What, Why, How to Test, and a checklist
- [ ] All CI checks pass on the PR (green checkmarks)

### Extensions

Once the basic workflow is solid, try these extensions:

**Add signed commits to the PR:**

```bash
git config commit.gpgsign true
# All commits from here are signed — GitHub shows "Verified"
```

**Add a CI status check:**

Create `.github/workflows/ci.yml` in your fork:

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - run: python3 -m pytest
```

**Use a PR template:**

Create `.github/pull_request_template.md` in the repository so all PRs auto-fill with your standard checklist.

---

## Course Wrap-Up

You have now covered the full Git and version control curriculum:

| Chapter | Topic |
|---------|-------|
| 1 | Introduction to Git |
| 2 | Git Internals |
| 3 | Essential Commands |
| 4 | Branching |
| 5 | Remote Repositories |
| 6 | Merging |
| 7 | Rebasing |
| 8 | Stash and Cherry-pick |
| 9 | Undoing Changes |
| 10–12 | (Advanced topics) |
| 13 | Best Practices |
| 14 | Common Mistakes & Fixes |
| 15 | Hands-On Projects (this chapter) |

The next chapter covers interview preparation — common Git questions, how to talk about your workflow, and what to show in a portfolio.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./14-common-mistakes.md">← Previous: Common Mistakes</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./16-interview-preparation.md">Next: Interview Preparation →</a>
</div>
