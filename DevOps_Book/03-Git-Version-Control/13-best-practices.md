# Chapter 13 — Git Best Practices

## Learning Objectives

By the end of this chapter, you will be able to:

- Write clear, meaningful commit messages following established conventions
- Configure and maintain effective `.gitignore` files at both global and project levels
- Use Git LFS to handle large binary files without bloating your repository
- Sign commits with GPG or SSH to prove authorship
- Keep your repository clean and free of sensitive data
- Understand and apply the principle of commit atomicity

**Prerequisites:** Chapter 12 — Advanced Git

---

## Commit Message Best Practices

### The 7 Rules of Great Commit Messages

Chris Beams' widely adopted rules for writing excellent commit messages:

1. **Separate subject from body with a blank line**
2. **Limit subject line to 50 characters**
3. **Capitalize the subject line**
4. **Do not end subject line with a period**
5. **Use imperative mood** — write "Add feature" not "Added feature"
6. **Wrap body at 72 characters**
7. **Use body to explain WHAT and WHY, not HOW**

The imperative mood matches the language Git itself uses: "Merge branch", "Revert commit", "Fix typo". Think of each commit message as completing the sentence: *"If applied, this commit will..."*

**Full example:**

```
Refactor user authentication to use JWT tokens

Previously, session-based authentication was tightly coupled to
a single server instance, preventing horizontal scaling. JWT
tokens allow stateless authentication across multiple nodes.

This change removes the session store dependency and updates
all protected route middleware to validate tokens instead.

Closes #142
```

---

### Conventional Commits Specification

The [Conventional Commits](https://www.conventionalcommits.org/) specification provides a machine-readable format on top of the 7 rules. It powers automated changelogs, semantic versioning, and CI tooling.

**Format:**

```
type(scope): description

[optional body]

[optional footer(s)]
```

**Commit types:**

| Type       | When to use                                      |
|------------|--------------------------------------------------|
| `feat`     | A new feature                                    |
| `fix`      | A bug fix                                        |
| `docs`     | Documentation only                               |
| `style`    | Formatting, whitespace (no logic change)         |
| `refactor` | Code restructure (no feature/fix)                |
| `test`     | Adding or updating tests                         |
| `chore`    | Maintenance tasks, tooling, config               |
| `perf`     | Performance improvement                          |
| `ci`       | CI/CD pipeline changes                           |
| `build`    | Build system or dependency changes               |
| `revert`   | Reverts a previous commit                        |

**Examples:**

```bash
feat(auth): add OAuth2 Google login
fix(api): handle null user response
docs(readme): update installation steps
chore(deps): upgrade axios to 1.6.0
refactor(db): extract connection pool to separate module
test(auth): add unit tests for token validation
```

**Breaking changes** — two ways to signal them:

```bash
# Option 1: exclamation mark shorthand
feat!: change API endpoint from /v1 to /v2

# Option 2: BREAKING CHANGE footer
feat(api): redesign endpoint structure

BREAKING CHANGE: /v1/users is now /v2/users. Update all clients.
```

---

### Good vs Bad Commit Messages

| Bad | Good |
|-----|------|
| `fix bug` | `fix(auth): prevent login bypass on empty password` |
| `WIP` | `feat(cart): add item quantity stepper component` |
| `changes` | `refactor(db): extract query builder to separate class` |
| `updated stuff` | `chore(deps): upgrade jest from 28 to 29` |
| `asdfgh` | `test(api): add integration tests for /users endpoint` |
| `fixed the thing john mentioned` | `fix(payments): handle Stripe webhook duplicate events` |

---

### Commit Atomicity

**One logical change per commit.** An atomic commit:

- Can be understood in isolation
- Can be reverted cleanly without breaking other changes
- Can be cherry-picked to another branch safely
- Makes `git bisect` effective for hunting regressions

**Anti-pattern — a non-atomic commit:**

```
feat: add user profile page, fix login bug, update README, refactor DB layer
```

**Correct approach — four separate commits:**

```
feat(profile): add user profile page
fix(auth): prevent redirect loop on expired session
docs(readme): add environment setup instructions
refactor(db): extract connection logic to db.js
```

---

### Commit Frequency

- Commit at least at the end of every logical unit of work
- Do not let unrelated changes accumulate in a single commit
- Frequent small commits are easier to reason about than rare large ones
- Use `git add -p` (patch mode) to stage partial file changes when needed

---

## .gitignore Best Practices

### Global .gitignore (Developer Machine)

Configure a global ignore file for files that should never be committed on *your* machine regardless of project:

```bash
git config --global core.excludesfile ~/.gitignore_global
```

**Recommended `~/.gitignore_global` contents:**

```gitignore
# macOS
.DS_Store
.AppleDouble
.LSOverride

# Windows
Thumbs.db
ehthumbs.db
Desktop.ini

# IDEs & Editors
.idea/
.vscode/
*.iml
*.suo
*.user
*.sublime-workspace
*.swp
*~
.project
.classpath
.settings/

# OS temp files
*.tmp
*.bak
```

### Per-Repository .gitignore

Each project should have its own `.gitignore` tailored to the language and framework:

**Node.js example:**

```gitignore
node_modules/
dist/
build/
.env
.env.local
.env.*.local
*.log
npm-debug.log*
coverage/
.nyc_output/
```

**Python example:**

```gitignore
__pycache__/
*.py[cod]
*.egg-info/
.venv/
venv/
dist/
build/
.pytest_cache/
.coverage
htmlcov/
*.log
.env
```

### Generating Templates

Use [gitignore.io](https://www.toptal.com/developers/gitignore) to generate accurate templates:

```bash
# via curl
curl -sL https://www.toptal.com/developers/gitignore/api/python,django,vscode > .gitignore
```

### Debugging .gitignore Issues

```bash
# Check why a specific file is (or isn't) being ignored
git check-ignore -v path/to/file

# Check all rules affecting a file
git check-ignore -v --no-index path/to/file
```

### Stop Tracking an Already-Committed File

If a file was committed before being added to `.gitignore`, you must untrack it:

```bash
# Remove from index only (keeps local file)
git rm --cached path/to/file

# For a whole directory
git rm -r --cached path/to/dir/

# Then commit the removal
git commit -m "chore: stop tracking .env file"
```

After this, `.gitignore` will take effect for future changes.

### Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Committing `node_modules/` | Bloated repo, platform-specific binaries |
| Committing `.env` | Credentials exposed to all repo access |
| Committing `dist/` or `build/` | Large diffs, merge conflicts on generated code |
| Committing `*.log` files | Noise in history, large file sizes |
| Ignoring a file after it is already tracked | `.gitignore` silently has no effect |

---

## Git LFS (Large File Storage)

### The Problem

Git stores the full content of every version of every file in its object database. This is fine for text files, but catastrophic for:

- Video and audio files
- High-resolution images and PSDs
- ML model weights (`.h5`, `.pkl`, `.onnx`)
- Binary compiled artifacts
- Fonts and other large assets

A 100MB video committed 10 times = 1GB in `.git/objects`.

### How LFS Works

Git LFS replaces large files in your repository with small **text pointers**. The actual file content is stored on an LFS server (GitHub, GitLab, etc.).

```
version https://git-lfs.github.com/spec/v1
oid sha256:4d7a...
size 1048576
```

When you `git checkout`, LFS transparently downloads the actual file.

### Setup and Usage

```bash
# One-time global install
git lfs install

# Track a file pattern (writes to .gitattributes)
git lfs track "*.psd"
git lfs track "*.mp4"
git lfs track "models/*.h5"

# Commit the .gitattributes file
git add .gitattributes
git commit -m "chore: configure Git LFS tracking"

# Normal git workflow from here — LFS is transparent
git add design/mockup.psd
git commit -m "feat(design): add homepage mockup"
git push
```

**Inspection commands:**

```bash
# List all LFS-tracked files in the repo
git lfs ls-files

# Show LFS status
git lfs status

# See what patterns are tracked
cat .gitattributes
```

### Storage Limits by Provider

| Provider | Free LFS Storage | Free Bandwidth |
|----------|-----------------|----------------|
| GitHub | 1 GB | 1 GB/month |
| GitLab | 5 GB | 10 GB/month |
| Bitbucket | 1 GB | 1 GB/month |

### Alternatives to LFS

| Tool | Best For |
|------|----------|
| **DVC** (Data Version Control) | ML datasets and model artifacts |
| **AWS S3 + references** | Large assets stored externally, referenced in code |
| **Artifactory / Nexus** | Enterprise binary artifact management |

---

## Commit Signing

### Why Sign Commits?

Without signing, anyone with push access can set `git config user.email` to your address and make commits that appear to be from you. Commit signing cryptographically proves the commit came from you.

GitHub displays a green **Verified** badge on signed commits.

This matters in:
- Enterprise environments with audit requirements
- Open source projects that want supply chain security
- Regulated industries (finance, healthcare)

### SSH Signing (Simpler — Recommended)

```bash
# Use the same SSH key you use for authentication
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true

# Add your SSH key as a signing key on GitHub
# (Settings → SSH and GPG keys → New signing key)
```

### GPG Signing (Traditional)

```bash
# Generate a GPG key (if you don't have one)
gpg --full-generate-key

# List keys to get your key ID
gpg --list-secret-keys --keyid-format=long

# Configure git to use it
git config --global user.signingkey YOUR_KEY_ID
git config --global commit.gpgsign true

# Export your public key and add to GitHub
gpg --armor --export YOUR_KEY_ID
```

### Verifying Signatures

```bash
# Show signature status in log
git log --show-signature

# Verify a specific commit
git verify-commit <hash>
```

---

## Repository Hygiene

### Garbage Collection

Git accumulates loose objects over time. Run GC periodically to pack objects and reclaim space:

```bash
# Run garbage collection
git gc

# More aggressive (safe but slower)
git gc --aggressive
```

Git runs GC automatically, but manual runs help on large repositories.

### Prune Stale Remote References

```bash
# Fetch and remove stale remote-tracking branches in one step
git fetch --prune

# Prune without fetching
git remote prune origin
```

### Delete Merged Branches

```bash
# Delete merged local branch
git branch -d feature/my-feature

# Delete remote branch
git push --delete origin feature/my-feature

# Find all local branches already merged into main
git branch --merged main

# Bulk delete all local merged branches (excluding main and develop)
git branch --merged main | grep -vE '^\*|main|develop' | xargs git branch -d
```

### Keep Branches Short-Lived

Long-lived feature branches accumulate drift from the base branch and become harder to merge. Aim to:

- Merge feature branches within days, not weeks
- Use feature flags to hide incomplete work behind a toggle rather than in a branch
- Regularly rebase long-running branches to stay current

---

## Security: Never Commit Secrets

### What Counts as a Secret

Never commit any of the following to any repository (public or private):

- API keys and tokens
- Passwords and passphrases
- Private keys and certificates
- Database connection strings
- OAuth client secrets
- AWS/GCP/Azure credentials

### Prevention Strategies

**1. .gitignore all environment files**

```gitignore
.env
.env.*
*.pem
*.key
secrets/
credentials.json
```

**2. Pre-commit hooks**

```bash
# git-secrets (AWS)
brew install git-secrets
git secrets --install
git secrets --register-aws

# detect-secrets (Yelp)
pip install detect-secrets
detect-secrets scan > .secrets.baseline
detect-secrets audit .secrets.baseline

# gitleaks (comprehensive)
brew install gitleaks
gitleaks detect --source .
```

**3. GitHub Secret Scanning**

GitHub automatically scans pushes for known credential patterns (AWS keys, GitHub tokens, Stripe keys, and 200+ more) and alerts you or blocks the push.

### If a Secret IS Committed

> **Warning:** Treat any committed secret as fully compromised, even if you remove it immediately. Git history is distributed — anyone who cloned the repo may have a copy.

**Step 1: Rotate the secret IMMEDIATELY**
Before anything else — change the password, revoke the API key, generate new credentials.

**Step 2: Remove from history with git filter-repo**

```bash
# Install git-filter-repo
pip install git-filter-repo

# Remove a specific file from all history
git filter-repo --path secrets/.env --invert-paths

# Remove by content pattern
git filter-repo --replace-text expressions.txt
```

**Step 3: Force push and coordinate**

```bash
git push --force --all
git push --force --tags
```

Alert all collaborators — they must re-clone or reset their local copies.

**Step 4: Notify security**
If the secret had access to production systems or customer data, notify your security team and follow your incident response process.

---

## Summary

| Practice | Key Point |
|----------|-----------|
| Commit messages | Follow the 7 rules; use Conventional Commits for automation |
| Atomicity | One logical change per commit |
| .gitignore | Global for dev tools, per-repo for project artifacts |
| LFS | Use for binaries > ~1MB that change over time |
| Signing | Proves identity; use SSH signing for simplicity |
| Hygiene | Prune remote refs, delete merged branches, run `git gc` |
| Secrets | Never commit them; if you do, rotate first, then rewrite history |

---

## Knowledge Check

1. What is the maximum recommended length for a commit subject line?
2. In Conventional Commits, what type prefix would you use for a performance improvement?
3. What command removes a file from Git tracking without deleting it from disk?
4. What does Git LFS store in the repository instead of the actual file?
5. What must you do FIRST if you accidentally commit an API key?
6. What git command removes stale remote-tracking branches?
7. Why does commit atomicity make `git bisect` more effective?

---

## Hands-On Exercises

### Exercise 1 — Global .gitignore Setup

1. Create `~/.gitignore_global` with entries for your OS and editor
2. Run `git config --global core.excludesfile ~/.gitignore_global`
3. Create a test repo, create a `.DS_Store` file, and verify it is ignored
4. Run `git check-ignore -v .DS_Store` to confirm which rule is matching

### Exercise 2 — Conventional Commits Pre-commit Hook

Create a `commit-msg` hook that validates Conventional Commits format:

```bash
#!/bin/bash
# .git/hooks/commit-msg

COMMIT_MSG_FILE="$1"
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

PATTERN="^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\(.+\))?: .{1,72}"

if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
  echo "ERROR: Commit message does not follow Conventional Commits format."
  echo "Expected: type(scope): description"
  echo "Example:  feat(auth): add OAuth2 login"
  exit 1
fi

exit 0
```

```bash
chmod +x .git/hooks/commit-msg

# Test with a bad message (should be rejected)
git commit -m "updated stuff"

# Test with a good message (should pass)
git commit -m "feat(auth): add login form validation"
```

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./12-advanced-git.md">← Previous: Advanced Git</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./14-common-mistakes.md">Next: Common Mistakes →</a>
</div>
