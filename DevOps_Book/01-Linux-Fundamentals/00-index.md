# Linux Fundamentals — Complete Course Index

> **DevOps Learning Path — Topic 1 of 11**
> Phase 0: Core Foundations | Estimated Duration: 2–3 weeks

---

## Course Overview

Linux is the **operating system of the internet**. Over 96% of the world's top web servers, all major cloud providers (AWS, GCP, Azure), and nearly every DevOps tool runs on Linux. Before you touch Docker, Kubernetes, CI/CD pipelines, or Terraform — you must be fluent in Linux.

This course takes you from **absolute zero** (never touched a terminal) to **professional-grade Linux competency** used daily in production DevOps roles. You will not just memorize commands — you will understand *why* Linux works the way it does, which makes you dangerous in any environment.

**What you will be able to do after this course:**

- Navigate and manage any Linux system entirely from the terminal
- Read, process, and transform logs and data files using text tools
- Manage users, permissions, and file ownership at a production level
- Monitor and control running processes and system resources
- Use networking tools to diagnose and debug connectivity issues
- Write shell scripts to automate repetitive DevOps tasks
- Administer services with systemd and cron
- Apply Linux security hardening fundamentals
- Confidently answer Linux questions in DevOps interviews

---

## Prerequisites

**No prior Linux knowledge required.** However, you should be comfortable with:

- Using a computer (files, folders, clicking around)
- Basic understanding that code/programs exist (you don't need to write code yet)
- Willingness to type commands and experiment

**What to set up before starting:**

| Option | Description | Recommended For |
|--------|-------------|-----------------|
| WSL2 (Windows) | Windows Subsystem for Linux — Ubuntu inside Windows | Windows users |
| Virtual Machine | VirtualBox + Ubuntu 22.04 LTS | Any OS, full isolation |
| Cloud VM | AWS EC2 free tier / Google Cloud free VM | Direct cloud exposure |
| Native Linux | Dual boot or dedicated Linux machine | Best experience |

> **Recommendation:** Use WSL2 on Windows or a cloud VM — these mirror real DevOps environments most closely.

---

## Estimated Learning Timeline

| Week | Focus Areas | Chapters |
|------|-------------|----------|
| Week 1 | Filesystem, navigation, file ops, text viewing | 01–04 |
| Week 2 | Text processing, permissions, processes | 05–07 |
| Week 3 | Networking, scripting, system administration | 08–11 |
| Week 4 | Advanced topics, projects, interview prep | 12–17 |

> **Total: 3–4 weeks** for job-ready Linux proficiency (2–3 hrs/day)

---

## Learning Roadmap

```
BEGINNER                 INTERMEDIATE              ADVANCED
─────────────────────    ──────────────────────    ──────────────────────
✓ Terminal navigation    ✓ Text processing (grep,  ✓ Advanced shell scripting
✓ File/dir commands        awk, sed)               ✓ Performance tuning
✓ Filesystem hierarchy   ✓ Permissions & ownership ✓ System internals
✓ Text viewing           ✓ Process management      ✓ Security hardening
                         ✓ Networking tools         ✓ Systemd & services
                         ✓ Shell scripting basics   ✓ Production operations
```

---

## Milestones

### Milestone 1 — Terminal Confident (End of Week 1)
- [ ] Navigate the entire Linux filesystem using only the terminal
- [ ] Create, copy, move, delete files and directories
- [ ] Read and search file contents
- [ ] Understand the purpose of every major directory (`/etc`, `/var`, `/usr`)

### Milestone 2 — Text & Permissions Master (End of Week 2)
- [ ] Process log files with `grep`, `awk`, `sed`
- [ ] Set correct file permissions and ownership
- [ ] Understand and manage running processes
- [ ] Use pipes and redirects fluently

### Milestone 3 — Scripting & Networking (End of Week 3)
- [ ] Write shell scripts that automate real tasks
- [ ] Debug network issues with `curl`, `ss`, `netstat`
- [ ] Manage users and groups
- [ ] Schedule tasks with cron and manage services with systemd

### Milestone 4 — Production Ready (End of Week 4)
- [ ] Complete all three projects
- [ ] Pass the knowledge check questions in every chapter
- [ ] Confidently answer Linux interview questions

---

## Complete Chapter Index

| # | Chapter | Topics Covered | Level |
|---|---------|----------------|-------|
| [01](01-introduction-and-prerequisites.md) | Introduction & Why Linux | Linux history, distributions, terminal basics | Beginner |
| [02](02-linux-filesystem-hierarchy.md) | Linux Filesystem Hierarchy | FHS, `/etc`, `/var`, `/usr`, `/opt`, `/proc`, `/sys` | Beginner |
| [03](03-file-and-directory-commands.md) | File & Directory Commands | `ls`, `cd`, `pwd`, `mkdir`, `cp`, `mv`, `rm`, `find` | Beginner |
| [04](04-text-viewing-and-editing.md) | Text Viewing & Editing | `cat`, `less`, `head`, `tail`, `nano`, `vim` | Beginner |
| [05](05-text-processing.md) | Text Processing | `grep`, `awk`, `sed`, `cut`, `sort`, `uniq`, `wc` | Intermediate |
| [06](06-permissions-and-ownership.md) | Permissions & Ownership | `chmod`, `chown`, `chgrp`, `umask`, SUID/SGID | Intermediate |
| [07](07-process-management.md) | Process Management | `ps`, `top`, `htop`, `kill`, `jobs`, `nice`, `systemctl` | Intermediate |
| [08](08-networking-tools.md) | Networking Tools | `curl`, `wget`, `ss`, `netstat`, `ping`, `dig`, `ssh` | Intermediate |
| [09](09-shell-scripting.md) | Shell Scripting | Variables, conditionals, loops, functions, scripts | Intermediate |
| [10](10-users-and-system-administration.md) | Users & System Administration | Users, groups, sudo, cron, systemd, logs | Intermediate |
| [11](11-intermediate-concepts.md) | Pipes, Redirects & Environment | `|`, `>`, `>>`, env vars, aliases, PATH, dotfiles | Intermediate |
| [12](12-advanced-concepts.md) | Advanced Linux Concepts | `find`+`xargs`, `strace`, `/proc`, performance tools | Advanced |
| [13](13-best-practices.md) | Linux Best Practices for DevOps | Security, scripting standards, production habits | Advanced |
| [14](14-common-mistakes-and-pitfalls.md) | Common Mistakes & Pitfalls | `rm -rf`, permission errors, process zombies | All Levels |
| [15](15-projects.md) | Hands-On Projects | 4 real-world projects from beginner to advanced | All Levels |
| [16](16-interview-preparation.md) | Interview Preparation | Top questions, scenarios, system design | Advanced |
| [17](17-course-summary.md) | Course Summary & Next Steps | Review, resources, DevOps path continuation | All Levels |

---

## DevOps Roadmap — Course Series

| # | Course | Status |
|---|--------|--------|
| 1 | [Linux Fundamentals](../01-Linux-Fundamentals/00-index.md) | ✅ You are here |
| 2 | [Networking Basics](../02-Networking-Basics/00-index.md) | ✅ Available |
| 3 | Git & Version Control | Coming soon |
| 4 | Docker | Coming soon |
| 5 | CI/CD Pipelines | Coming soon |
| 6 | Cloud Fundamentals (AWS) | Coming soon |
| 7 | Infrastructure as Code (Terraform) | Coming soon |
| 8 | Kubernetes Basics | Coming soon |
| 9 | Advanced Kubernetes | Coming soon |
| 10 | Monitoring & Logging | Coming soon |
| 11 | Security (DevSecOps) | Coming soon |

---

## Recommended Resources

### Books
- *The Linux Command Line* by William Shotts (free at linuxcommand.org)
- *Linux Pocket Guide* by Daniel J. Barrett (quick reference)
- *How Linux Works* by Brian Ward (internals deep dive)

### Interactive Practice
- **OverTheWire: Bandit** — gamified Linux challenges
- **Linux Journey** (linuxjourney.com) — structured interactive lessons
- **Explain Shell** (explainshell.com) — paste any command, get an explanation

### Cheat Sheets
- `tldr` command — simplified man pages (`sudo apt install tldr`)
- `man <command>` — built-in manual pages
- `<command> --help` — quick flag reference

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <span></span>
  <a href="./00-index.md">🏠 Index</a>
  <a href="01-introduction-and-prerequisites.md">Next: Introduction & Why Linux →</a>
</div>
