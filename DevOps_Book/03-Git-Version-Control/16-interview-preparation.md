# Chapter 16 — Interview Preparation

## Learning Objectives

By the end of this chapter you will be able to:

- Answer the most common Git conceptual questions with precision and confidence
- Walk through real-world Git scenarios step by step
- Explain DevOps-specific Git patterns (GitOps, shallow clones, semantic versioning via tags)
- Pass Git knowledge sections in junior, mid-level, and senior DevOps/backend engineer interviews

## Prerequisites

Complete Chapter 15 — Hands-On Projects before working through this chapter. You should already be comfortable with branching, rebasing, undoing changes, hooks, and branching strategies.

---

## Core Concept Questions

### 1. What is the difference between `git fetch` and `git pull`?

`git fetch` downloads commits, branches, and tags from a remote repository into your local remote-tracking references (e.g., `origin/main`) but does **not** touch your working directory or current branch. Your local work is completely unaffected.

`git pull` is a two-step operation: it runs `git fetch` first, then immediately integrates the fetched changes into your current branch — either via a merge (default) or via a rebase if you pass `--rebase`.

**When to use which:**

- Use `git fetch` when you want to inspect what changed on the remote before integrating (`git log origin/main..HEAD` after fetching).
- Use `git pull` when you are ready to incorporate upstream changes immediately.
- In automated CI pipelines `git fetch` is preferred because it gives you full control over what happens next.

```bash
# Inspect before merging
git fetch origin
git log --oneline HEAD..origin/main   # what will come in
git merge origin/main                  # merge when ready

# vs the quick shorthand
git pull origin main
```

---

### 2. What is the difference between merge and rebase? When would you use each?

Both integrate changes from one branch into another, but they produce different histories.

**Merge** creates a new "merge commit" that has two parents — the tip of your branch and the tip of the target branch. The original commit timestamps and SHAs are preserved. History is non-linear but accurate.

**Rebase** replays your branch's commits on top of another branch, writing brand-new commits (new SHAs, today's timestamp). History becomes linear, as if you had branched off the latest commit all along.

| Aspect | Merge | Rebase |
|---|---|---|
| History shape | Non-linear (merge commits) | Linear (no merge commits) |
| Commit SHAs | Preserved | Rewritten |
| Safe on shared branches? | Yes | No — rewrites history |
| `git bisect` friendliness | Works | Works even better |

**Use merge when:**
- Merging a feature branch into `main` via a PR (preserves context)
- Working on a shared branch that others have checked out
- The non-linear history is acceptable or preferred

**Use rebase when:**
- Updating your local feature branch with upstream `main` changes before opening a PR
- Cleaning up messy WIP commits with interactive rebase before review
- Your team follows a linear history policy

**Golden rule:** Never rebase commits that have been pushed to a shared branch.

---

### 3. Explain `git reset --soft`, `--mixed`, and `--hard` with examples

All three move the branch pointer (HEAD) to a specified commit. They differ in what they do to the staging area (index) and working directory.

| Flag | HEAD moves | Index (staging) | Working directory |
|---|---|---|---|
| `--soft` | Yes | Unchanged | Unchanged |
| `--mixed` (default) | Yes | Reset to HEAD | Unchanged |
| `--hard` | Yes | Reset to HEAD | Reset to HEAD |

**`--soft` example** — undo last commit but keep changes staged, ready to recommit:

```bash
git reset --soft HEAD~1
# Your changes are staged. Just run `git commit` again with a better message.
```

**`--mixed` example** (default) — undo last commit, unstage changes, keep files:

```bash
git reset HEAD~1        # same as git reset --mixed HEAD~1
# Changes are back in working directory, unstaged.
# Use this to re-stage selectively: git add -p
```

**`--hard` example** — discard last commit AND all its changes permanently:

```bash
git reset --hard HEAD~1
# WARNING: working directory changes are gone. Use git reflog to recover if needed.
```

---

### 4. What is a detached HEAD state and how do you get out of it?

Normally, `HEAD` is a symbolic reference pointing to a branch name (e.g., `refs/heads/main`). When you check out a specific commit SHA, a tag, or a remote branch directly, `HEAD` points to a commit instead of a branch. This is "detached HEAD."

```bash
git checkout abc1234    # detached HEAD — HEAD points to commit abc1234
```

**Danger:** Any new commits you make in this state are not reachable by any branch. If you switch away without saving them, those commits will eventually be garbage-collected.

**How to get out:**

Option 1 — If you made no new commits, just switch back to a branch:
```bash
git switch main
```

Option 2 — If you made commits you want to keep, create a branch to capture them:
```bash
git switch -c my-experimental-work
# Your commits are now safely on a branch.
```

Option 3 — Find commits made in detached HEAD state via reflog:
```bash
git reflog
git switch -c recovery-branch abc1234
```

---

### 5. How do you revert a commit that has already been pushed to the main branch?

You **never** use `git reset` on a shared branch because it rewrites history that others already have. Instead, use `git revert`, which creates a new commit that undoes the changes of the target commit.

```bash
# Find the commit hash to undo
git log --oneline

# Create a revert commit
git revert abc1234

# Push normally — no force push required
git push origin main
```

`git revert` is safe for public/shared branches because it only adds to history. The original commit still exists; the revert commit applies the inverse diff.

**Reverting a merge commit** requires specifying a parent:
```bash
git revert -m 1 <merge-commit-sha>
# -m 1 means: "revert to the first parent" (usually the branch you merged into)
```

---

### 6. What is the difference between `git revert` and `git reset`?

| Aspect | `git revert` | `git reset` |
|---|---|---|
| What it does | Creates a new commit that undoes changes | Moves HEAD (and branch) to an earlier commit |
| Rewrites history? | No | Yes (for commits after the new HEAD) |
| Safe on shared branches? | Yes | No |
| Commit still visible? | Yes (in history) | Not directly (reachable via reflog only) |
| Use case | Undo a pushed commit | Undo a local commit before pushing |

**Rule of thumb:** If the commit exists on a remote that others are using, always use `git revert`. If the commit is purely local, either works — `git reset` is faster.

---

### 7. How do you squash multiple commits into one?

The standard approach is interactive rebase:

```bash
# Squash the last 4 commits into one
git rebase -i HEAD~4
```

In the editor that opens, change `pick` to `squash` (or `s`) for every commit you want to fold into the one above it. Keep `pick` on the first commit.

```
pick  abc1111  feat: add user model
squash abc2222  wip
squash abc3333  fix typo
squash abc4444  fix2
```

Git will then prompt you for a combined commit message.

**Alternative — merge squash:**
```bash
git switch main
git merge --squash feature/my-feature
git commit -m "feat: implement user model"
```

This takes all commits from `feature/my-feature`, squashes them into the staging area, and lets you write one clean commit message. The feature branch itself is not altered.

---

### 8. What is a merge conflict and walk me through how you resolve it?

A merge conflict occurs when Git cannot automatically reconcile changes because two branches modified the **same lines** of the same file (or one branch deleted a file the other modified).

Git marks the conflicting sections in the file with conflict markers:

```
<<<<<<< HEAD
const PORT = 3000;
=======
const PORT = 8080;
>>>>>>> feature/config-update
```

**Step-by-step resolution:**

```bash
# 1. Attempt the merge (conflict occurs)
git merge feature/config-update

# 2. See which files conflict
git status

# 3. Open each conflicting file and resolve:
#    - Keep HEAD's version, the incoming version, or write a new version
#    - Remove ALL conflict markers (<<<<<<<, =======, >>>>>>>)

# 4. Stage the resolved file
git add src/config.js

# 5. Complete the merge
git commit   # Git pre-fills the merge commit message
```

**Tools that help:**
- `git mergetool` — opens a configured visual diff tool
- VS Code, IntelliJ, and most IDEs have built-in merge editors
- `git diff --diff-filter=U` — shows only unresolved files

**Prevention tips:** Communicate with your team, merge `main` into your feature branch frequently, and keep feature branches short-lived.

---

### 9. What is `git stash` and when would you use it vs creating a WIP commit?

`git stash` saves your uncommitted changes (both staged and unstaged) onto a stack and reverts your working directory to the last commit. Your changes are preserved but temporarily hidden.

```bash
git stash              # save changes
git stash pop          # restore and remove from stack
git stash list         # see all stash entries
git stash apply stash@{2}  # restore specific entry without removing
```

**Use `git stash` when:**
- You need to quickly switch branches to fix an urgent bug but your current work is incomplete and not ready to commit
- You want a clean working directory to run tests or check something out
- The interruption will be short (same day)

**Use a WIP commit when:**
- You are about to leave for the day and want the work in remote history as a backup
- The "work in progress" may span multiple days
- You want to share the incomplete work with a teammate for review

```bash
# WIP commit pattern
git add -A
git commit -m "wip: half-done auth middleware — DO NOT MERGE"
# Later, before opening a PR:
git rebase -i HEAD~N   # squash/fixup the wip commits
```

**Key difference:** A stash exists only on your local machine (unless pushed via `git stash push` remote workflows). A WIP commit is in history and can be pushed.

---

### 10. Explain fast-forward merge vs 3-way merge — when does each happen?

**Fast-forward merge** happens when the branch being merged into has no new commits since the feature branch diverged. Git simply moves the target branch pointer forward — no new commit is created.

```
main:   A - B
                \
feature:         C - D

After ff merge:
main:   A - B - C - D   (no merge commit)
```

```bash
git merge feature/login    # fast-forward if main hasn't moved
```

**3-way merge** happens when both branches have diverged — both have commits the other does not. Git finds the common ancestor, compares the three snapshots (common ancestor, tip of branch A, tip of branch B), and creates a new merge commit.

```
main:   A - B - E
                \
feature:     C - D

After 3-way merge:
main:   A - B - E - M   (M is the merge commit with parents E and D)
                    |
                    D
```

**Summary:**

| Scenario | Merge type |
|---|---|
| Feature branch, `main` not updated since branch point | Fast-forward |
| Both branches have new commits | 3-way merge |
| `git merge --no-ff` | Forces a merge commit even when ff is possible |

Teams often use `--no-ff` to preserve the context that a group of commits belonged to a feature branch.

---

### 11. What is `git cherry-pick` and give a real-world use case?

`git cherry-pick` applies the changes introduced by one or more existing commits onto your current branch, creating a new commit (new SHA, same diff).

```bash
git cherry-pick <commit-sha>            # single commit
git cherry-pick abc123..def456          # range (exclusive..inclusive)
git cherry-pick -n <commit-sha>         # apply changes but don't auto-commit
```

**Real-world use case — backporting a security fix:**

Your team maintains `main` and `release/2.4`. A critical SQL injection fix is merged to `main` as commit `f3a9b2c`. The release branch cannot receive all of `main` (it would include unreleased features), so you cherry-pick just the fix:

```bash
git switch release/2.4
git cherry-pick f3a9b2c
git push origin release/2.4
# Trigger CI/CD → new patch release 2.4.1
```

**Other common uses:**
- Applying a hotfix made directly on `main` to older supported versions
- Rescuing a single good commit from an otherwise abandoned branch
- Pulling a colleague's commit into your branch before their PR is merged

---

### 12. How do you find which commit introduced a bug?

Use `git bisect` — a binary search through your commit history.

```bash
# 1. Start bisect
git bisect start

# 2. Mark current state as bad
git bisect bad

# 3. Mark a known-good commit (could be days/weeks ago)
git bisect good v2.0.0

# Git checks out the midpoint commit
# 4. Run your test or reproduction steps, then tell Git the result
git bisect good   # or
git bisect bad

# Git narrows down — repeat until it identifies the first bad commit
# "abc1234 is the first bad commit"

# 5. End and return to original branch
git bisect reset
```

**Automating bisect with a script:**
```bash
git bisect run npm test   # runs npm test at each step; exit 0 = good, non-zero = bad
```

Bisect is O(log n) — in a repo with 1,000 commits between good and bad, it finds the culprit in ~10 steps.

**Alternative quick approach** — `git log -S "string"` (pickaxe search):
```bash
git log -S "dangerousFunction" --oneline
# Shows commits that added or removed that string
```

---

### 13. What are Git hooks? Give two practical examples

Git hooks are scripts stored in `.git/hooks/` that Git executes automatically at specific points in the Git workflow. They can be written in any scripting language (bash, Python, Node.js, etc.).

**Two types:**
- **Client-side hooks** run on your local machine (pre-commit, commit-msg, pre-push, etc.)
- **Server-side hooks** run on the remote (pre-receive, update, post-receive)

**Example 1 — pre-commit: run linter before every commit**

```bash
#!/bin/bash
# .git/hooks/pre-commit
npm run lint
if [ $? -ne 0 ]; then
  echo "Lint failed. Fix errors before committing."
  exit 1   # non-zero exit aborts the commit
fi
```

**Example 2 — commit-msg: enforce Conventional Commits format**

```bash
#!/bin/bash
# .git/hooks/commit-msg
MSG=$(cat "$1")
PATTERN="^(feat|fix|docs|style|refactor|test|chore|perf|ci)(\(.+\))?: .{1,72}$"
if ! echo "$MSG" | grep -qE "$PATTERN"; then
  echo "ERROR: Commit message must follow Conventional Commits."
  echo "  e.g., feat(auth): add OAuth2 login"
  exit 1
fi
```

**Sharing hooks with the team** — hooks in `.git/hooks/` are not committed. Use a package like `husky` (Node.js projects) or store hooks in a `scripts/hooks/` directory and configure `core.hooksPath`:

```bash
git config core.hooksPath scripts/hooks
```

---

### 14. What is the difference between annotated and lightweight tags?

**Lightweight tag** — a simple pointer to a commit, stored as a file in `.git/refs/tags/`. It contains only the commit SHA. No extra metadata.

```bash
git tag v1.0.0              # lightweight
git show v1.0.0             # shows the commit directly
```

**Annotated tag** — a full Git object stored in the object database. It has its own SHA and stores: tagger name, email, date, and a tagging message. Can be signed with GPG.

```bash
git tag -a v1.0.0 -m "Release 1.0.0 — stable production release"
git show v1.0.0             # shows tag object metadata, then the commit
```

**Key differences:**

| Aspect | Lightweight | Annotated |
|---|---|---|
| Stored as | File pointer | Full Git object |
| Metadata | None | Tagger, date, message |
| GPG signing | No | Yes (`-s` flag) |
| `git describe` | Not used | Used |
| Best for | Local bookmarks | Official releases |

**Best practice:** Always use annotated tags for releases (`v1.0.0`, `v2.1.3`). Use lightweight tags for local temporary markers.

---

### 15. What is `git reflog` and how has it saved developers from data loss?

`git reflog` records every movement of `HEAD` on your local machine — every commit, checkout, rebase, reset, merge, stash. It retains entries for 90 days by default.

```bash
git reflog
# Output:
# abc1234 HEAD@{0}: reset: moving to HEAD~3
# def5678 HEAD@{1}: commit: feat: add payment module
# ghi9012 HEAD@{2}: commit: fix: correct tax calculation
```

**Real data-loss recovery scenarios:**

**Scenario A — Accidental `git reset --hard`:**
```bash
git reset --hard HEAD~5   # "Oh no, I just deleted 5 commits!"
git reflog                # Find the SHA before the reset
git reset --hard def5678  # Restore to it
```

**Scenario B — Deleted branch:**
```bash
git branch -D feature/payment   # Deleted before merging
git reflog                       # Find last commit of that branch
git switch -c feature/payment abc1234  # Recreate branch at that commit
```

**Scenario C — Lost commits after a botched rebase:**
```bash
git reflog                    # Find the SHA from before the rebase started
git reset --hard <sha>        # Restore
```

**Important:** `git reflog` is local only — it does not exist on the remote. This means if you lose commits and they were never pushed, reflog is your only safety net.

---

### 16. What is the difference between GitFlow and trunk-based development?

**GitFlow** is a branching model with multiple long-lived branches:
- `main` — production-ready code
- `develop` — integration branch
- `feature/*` — individual features branched from `develop`
- `release/*` — stabilization branches before going to `main`
- `hotfix/*` — emergency fixes branched directly from `main`

Releases go through: `feature` → `develop` → `release` → `main`.

**Trunk-based development (TBD)** has one long-lived branch (`main`/`trunk`). Developers commit directly to trunk or merge very short-lived feature branches (< 1 day to 2 days). Releases are done from trunk using tags or release branches cut from trunk.

**Comparison:**

| Aspect | GitFlow | Trunk-Based |
|---|---|---|
| Long-lived branches | Many (main, develop, release, hotfix) | One (trunk/main) |
| Feature branch lifetime | Days to weeks | Hours to 2 days |
| Release process | Explicit release branches | Tags on trunk, or short-lived release cuts |
| CI/CD compatibility | Complex | Natural fit |
| Team size | Works for larger teams with explicit release cycles | Scales to any size; preferred for high-frequency deployment |
| Risk of merge conflicts | Higher (long-lived branches diverge) | Lower (frequent integration) |

**When to choose GitFlow:** You ship versioned software (mobile apps, libraries, enterprise products) with explicit release cycles and long QA phases.

**When to choose TBD:** You practice continuous delivery, deploy multiple times per day, and invest in feature flags for incomplete work.

---

### 17. What is a protected branch and why is it important in a team?

A **protected branch** is a branch with rules enforced by the Git hosting platform (GitHub, GitLab, Bitbucket) that prevent certain operations.

**Common protection rules:**
- Require pull requests before merging — no direct pushes
- Require a minimum number of PR approvals (e.g., 2 reviewers)
- Require status checks to pass (CI pipeline must be green)
- Require branches to be up to date before merging
- Restrict who can push (only certain teams or roles)
- Prevent force pushes
- Prevent deletion

**Why it matters:**

1. **Prevents accidental breakage** — a single developer cannot push broken code directly to `main` and take down production
2. **Enforces code review** — every change is reviewed by at least one other engineer
3. **Maintains a deployable state** — CI must pass, so `main` is always theoretically deployable
4. **Audit trail** — every change to `main` came through a PR with a traceable discussion
5. **Compliance** — SOC 2, ISO 27001, and other frameworks require change approval processes; protected branches provide evidence

**Setup on GitHub:**
Repository Settings → Branches → Add branch protection rule → set rules on `main`.

---

## Scenario-Based Questions

### Scenario 1: "You committed an API key to a public GitHub repo 5 minutes ago. What do you do, step by step?"

**Step 1 — Revoke immediately (parallel to Git cleanup)**

Before touching Git, go to your cloud provider (AWS, GCP, GitHub, Stripe, etc.) and revoke/rotate the exposed key. Assume it is already compromised — GitHub's secret scanning may have already detected and notified the service provider.

**Step 2 — Remove from Git history**

Since the commit is on a public repo, `git reset` alone is not enough — the commit is in GitHub's cache. Use `git filter-repo`:

```bash
pip install git-filter-repo

git filter-repo --path config/secrets.env --invert-paths
# Rewrites entire history, removing the file

# OR remove a specific string pattern:
git filter-repo --replace-text <(echo "AKIAIOSFODNN7EXAMPLE==>REDACTED")
```

**Step 3 — Force push**

```bash
git push origin --force --all
git push origin --force --tags
```

**Step 4 — Contact GitHub support**

GitHub caches commits. File a support ticket to purge their cached copies.

**Step 5 — Audit**

Check GitHub's "Secret scanning alerts" tab. Review git log to ensure no other secrets are lurking.

**Step 6 — Prevent recurrence**

```bash
# Add to .gitignore
echo "*.env" >> .gitignore
echo ".env*" >> .gitignore

# Add a pre-commit hook with truffleHog or detect-secrets
pip install detect-secrets
detect-secrets scan > .secrets.baseline
```

---

### Scenario 2: "Your feature branch has 15 messy commits with messages like 'wip', 'fix', 'fix2'. You need to open a PR. How do you clean this up?"

**Step 1 — Make sure your branch is up to date with main:**

```bash
git fetch origin
git rebase origin/main
```

Resolve any conflicts during the rebase.

**Step 2 — Interactive rebase to squash:**

```bash
git rebase -i origin/main
# or if you know it's 15 commits:
git rebase -i HEAD~15
```

**Step 3 — In the editor**, reorganize the commits:

```
pick  a1b2c3d  feat: initial user auth scaffold
squash  b2c3d4e  wip
squash  c3d4e5f  fix
squash  d4e5f6a  fix2
squash  e5f6a7b  wip
pick  f6a7b8c  feat: add JWT token validation
squash  g7b8c9d  fix token expiry bug
...
```

Change `squash` to `fixup` if you want to discard those commit messages entirely (keeps only the `pick` message).

**Step 4 — Write clean commit messages** when prompted:

```
feat(auth): implement user authentication with JWT

- Add login/logout endpoints
- Validate JWT tokens with expiry
- Add refresh token rotation
- Unit tests for auth middleware
```

**Step 5 — Force push (it's your feature branch, no one else is on it):**

```bash
git push --force-with-lease origin feature/user-auth
```

`--force-with-lease` is safer than `--force`: it will refuse if someone else pushed to the branch since your last fetch.

**Step 6 — Open the PR.** Reviewers now see 2–3 well-scoped commits instead of 15 noise commits.

---

### Scenario 3: "You need to apply a critical security fix from main to the release/2.1 branch without merging all of main. How?"

This is the canonical `git cherry-pick` use case.

**Step 1 — Identify the exact commit on main:**

```bash
git log main --oneline | head -20
# Find the security fix commit: abc1234 fix(auth): prevent SQL injection in user query
```

**Step 2 — Switch to the release branch:**

```bash
git switch release/2.1
git pull origin release/2.1   # make sure you're up to date
```

**Step 3 — Cherry-pick the fix:**

```bash
git cherry-pick abc1234
```

If there are conflicts (the code structure differs between branches):
```bash
# Resolve conflicts in the files
git add <resolved-files>
git cherry-pick --continue
```

**Step 4 — Push and tag a patch release:**

```bash
git push origin release/2.1
git tag -a v2.1.1 -m "Security patch: fix SQL injection in user query"
git push origin v2.1.1
```

**Step 5 — Trigger CI/CD** for `release/2.1` to deploy `v2.1.1`.

**Note:** If the fix involved multiple commits:
```bash
git cherry-pick abc1234^..def5678   # apply a range
```

---

### Scenario 4: "Two developers both modified the same config file in their feature branches. Walk me through what happens when they merge."

**Setup:**
- `feature/alice` modifies `config/database.yml` line 12: changes host from `localhost` to `db.internal`
- `feature/bob` modifies `config/database.yml` line 12: changes host from `localhost` to `postgres.prod`

**When Alice merges first:**

```bash
git switch main
git merge feature/alice   # succeeds, fast-forward or 3-way, no conflict
```

**When Bob tries to merge:**

```bash
git merge feature/bob
# CONFLICT (content): Merge conflict in config/database.yml
```

Git marks the file:
```yaml
<<<<<<< HEAD
host: db.internal
=======
host: postgres.prod
>>>>>>> feature/bob
```

**Resolution process:**

Bob (or a merge coordinator) opens the file, decides the correct value — perhaps `postgres.prod` is right for the environment, or they need to discuss with Alice:

```yaml
host: postgres.prod   # resolved value
```

```bash
git add config/database.yml
git commit   # completes the merge
```

**Key insight for the interviewer:** Git detects conflicts at the line level, not the semantic level. Two people could change the same variable in ways that are semantically incompatible without a line conflict. Code review and communication prevent this more than tooling does.

---

### Scenario 5: "How would you set up Git for a 15-person team practicing continuous delivery?"

**Branching strategy:** Trunk-based development — one `main` branch, short-lived feature branches (< 2 days), merging via PR.

**Branch protection on `main`:**
- Require 1–2 PR approvals
- Require CI (tests + linting) to pass
- Require branch to be up to date before merge
- Prevent direct pushes, prevent force pushes

**PR conventions:**
- Feature branches named `feature/<ticket-id>-short-description`
- All branches rebased on latest `main` before PR
- Interactive rebase to clean up commits before PR
- Conventional Commits enforced via commit-msg hook + CI check

**CI/CD integration:**
- Every push to any branch triggers tests
- Every merge to `main` triggers deployment to staging automatically
- Git tags (`v1.2.3`) trigger production deployment

**Hooks (shared via `core.hooksPath`):**
- `pre-commit`: lint, format check
- `commit-msg`: Conventional Commits validation

**Feature flags:** Because branches are short-lived, incomplete features are hidden behind feature flags (LaunchDarkly, Unleash) rather than long feature branches.

**Release process:**
```bash
git tag -a v1.4.0 -m "Release 1.4.0"
git push origin v1.4.0
# CI/CD sees the tag, builds release artifact, deploys to production
```

**Protected `main` means:** You can confidently deploy `main` at any time. Continuous delivery is achieved.

---

## System Design Question

### "Design a Git workflow for a microservices company doing 10 deployments per day"

**Branching strategy: Trunk-based development**

With 10 deployments per day, GitFlow is too heavyweight — its `develop` and `release` branches would be permanent bottlenecks. Trunk-based development is the right choice:

- `main` is always deployable
- Feature branches live for hours to 2 days maximum
- Merges to `main` trigger automated deployment

**PR Process:**

```
feature/JIRA-123-payment-retry → main
```

1. Developer creates a branch, pushes, opens a PR as soon as the first commit lands
2. CI runs automatically: unit tests, integration tests, linting, security scanning
3. One required reviewer (two for high-risk areas like auth or payments)
4. Branch must be up to date with `main` before merge
5. Merge strategy: "Squash and merge" for small features; "Merge commit" for larger grouped features

**CI/CD Integration:**

- `main` merge → deploy to staging → run smoke tests → deploy to production (if smoke tests pass)
- Tags → deploy to a specific environment or trigger a release artifact build
- Every PR branch → ephemeral preview environment (if budget allows)

**Tagging and Releases:**

Use semantic versioning via git tags triggered by Conventional Commits:

```bash
# Automated by semantic-release or similar tool
# "feat:" commits → minor version bump
# "fix:" commits → patch version bump
# "BREAKING CHANGE:" → major version bump

git tag -a v3.14.0 -m "chore: release v3.14.0"
```

Each microservice has its own tag namespace if in a polyrepo, or a prefix in a monorepo:
`payment-service/v2.1.0`, `auth-service/v1.8.3`

**Monorepo vs Polyrepo:**

| Factor | Monorepo | Polyrepo |
|---|---|---|
| Atomic cross-service changes | Easy — one commit | Hard — multiple PRs to coordinate |
| CI scope | Complex — need change detection to only build affected services | Simple — each repo has its own CI |
| Shared library updates | Easy — update once, all services get it | Hard — update library + update each consumer |
| Onboarding | One clone, unified tooling | Multiple repos to set up |
| Build tooling | Requires Nx, Bazel, or Turborepo for efficiency | Standard tools work |

**Recommendation for a 10-deploy/day microservices company:** Start with a polyrepo for clear service ownership and simple CI. Move to a monorepo only if you find yourself spending significant time coordinating cross-service changes.

**Secrets Management:**

- Never commit secrets — enforce with `detect-secrets` pre-commit hook and GitHub secret scanning
- Secrets live in a vault (HashiCorp Vault, AWS Secrets Manager)
- CI/CD injects secrets at runtime via environment variables
- Rotate keys via the vault, not by editing repo files

**Complete workflow summary:**

```
Developer writes code
    ↓
Pre-commit hook: lint + format
    ↓
Commit with Conventional Commits message
    ↓
Push → CI runs tests on feature branch
    ↓
PR opened → reviewer approves + CI passes
    ↓
Squash merge to main
    ↓
CI/CD: staging deploy → smoke tests
    ↓
Automatic production deploy (or manual gate for major changes)
    ↓
semantic-release creates tag + changelog
```

---

## Quick-Fire Q&A

| Question | Answer |
|---|---|
| What does `git pull --rebase` do? | Fetches and replays your local commits on top of the updated remote branch, producing linear history |
| What flag makes force push safer? | `--force-with-lease` — refuses if someone else pushed since your last fetch |
| What does `git describe --tags` output? | The nearest tag + number of commits since it + short SHA, e.g., `v1.2.0-5-gabcdef1` |
| How do you see who last changed a line? | `git blame <filename>` |
| What is `git bisect`? | Binary search through commit history to find which commit introduced a bug |
| How do you list all branches (local + remote)? | `git branch -a` |
| What does `git clean -fd` do? | Deletes all untracked files and directories (WARNING: irreversible) |
| How do you undo a staged change without losing it? | `git restore --staged <file>` |
| What is the difference between `origin` and `upstream`? | `origin` = your fork; `upstream` = the original repo you forked from |
| How do you rename a branch? | `git branch -m old-name new-name` |
| What does `--no-ff` do in a merge? | Forces creation of a merge commit even when fast-forward is possible |
| How do you see the diff of a specific commit? | `git show <sha>` |
| What is `git worktree`? | Checks out multiple branches simultaneously into different directories |
| How do you clone only the latest snapshot (faster CI)? | `git clone --depth 1 <url>` |
| What does `git log --follow` do? | Follows file renames in history |
| How do you push a new local branch to remote? | `git push -u origin <branch-name>` |
| What is `FETCH_HEAD`? | A special ref recording what was last fetched |
| What does `git shortlog -sn` show? | Commit count per author, sorted by count |
| How do you apply a patch file? | `git apply patch.diff` or `git am patch.mbox` |
| What is the staging area (index)? | A layer between working directory and repo where you build your next commit |
| How do you see all tags? | `git tag -l` or `git tag --list` |
| What does `git submodule update --init` do? | Initializes and fetches content for all submodules |
| How do you check if a branch is merged? | `git branch --merged main` |
| What is `git archive`? | Creates a tar/zip of a tree without `.git` metadata (for deployments) |
| How do you add a remote? | `git remote add <name> <url>` |
| What does `git log -p` show? | Full diff (patch) for each commit |
| How do you sign a commit? | `git commit -S` (requires GPG key configured) |
| What is a bare repository? | A repo without a working directory — used on servers/remotes |
| How do you see your Git config? | `git config --list` |
| What does `git gc` do? | Runs garbage collection — packs loose objects, prunes unreachable objects |

---

## DevOps-Specific Git Knowledge

### Shallow Clones in CI

Full repository clones in CI are wasteful — a repo with years of history can be hundreds of MB, slowing every pipeline run.

```bash
# Clone only the latest snapshot
git clone --depth 1 https://github.com/org/repo.git

# Fetch only the branch you need (GitHub Actions does this by default)
git clone --depth 1 --branch main https://github.com/org/repo.git
```

**Trade-offs:**
- Faster CI startup time (significantly for large repos)
- `git bisect`, `git log`, and `git describe` won't work fully without full history
- `git fetch --unshallow` converts back to a full clone if needed

**In GitHub Actions**, the `actions/checkout` action uses `--depth 1` by default. Pass `fetch-depth: 0` for full history when you need `git describe` or changelog generation.

---

### `git describe --tags`

Generates a human-readable name for a commit based on the nearest tag. Useful for automatic versioning in CI.

```bash
git describe --tags
# Output: v1.2.0-5-gabcdef1
# Meaning: 5 commits after tag v1.2.0, at commit gabcdef1
```

This is used by tools like `setuptools` (Python), `cargo` (Rust), and custom build scripts to automatically derive a version without hardcoding it.

```bash
# In a Makefile or CI script:
VERSION=$(git describe --tags --always)
docker build -t myapp:$VERSION .
```

`--always` falls back to the short SHA if no tags exist.

---

### Monorepo Trade-offs

**Pros of a monorepo:**
- Atomic commits across multiple services — one PR can update service A and its consumer service B simultaneously
- Shared tooling, linting, and CI configuration lives in one place
- Easier dependency management — no "which version of the shared library should I use?"
- Simpler onboarding — one `git clone` gets everything
- Refactoring across service boundaries is easier

**Cons of a monorepo:**
- Repository size grows large — slow clones without tooling (sparse checkout, shallow clone)
- CI must be smart about only building changed services (requires Nx, Turborepo, Bazel, or custom scripts)
- Permissions are harder — you cannot give a contractor access to one service without access to the entire repo
- Git history becomes noisy — commits from 20 teams in one log
- Merge conflicts increase as more people commit to the same repo

**Used by:** Google (Piper/Blaze), Meta (Mercurial monorepo), Microsoft (Git VFS), Airbnb, Uber.

---

### GitOps Concept

**GitOps** is an operational model where Git is the **single source of truth** for both application code AND infrastructure configuration. The desired state of the system is declared in Git, and automated tools continuously reconcile the actual state to match.

**Core principles:**
1. Declarative infrastructure (Kubernetes manifests, Helm charts, Terraform)
2. Versioned and immutable — all changes go through Git (PRs, reviews, history)
3. Pulled automatically — agents in the cluster pull from Git, rather than CI pushing to the cluster
4. Continuously reconciled — if someone manually changes a cluster resource, the GitOps agent reverts it

**The .gitops pattern:**

```
Developer opens PR changing a Kubernetes Deployment manifest
    ↓
PR reviewed and merged to main
    ↓
ArgoCD / Flux (running in the cluster) detects the change
    ↓
Agent pulls the new manifest from Git
    ↓
Agent applies the change to the cluster
    ↓
If someone manually edits the cluster (kubectl edit), ArgoCD reverts it
```

**Tools:** ArgoCD, Flux, Jenkins X

**Why it matters for interviews:** GitOps is a key pattern in modern Kubernetes/cloud-native environments. Understanding it shows that you see Git as more than a code storage tool — it is an infrastructure control plane.

```
Repo structure for GitOps:
apps/
  frontend/
    deployment.yaml
    service.yaml
  backend/
    deployment.yaml
    service.yaml
infra/
  networking/
  storage/
```

ArgoCD watches a specific path in a specific branch. A change to `apps/backend/deployment.yaml` merged to `main` triggers a reconciliation cycle that updates the running pods.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="15-projects.md">← Previous: Hands-On Projects</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="17-course-summary.md">Next: Course Summary →</a>
</div>
