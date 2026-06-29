# Git & Version Control — Complete Course Index

## Course Overview

Git is not just a tool — it is the backbone of every modern DevOps workflow. From tracking a single developer's local changes to coordinating thousands of commits across distributed teams, Git enables collaboration, rollback, automation, and continuous delivery at scale. This course takes you from Git fundamentals all the way through advanced internals, branching strategies, GitHub/GitLab workflows, and GitOps principles. By the end you will think in commits, speak in diffs, and design workflows that scale with your team.

---

## Prerequisites

Before starting this course, ensure you are comfortable with the following:

| Prerequisite | Course | Status |
|---|---|---|
| Linux command line basics (navigation, file operations, permissions) | Linux Fundamentals | ✅ Complete |
| Networking concepts (SSH, HTTPS, DNS) | Networking Basics | ✅ Complete |
| Terminal/shell comfort — running commands, reading output | — | Required |
| A text editor or IDE installed (VS Code, Vim, Nano) | — | Required |

---

## 4-Week Learning Timeline

| Week | Focus | Chapters | Goal |
|---|---|---|---|
| Week 1 | Git Basics & Internals | 01 – 03 | Understand what Git is, how it works internally, and master day-to-day commands |
| Week 2 | Branching, Remotes & Merging | 04 – 07 | Create and manage branches, work with remote repositories, merge and rebase confidently |
| Week 3 | Advanced Operations & Workflows | 08 – 12 | Stash, cherry-pick, undo changes, adopt professional branching strategies, use GitHub/GitLab |
| Week 4 | Best Practices, Projects & Interview Prep | 13 – 17 | Write clean commit histories, avoid common pitfalls, complete hands-on projects, ace interviews |

---

## Learning Milestones

### Milestone 1 — Beginner
> Clone a repository, make changes, stage them, commit, and push to a remote.

You will have achieved this milestone when you can:
- Initialize a Git repository and understand its structure
- Configure Git with your identity and preferred settings
- Perform the full edit → stage → commit → push cycle
- Read `git log` output and understand the commit graph
- Write a meaningful `.gitignore` file

### Milestone 2 — Intermediate
> Design and implement a branching strategy, open pull requests, and review others' code.

You will have achieved this milestone when you can:
- Create, switch, and delete branches confidently
- Merge branches and resolve conflicts without panic
- Rebase a feature branch onto the latest `main`
- Open, review, and merge pull requests on GitHub or GitLab
- Describe the difference between Git Flow, trunk-based development, and GitHub Flow

### Milestone 3 — Advanced
> Use Git hooks, rewrite history safely, and implement GitOps principles in a CI/CD pipeline.

You will have achieved this milestone when you can:
- Write custom Git hooks (pre-commit, commit-msg, pre-push)
- Perform interactive rebases to clean up commit history
- Use `git bisect` to locate regressions
- Understand and implement GitOps with Git as the source of truth
- Recover from any state — detached HEAD, accidental force push, lost commits — using `reflog`

---

## Chapter Index

| # | File | Title | Topics Covered |
|---|---|---|---|
| 01 | [01-introduction-to-git.md](./01-introduction-to-git.md) | Introduction to Git & Version Control | VCS evolution, Git history, key concepts, installation, first-time setup |
| 02 | [02-git-internals.md](./02-git-internals.md) | Git Internals: How Git Really Works | .git directory, object model (blob/tree/commit/tag), SHA-1, DAG, HEAD, index, packfiles |
| 03 | [03-essential-commands.md](./03-essential-commands.md) | Essential Git Commands | init, clone, status, add, diff, commit, log, show, rm, mv, .gitignore |
| 04 | [04-branching.md](./04-branching.md) | Branching in Git | Creating/switching/deleting branches, tracking branches, HEAD detachment |
| 05 | [05-remote-repositories.md](./05-remote-repositories.md) | Remote Repositories | remote add/remove, fetch, pull, push, tracking, upstream configuration |
| 06 | [06-merging.md](./06-merging.md) | Merging Strategies | Fast-forward, 3-way merge, merge commits, conflict resolution, --no-ff |
| 07 | [07-rebasing.md](./07-rebasing.md) | Rebasing | Linear history, interactive rebase, squash, fixup, rebase vs merge |
| 08 | [08-stash-and-cherry-pick.md](./08-stash-and-cherry-pick.md) | Stash & Cherry-Pick | git stash, stash list/apply/pop/drop, cherry-pick single and range |
| 09 | [09-undoing-changes.md](./09-undoing-changes.md) | Undoing Changes | checkout, restore, reset (soft/mixed/hard), revert, clean, reflog recovery |
| 10 | [10-branching-strategies.md](./10-branching-strategies.md) | Branching Strategies | Git Flow, GitHub Flow, trunk-based development, release branches, hotfixes |
| 11 | [11-github-gitlab-workflows.md](./11-github-gitlab-workflows.md) | GitHub & GitLab Workflows | Forks, PRs, MRs, code review, CI/CD integration, protected branches |
| 12 | [12-advanced-git.md](./12-advanced-git.md) | Advanced Git | Hooks, submodules, worktrees, bisect, rerere, bundle, filter-branch, sparse checkout |
| 13 | [13-best-practices.md](./13-best-practices.md) | Git Best Practices | Commit message conventions, atomic commits, branch naming, tagging, signing |
| 14 | [14-common-mistakes.md](./14-common-mistakes.md) | Common Git Mistakes & Fixes | Force push disasters, secrets in history, merge conflicts mishandled, detached HEAD panic |
| 15 | [15-projects.md](./15-projects.md) | Hands-On Projects | Project 1: team simulation, Project 2: OSS contribution workflow, Project 3: CI/CD pipeline |
| 16 | [16-interview-preparation.md](./16-interview-preparation.md) | Interview Preparation | Top 40 Git interview questions with detailed answers, scenario-based questions |
| 17 | [17-course-summary.md](./17-course-summary.md) | Course Summary & Next Steps | Key takeaways, cheat sheet, DevOps roadmap continuation |

---

## DevOps Roadmap — Course Series

This course is part of a structured DevOps learning path. Each topic builds on the previous ones.

| # | Topic | Index Link | Status |
|---|---|---|---|
| 1 | Linux Fundamentals | [../01-Linux-Fundamentals/00-index.md](../01-Linux-Fundamentals/00-index.md) | ✅ Complete |
| 2 | Networking Basics | [../02-Networking-Basics/00-index.md](../02-Networking-Basics/00-index.md) | ✅ Complete |
| 3 | Git & Version Control | *(current)* | 📍 You are here |
| 4 | Docker & Containers | ../04-Docker/00-index.md | 🔜 Coming Soon |
| 5 | CI/CD Pipelines | ../CI-CD/00-index.md | 🔜 Coming Soon |
| 6 | Kubernetes | ../Kubernetes/00-index.md | 🔜 Coming Soon |
| 7 | Infrastructure as Code (Terraform) | ../Terraform/00-index.md | 🔜 Coming Soon |
| 8 | Cloud Platforms (AWS/GCP/Azure) | ../Cloud/00-index.md | 🔜 Coming Soon |
| 9 | Monitoring & Observability | ../Monitoring/00-index.md | 🔜 Coming Soon |
| 10 | Security & DevSecOps | ../DevSecOps/00-index.md | 🔜 Coming Soon |
| 11 | GitOps & ArgoCD | ../GitOps/00-index.md | 🔜 Coming Soon |

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <span></span>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./01-introduction-to-git.md">Next: Introduction to Git →</a>
</div>
