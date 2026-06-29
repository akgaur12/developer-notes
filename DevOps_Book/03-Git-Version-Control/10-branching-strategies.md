# Chapter 10 — Branching Strategies

## Learning Objectives

By the end of this chapter you will be able to:

- Explain why a documented branching strategy is essential for team productivity
- Describe the core mechanics of GitFlow, GitHub Flow, and Trunk-Based Development
- Select the right strategy given your team size, release cadence, and delivery model
- Apply consistent branch naming conventions on real projects
- Use the `git-flow` CLI extension to automate the GitFlow lifecycle

**Prerequisites:** Chapter 09 — Undoing Changes

---

## Why Branching Strategy Matters

Without a shared agreement on how branches are created, merged, and deleted, teams quickly run into:

- **Merge conflicts** — long-lived branches diverge so far from the main line that integration becomes days of painful work
- **Unclear ownership** — nobody knows whether a branch is active, abandoned, or ready to ship
- **Broken CI/CD pipelines** — builds fail because untested code lands in the wrong place at the wrong time
- **Release chaos** — it is impossible to identify what is in a release when dozens of half-finished branches are involved

A branching strategy answers four questions for every commit:

1. Where do I start my work?
2. Where do I merge my work when it is done?
3. How does code get to production?
4. How do I fix a production emergency without disrupting in-flight work?

---

## GitFlow

GitFlow was introduced by Vincent Driessen in 2010 and became one of the most widely adopted branching models in the industry. It is highly structured and suits teams that ship versioned releases on a defined schedule.

### Branch Model

| Branch | Purpose | Lifetime |
|---|---|---|
| `main` | Production-ready code; every commit is a release | Permanent |
| `develop` | Integration branch; latest delivered development changes | Permanent |
| `feature/*` | New functionality | Short (per feature) |
| `release/*` | Release preparation and stabilisation | Short (per release) |
| `hotfix/*` | Urgent production bug fixes | Very short |

### Lifecycle Diagram

```
main      ──●────────────────────────────────●────────●──────────────▶
             │ tag v0.1                       │ tag v1.0│ tag v1.0.1
             │                               │         │
develop   ───●────●──────────────────────────●────●────●──────────▶
              \   │                          │   /
feature/*      ●──● (merge --no-ff back      │  /
                    to develop)              │ /
                                            │/
release/*                   ●──────────────●
                             (stabilise, bump version)
                                                  \
hotfix/*                                           ●──●
                                                      (fix, tag, merge to main + develop)
```

### Feature Development

```bash
# Start a feature (from develop)
git checkout develop
git checkout -b feature/JIRA-123-add-oauth-login

# ... do work, commit ...

# Finish: merge back to develop (always use --no-ff to preserve history)
git checkout develop
git merge --no-ff feature/JIRA-123-add-oauth-login
git branch -d feature/JIRA-123-add-oauth-login
```

The `--no-ff` flag creates a merge commit even when a fast-forward is possible. This keeps the full context of the feature visible in the log.

### Release Preparation

```bash
# Cut a release branch from develop
git checkout develop
git checkout -b release/v1.0.0

# Bump version number, fix minor bugs, update changelog
git commit -am "chore: bump version to 1.0.0"

# Finish the release
git checkout main
git merge --no-ff release/v1.0.0
git tag -a v1.0.0 -m "Release 1.0.0"

# Back-merge to develop so it gets the release fixes
git checkout develop
git merge --no-ff release/v1.0.0
git branch -d release/v1.0.0
```

### Hotfix

```bash
# Branch from main (production)
git checkout main
git checkout -b hotfix/critical-xss-vulnerability

# Fix the bug
git commit -am "fix: sanitise user input in comment renderer"

# Merge to main and tag
git checkout main
git merge --no-ff hotfix/critical-xss-vulnerability
git tag -a v1.0.1 -m "Hotfix 1.0.1"

# Back-merge to develop
git checkout develop
git merge --no-ff hotfix/critical-xss-vulnerability
git branch -d hotfix/critical-xss-vulnerability
```

### git-flow CLI Extension

The `git-flow` extension wraps all of the above into simple commands:

```bash
# Install (macOS)
brew install git-flow-avh

# Initialise a repo with GitFlow structure
git flow init        # prompts for branch names; press Enter to accept defaults

# Feature workflow
git flow feature start JIRA-123-add-oauth-login
# ... work ...
git flow feature finish JIRA-123-add-oauth-login   # merges to develop, deletes branch

# Release workflow
git flow release start v1.0.0
git flow release finish v1.0.0   # merges to main + develop, creates tag

# Hotfix workflow
git flow hotfix start critical-xss-vulnerability
git flow hotfix finish critical-xss-vulnerability
```

### Pros and Cons

**Pros:**
- Very structured; every branch has a defined role
- Supports parallel release versions and hotfixes cleanly
- Well understood by large enterprise teams
- Strong tooling support

**Cons:**
- Complex; five branch types to keep track of
- Long-lived `feature` branches cause integration pain
- Slow feedback loop — code takes a long time to reach production
- Poor fit for continuous delivery where you deploy multiple times per day

---

## GitHub Flow

GitHub Flow, popularised by GitHub in 2011, is a minimal model built around one rule: **`main` is always deployable**.

### Branch Model

Only two types of branches exist:

- `main` — always production-ready
- Short-lived feature/fix branches — everything else

### Workflow

```
1. Pull latest main
2. Create a descriptive branch
3. Commit early and often
4. Open a Pull Request as soon as you have something to discuss
5. CI runs; team reviews
6. Address feedback; CI stays green
7. Merge PR to main
8. Deploy main to production
9. Delete the branch
```

### Lifecycle Diagram

```
main   ──●──────────────────●──────────────────●──────────▶
          \                /                  /
feature    ●──●──●──●──●──●                  /
                                            /
bugfix                    ●──●──●──●──●────●
```

### Example Commands

```bash
git checkout main && git pull origin main
git checkout -b feature/JIRA-789-dark-mode

# ... commit work ...

git push -u origin feature/JIRA-789-dark-mode
# Open PR on GitHub
# After approval and green CI:
# Merge PR via GitHub UI (squash, rebase, or merge commit)
git branch -d feature/JIRA-789-dark-mode
```

### Pros and Cons

**Pros:**
- Simple — only two branch types
- Integrates perfectly with CI/CD pipelines
- Short feedback loops; code ships quickly
- Easy to on-board new developers

**Cons:**
- Requires strong automated testing (main must always be green)
- Less structure for managing multiple concurrent release versions
- Not ideal for apps that cannot deploy continuously (e.g., installed software)

**Best for:** web applications, SaaS products, continuous deployment teams.

---

## Trunk-Based Development (TBD)

Trunk-Based Development is the model used by Google, Facebook, and Netflix at scale. The core principle: **everyone integrates to the trunk (main) continuously**, ideally multiple times per day.

### Rules

1. There is one shared trunk — `main` (or `trunk`)
2. Feature branches exist but live for **less than one day** (ideally hours)
3. Incomplete features ship behind **feature flags** — code is in production but not yet visible
4. The trunk must always be in a releasable state
5. No long-lived branches of any kind

### Lifecycle Diagram

```
        Dev A   Dev B   Dev C   Dev D
           \     |       |      /
main ──●────●────●───────●─────●────●────●──▶
             ↑           ↑
         small commit   small commit
         (feature flag  (feature flag
          off)           on — feature ships)
```

### Feature Flags

```python
# Code ships to production; feature is hidden until flag is enabled
if feature_flags.is_enabled("new-checkout-flow", user_id):
    return new_checkout_flow(cart)
else:
    return legacy_checkout_flow(cart)
```

Feature flag services: LaunchDarkly, Unleash, Flagsmith, Flipt.

### Short-Lived Branch Variant (Scaled TBD)

For teams that are not yet committing directly to trunk, Scaled TBD allows short-lived branches of **one to two days maximum**:

```bash
git checkout -b feat/add-payment-retry   # branch lives < 2 days
# ... a few focused commits ...
git push origin feat/add-payment-retry
# PR reviewed same day, merged, branch deleted
```

### Pros and Cons

**Pros:**
- True continuous integration — no integration hell
- Fastest possible feedback loops
- Forces small, reversible changes
- Scales to thousands of engineers (proven at Google/Meta)

**Cons:**
- Requires high discipline and trust in the team
- Needs mature CI (fast, comprehensive test suites)
- Needs feature flag infrastructure for incomplete work
- Cultural shift for teams used to long feature branches

---

## Release Branches

When your product has multiple supported versions (e.g., enterprise software or an open source library), release branches let you maintain each version independently.

```bash
# At release time, cut a branch from main
git checkout -b release/1.x main
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin release/1.x --tags

# Bug found in v1.x — fix on main first, then cherry-pick
git checkout main
git commit -m "fix: resolve null pointer in login handler"

git checkout release/1.x
git cherry-pick <commit-hash>
git tag -a v1.0.1 -m "Release 1.0.1"
git push origin release/1.x --tags
```

Tags mark the actual releases; the branch accumulates only backported bug fixes.

---

## Strategy Comparison

| Dimension | GitFlow | GitHub Flow | Trunk-Based Dev |
|---|---|---|---|
| **Complexity** | High (5 branch types) | Low (2 types) | Very low (1 branch) |
| **CD readiness** | Poor | Good | Excellent |
| **Suitable team size** | Any | Small–large | Any (proven at massive scale) |
| **Release frequency** | Scheduled / infrequent | Continuous | Multiple times per day |
| **Feature isolation** | Strong (feature branches) | Moderate | Feature flags |
| **Learning curve** | Steep | Gentle | Moderate (flags needed) |
| **Best for** | Versioned products | Web/SaaS apps | FAANG-style engineering |

---

## Choosing a Strategy

| Situation | Recommended Strategy |
|---|---|
| Startup or small team | GitHub Flow or TBD |
| Open source library | GitHub Flow (PRs from forks) |
| Enterprise product with versioned releases | GitFlow or release branches |
| High-velocity / FAANG-style engineering | Trunk-Based Development |
| Mobile app (app store gating) | GitFlow or release branches |
| Microservices with independent deploy | GitHub Flow or TBD per service |

---

## Branch Naming Conventions

Consistent names make branch purpose obvious at a glance and enable automation (e.g., auto-close a JIRA ticket when a branch merges).

| Prefix | Purpose | Example |
|---|---|---|
| `feature/` | New functionality | `feature/JIRA-123-add-oauth-login` |
| `bugfix/` | Non-urgent bug fix | `bugfix/JIRA-456-fix-null-pointer` |
| `hotfix/` | Urgent production fix | `hotfix/critical-xss-vulnerability` |
| `release/` | Release preparation | `release/v2.1.0` |
| `chore/` | Maintenance, dependencies, tooling | `chore/update-dependencies` |
| `docs/` | Documentation only | `docs/improve-api-readme` |
| `refactor/` | Code restructuring without behaviour change | `refactor/extract-auth-service` |
| `experiment/` | Spike / proof of concept | `experiment/graphql-migration` |

**Naming rules:**
- Use lowercase and hyphens — no spaces, no underscores
- Keep names short but descriptive (under 60 characters)
- Include a ticket number when one exists
- Use imperative verbs: `add`, `fix`, `remove`, `update`, not `added` or `adding`

---

## Summary

- A branching strategy is a team contract that governs how code moves from idea to production
- **GitFlow** is structured and powerful but complex; best for versioned product releases
- **GitHub Flow** is simple and CI/CD friendly; best for web/SaaS teams
- **Trunk-Based Development** maximises speed and true continuous integration; requires discipline and feature flags
- Release branches handle long-term maintenance of multiple versions
- Consistent naming conventions reduce cognitive overhead and enable automation

---

## Knowledge Check

1. In GitFlow, which branch serves as the integration branch between features and releases?
2. What does `--no-ff` guarantee when merging a feature branch?
3. Why does Trunk-Based Development require feature flags?
4. You are building a SaaS app that deploys to production 10 times per day. Which strategy fits best?
5. What is the difference between a `hotfix/` and a `bugfix/` branch?

---

## Hands-On Exercise

Implement a full GitFlow lifecycle on a sample repository:

```bash
# Step 1: Create a new repo and initialise GitFlow
mkdir gitflow-demo && cd gitflow-demo
git init
git commit --allow-empty -m "chore: initial commit"
git flow init    # accept all defaults (main/develop/feature/release/hotfix)

# Step 2: Create and finish a feature
git flow feature start JIRA-001-user-registration
echo "# User registration module" > registration.md
git add registration.md
git commit -m "feat: add user registration placeholder"
git flow feature finish JIRA-001-user-registration

# Step 3: Create and finish a release
git flow release start v1.0.0
echo "v1.0.0" > VERSION
git add VERSION
git commit -m "chore: bump version to 1.0.0"
git flow release finish v1.0.0    # enter a tag message when prompted

# Step 4: Simulate a hotfix
git checkout main
git flow hotfix start fix-typo-in-readme
echo "# App — corrected" > README.md
git add README.md
git commit -m "fix: correct typo in README"
git flow hotfix finish fix-typo-in-readme

# Verify the graph
git log --oneline --graph --all
```

Observe how `main`, `develop`, and the tags look in the graph output.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./09-undoing-changes.md">← Previous: Undoing Changes</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./11-github-gitlab-workflows.md">Next: GitHub & GitLab Workflows →</a>
</div>
