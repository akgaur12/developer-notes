# Chapter 11 — GitHub & GitLab Workflows

## Learning Objectives

By the end of this chapter you will be able to:

- Explain the purpose and anatomy of a Pull Request (PR) / Merge Request (MR)
- Write a clear, reviewable PR that speeds up the review cycle
- Conduct and respond to code reviews effectively
- Contribute to open source projects using the fork workflow
- Configure protected branches and CODEOWNERS
- Use GitHub Actions for basic CI and the GitHub CLI for day-to-day PR work
- Create annotated tags and GitHub/GitLab releases following semantic versioning

**Prerequisites:** Chapter 10 — Branching Strategies

---

## Pull Requests and Merge Requests

A **Pull Request** (GitHub terminology) or **Merge Request** (GitLab terminology) is two things simultaneously:

1. **A request to merge** one branch into another
2. **A collaboration surface** — a shared space for diff review, inline comments, CI status, and approval tracking

The terms PR and MR are functionally identical. This chapter uses "PR" but everything applies equally to GitLab MRs.

### PR Lifecycle

```
1. Push your feature branch to the remote
         git push -u origin feature/JIRA-123-add-oauth-login

2. Open a PR on GitHub/GitLab
         - Select base branch (main / develop)
         - Write title and description

3. Review cycle
         - Reviewers leave comments, suggestions, and approval/rejection
         - Author addresses feedback with new commits

4. CI checks run automatically on every push

5. Approval gates met + CI green → branch is mergeable

6. Merge the PR (choose merge strategy)

7. Delete the source branch
```

### Writing a Good PR

**Title format:**

Use the imperative mood. Reference the ticket number. Keep it under 72 characters.

```
feat: add OAuth login via Google [JIRA-123]
fix: prevent null pointer in payment handler [JIRA-456]
chore: upgrade Spring Boot to 3.2.0
docs: document rate-limiting behaviour
```

**Body checklist:**

A good PR body answers three questions for the reviewer:

- **What changed?** — a concise bullet-list summary of the changes
- **Why?** — context and motivation (link to ticket, Slack thread, design doc)
- **How to test?** — reproduction steps, environment setup, expected outcome

**PR size matters.** Diffs over 400 lines are consistently reviewed less carefully. Aim for small, focused PRs:

- One logical change per PR
- Split refactors from behaviour changes
- If a PR is large by necessity, add a detailed walkthrough in the description

### PR Templates

Store a template in `.github/pull_request_template.md` (GitHub) or `.gitlab/merge_request_templates/Default.md` (GitLab) and it pre-populates the PR body automatically.

```markdown
## Summary
<!-- What does this PR do? Bullet points work well. -->

## Motivation
<!-- Why is this change needed? Link to ticket/issue. -->

## How to Test
<!-- Steps to verify the change works. -->

## Screenshots
<!-- For UI changes, before/after screenshots. -->

## Checklist
- [ ] Tests added or updated
- [ ] Documentation updated
- [ ] No sensitive data introduced
- [ ] Migrations are backwards-compatible
```

---

## Code Review

Code review is not about finding fault — it is a knowledge-sharing exercise that improves code quality, catches bugs early, and distributes understanding across the team.

### Review States

| State | Meaning |
|---|---|
| **Comment** | General feedback with no formal verdict |
| **Approve** | Reviewer is satisfied; ready to merge (subject to other gates) |
| **Request changes** | Reviewer has blockers that must be addressed before merge |

### Inline Comments and Suggestions

**Inline comment** — attached to a specific line or range of lines. Keeps feedback precise:

```
Line 47: This loop runs O(n²) — consider building a lookup map before the loop
         to bring it down to O(n).
```

**GitHub Suggested Change** — instead of describing a fix in words, the reviewer writes the corrected code directly in the comment using a fenced code block. The author can apply it with a single click, which creates a commit automatically.

```suggestion
return users.stream()
    .filter(u -> u.isActive())
    .collect(Collectors.toList());
```

### Review Checklist

Before approving a PR, systematically check:

- **Logic** — does the implementation match the intended behaviour? Are edge cases handled?
- **Tests** — are new code paths covered? Do existing tests still pass?
- **Security** — SQL injection, XSS, authentication bypasses, secrets in code, insecure dependencies
- **Performance** — N+1 queries, unindexed lookups, unbounded loops, memory leaks
- **Style** — consistent with the project's conventions; naming is clear
- **Documentation** — public APIs, non-obvious decisions, and README changes where needed
- **Breaking changes** — API compatibility, migration requirements, deprecation notices

### Responding to Feedback

- **Address everything** — even if you disagree, acknowledge the comment and explain your reasoning
- **Don't just push a commit silently** — if the change is non-trivial, reply to the comment explaining what you did
- **Resolve conversations** only after they are genuinely resolved (not just to clear the UI)
- **Disagree respectfully** — "I considered that, but here's why I kept it this way: ..." is fine

---

## Fork Workflow (Open Source Contribution)

When you do not have write access to a repository (the common case for open source), you contribute via a **fork**: your own copy of the repo where you have full write access.

### Setup

```bash
# 1. Fork on GitHub (click "Fork" button) — creates github.com/YOUR-USERNAME/repo

# 2. Clone YOUR fork
git clone https://github.com/YOUR-USERNAME/repo.git
cd repo

# 3. Add the original repo as "upstream"
git remote add upstream https://github.com/original-owner/repo.git

# Verify remotes
git remote -v
# origin    https://github.com/YOUR-USERNAME/repo.git (fetch)
# origin    https://github.com/YOUR-USERNAME/repo.git (push)
# upstream  https://github.com/original-owner/repo.git (fetch)
# upstream  https://github.com/original-owner/repo.git (push)
```

### Keeping Your Fork in Sync

```bash
# Fetch changes from the original repo
git fetch upstream

# Rebase your local main onto upstream's main
git checkout main
git rebase upstream/main

# Push the updated main to your fork
git push origin main
```

Prefer `rebase` over `merge` here — it keeps your fork's history linear and identical to upstream.

### Contributing a Change

```bash
# Always start from an up-to-date main
git checkout main && git rebase upstream/main

# Create a focused branch
git checkout -b fix/typo-in-contributing-guide

# Make the change
vim CONTRIBUTING.md
git add CONTRIBUTING.md
git commit -m "docs: fix typo in contributing guide"

# Push to YOUR fork
git push origin fix/typo-in-contributing-guide

# Open a PR on GitHub from YOUR fork's branch → upstream's main
# GitHub will show a "Compare & pull request" banner automatically
```

---

## Protected Branches

Protected branches prevent accidental or unauthorised changes to critical branches like `main` and `develop`.

### Common Protection Rules (GitHub)

| Rule | Purpose |
|---|---|
| **Require a pull request before merging** | No direct pushes to the branch |
| **Require approvals (N)** | At least N reviewers must approve |
| **Require status checks to pass** | CI must be green before merge |
| **Require branches to be up to date** | Branch must include latest main before merge |
| **Restrict who can push** | Only admins or named users/teams |
| **Block force pushes** | Prevents history rewriting on the branch |
| **Block deletions** | The branch cannot be deleted |

Configure in **Settings → Branches → Branch protection rules** (GitHub) or **Settings → Repository → Protected branches** (GitLab).

---

## CODEOWNERS

The `CODEOWNERS` file (stored at `.github/CODEOWNERS`, `CODEOWNERS`, or `docs/CODEOWNERS`) defines which teams or individuals automatically become reviewers when their files are touched by a PR.

### Syntax

```
# Global owners — review everything unless a more specific rule matches
*   @org/platform-team

# Backend code requires backend-team approval
/backend/   @org/backend-team

# Frontend code
/frontend/   @org/frontend-team @alice

# Specific file: the security team must review all auth changes
/src/auth/   @org/security-team

# File type: all SQL migrations reviewed by DBA team
*.sql   @org/dba-team

# Docs can be reviewed by anyone on the docs team
/docs/   @org/docs-team
```

Rules are evaluated **bottom-up** (last matching rule wins). When a PR touches `/backend/services/AuthService.java`, only `@org/backend-team` is requested — the global `*` rule does not override it.

---

## GitHub-Specific Features

### GitHub Actions (CI/CD Overview)

GitHub Actions is a built-in CI/CD platform. Workflows are defined as YAML files in `.github/workflows/`.

**Key concepts:**

| Concept | Description |
|---|---|
| **Workflow** | A YAML file that defines automation |
| **Trigger (`on:`)** | What starts the workflow (push, pull_request, schedule, manual) |
| **Job** | A set of steps running on a single runner |
| **Step** | An individual command or action |
| **Action** | A reusable unit from the GitHub Marketplace |

**Example: Basic CI workflow**

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build
```

When this workflow is green, the PR's status check turns green and the branch can be merged (if the protection rule requires it).

### GitHub CLI

The `gh` CLI brings GitHub workflows into the terminal:

```bash
# Create a PR (opens an interactive prompt for title/body)
gh pr create

# Create a PR non-interactively
gh pr create --title "feat: add dark mode" \
             --body "Implements dark mode toggle. Closes #42." \
             --base main

# List open PRs
gh pr list

# Check out a PR locally by number (creates a tracking branch)
gh pr checkout 42

# View PR status (checks, reviewers, comments)
gh pr status

# Merge a PR (squash merge)
gh pr merge 42 --squash --delete-branch

# Create an issue
gh issue create --title "Login button misaligned on mobile" \
                --body "Steps to reproduce: ..."

# Clone a repo
gh repo clone owner/repo
```

### Dependabot

Dependabot automatically opens PRs to update your dependencies. Enable it with:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    assignees:
      - "alice"
```

---

## GitLab-Specific Features

### Merge Requests

GitLab MRs behave identically to GitHub PRs with a few extras:

- **Draft MRs** (prefix title with `Draft:`) — visible but not mergeable until marked ready
- **Merge when pipeline succeeds** — schedule the merge to happen automatically once CI passes
- **Squash commits on merge** — per-MR option, not just global configuration

### GitLab CI/CD (`.gitlab-ci.yml`)

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

test:
  stage: test
  image: node:20
  script:
    - npm ci
    - npm run lint
    - npm test
  only:
    - merge_requests
    - main

build:
  stage: build
  image: node:20
  script:
    - npm run build
  artifacts:
    paths:
      - dist/

deploy-staging:
  stage: deploy
  script:
    - ./deploy.sh staging
  environment:
    name: staging
  only:
    - main
```

Pipelines run automatically on MR creation and on every push, giving reviewers CI feedback directly on the MR page.

### Merge Request Approval Rules

In **Settings → Merge requests → Approval rules**, you can require:
- A minimum number of approvals from any user
- Approval from specific groups (e.g., security team for security-sensitive files)
- Approval from code owners

---

## Git Tags and Releases

### Creating Tags

**Annotated tag** (recommended for releases — includes author, date, and message):

```bash
git tag -a v1.0.0 -m "Release 1.0.0: initial public release"
```

**Lightweight tag** (a simple pointer, no metadata — useful for temporary markers):

```bash
git tag v1.0.0
```

### Pushing Tags

```bash
# Push a single tag
git push origin v1.0.0

# Push all local tags at once
git push origin --tags

# Delete a remote tag (requires force — use carefully)
git push origin --delete v1.0.0-rc1
```

### Semantic Versioning

All releases should follow **Semantic Versioning** (`MAJOR.MINOR.PATCH`):

| Part | Increment when... | Example |
|---|---|---|
| **MAJOR** | You make backwards-incompatible API changes | `1.0.0` → `2.0.0` |
| **MINOR** | You add functionality in a backwards-compatible way | `1.2.0` → `1.3.0` |
| **PATCH** | You make backwards-compatible bug fixes | `1.2.3` → `1.2.4` |

Pre-release suffixes: `v2.0.0-alpha.1`, `v2.0.0-beta.3`, `v2.0.0-rc.1`

### GitHub Releases

A GitHub Release attaches human-readable release notes and downloadable assets to an annotated tag.

```bash
# Create a release from a tag using gh CLI
gh release create v1.0.0 \
  --title "v1.0.0 — Initial Release" \
  --notes "## What's new
- OAuth login support
- Dark mode
- Performance improvements (50% faster page loads)

## Bug fixes
- Fixed null pointer in payment handler

## Upgrade notes
Run \`npm run migrate\` after upgrading." \
  ./dist/app-linux-amd64 \   # attach binary assets
  ./dist/app-darwin-amd64
```

### Generating Version Strings with `git describe`

```bash
git describe --tags
# v1.0.0-3-gabcdef1
# meaning: 3 commits after v1.0.0, current commit is abcdef1
```

This is useful for embedding build-time version strings in binaries or Docker images.

---

## Summary

- A PR/MR is both a merge request and a collaboration surface; invest in clear titles and descriptions
- Small PRs (under 400 lines) are reviewed more quickly and thoroughly
- The fork workflow is the standard way to contribute to open source without repository write access
- Protected branches enforce quality gates: required reviews, passing CI, and up-to-date branches
- CODEOWNERS automates the right reviewer getting assigned to the right files
- GitHub Actions provides CI/CD as YAML workflows; the `gh` CLI brings PR operations into the terminal
- Annotated tags mark releases; semantic versioning (`MAJOR.MINOR.PATCH`) communicates the impact of a release

---

## Knowledge Check

1. What is the difference between a PR "comment", "approve", and "request changes" review state?
2. In the fork workflow, why do you add an `upstream` remote?
3. What does a CODEOWNERS file do when a PR is opened?
4. A release bumps from `v1.4.2` to `v1.5.0`. What kind of change does this signal?
5. What command would you use to apply `git describe` output as a Docker image tag?

---

## Hands-On Exercise

### Part A: Create and Merge a PR on GitHub

```bash
# 1. Fork any public repo (e.g., https://github.com/octocat/Hello-World)
# 2. Clone your fork and add upstream
git clone https://github.com/YOUR-USERNAME/Hello-World.git
cd Hello-World
git remote add upstream https://github.com/octocat/Hello-World.git

# 3. Create a branch and a trivial change
git checkout -b docs/add-contributing-note
echo "## Contributing\nSee CONTRIBUTING.md" >> README.md
git add README.md
git commit -m "docs: add contributing note to README"
git push origin docs/add-contributing-note

# 4. Open a PR via gh CLI
gh pr create --title "docs: add contributing note to README" \
             --body "Adds a brief contributing section to README."

# 5. Review your own PR, then merge it
gh pr merge --squash --delete-branch
```

### Part B: Tag a Release

```bash
# After merging, tag the result
git checkout main && git pull
git tag -a v0.1.0 -m "Release 0.1.0: first contribution"
git push origin v0.1.0

# Create a GitHub Release
gh release create v0.1.0 \
  --title "v0.1.0" \
  --notes "First release: adds contributing note."
```

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./10-branching-strategies.md">← Previous: Branching Strategies</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./12-advanced-git.md">Next: Advanced Git →</a>
</div>
