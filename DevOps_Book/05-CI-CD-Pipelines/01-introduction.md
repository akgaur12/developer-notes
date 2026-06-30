# Chapter 1 — Introduction to CI/CD

## Learning Objectives

By the end of this chapter you will be able to:

- Explain the problem CI/CD solves in plain language
- Distinguish between Continuous Integration, Continuous Delivery, and Continuous Deployment
- Name the major CI/CD tools and when to use each
- Read and understand a basic GitHub Actions workflow file

---

## 1.1 The Problem CI/CD Solves

### Life Before CI/CD

In traditional software teams, developers would work in isolation on long-lived feature branches — sometimes for weeks or months. When it came time to merge, the result was painful: conflicting changes, broken builds, and frantic debugging sessions that could last days. This was called **integration hell**.

The pain didn't stop at merging. Testing was manual, slow, and inconsistent. One developer's machine had a slightly different environment, so "it works on my machine" became a running joke with serious consequences.

Deployments were treated like events — scheduled for Friday afternoons (or late nights), requiring all hands on deck, lengthy runbooks, and frequent emergency rollbacks. Failure was common and expensive.

**The core problems:**

- Bugs were caught in production instead of in pull requests
- Long feedback loops meant developers had forgotten the context of a bug by the time it was found
- Manual, inconsistent processes introduced human error at every step
- Fear of deployment slowed down the entire organization

### The Cost of Not Automating

Research from the DORA (DevOps Research and Assessment) program has tracked thousands of organizations over years. Teams without CI/CD consistently show:

- Deployment frequency: once per quarter or less
- Lead time for a change to reach production: weeks to months
- Change failure rate: 15–20% of deployments cause incidents
- Mean time to recover from an incident: hours to days

These are not just developer annoyances — they translate directly to business risk and lost revenue.

---

## 1.2 What Is CI/CD?

CI/CD is a set of practices that automate the path from a developer's code commit to running software in production. The three terms are related but distinct:

### Continuous Integration (CI)

Every code commit triggers an automated sequence: the codebase is checked out on a clean machine, dependencies are installed, the code is built, and the full test suite runs. If anything fails, the developer gets notified immediately — while the change is still fresh in their mind.

Key principle: **merge small changes frequently** (at least daily) so that integration problems are caught when they are small and easy to fix.

### Continuous Delivery (CD)

Extends CI by ensuring that every build that passes all tests is **releasable to production at any time**. The act of releasing may still require a human to click a button or approve a deployment, but the system is always ready.

Key principle: the release decision is a business decision, not a technical one.

### Continuous Deployment

Goes one step further: every passing build is **automatically deployed to production** with no human intervention. This requires extremely high confidence in your automated tests and monitoring.

Key principle: if you trust your tests enough, remove the human from the loop entirely.

### Analogy

Think of CI/CD like a factory assembly line. Each station on the line does one job and performs a quality check before passing the product to the next station. If a defect is caught at station 2, it costs almost nothing to fix. If it makes it to the customer, the cost — in recalls, reputation, and support — is enormous.

CI/CD applies that same thinking to software.

```
Developer → git push → [CI] Build → Test → Scan → [CD] Deploy to Staging → Approve → Deploy to Prod
```

---

## 1.3 The CI/CD Value Chain

The DORA research quantifies what elite engineering teams achieve with mature CI/CD:

| Metric | Without CI/CD | With CI/CD |
|--------|--------------|------------|
| Deployment frequency | Once per quarter | Multiple times per day |
| Lead time for changes | Weeks to months | Hours |
| Change failure rate | 15–20% | 0–5% |
| Mean time to recovery | Hours to days | Minutes |

These improvements compound. When deployments are small and frequent, each one carries less risk. When the feedback loop is short, developers catch bugs before they become incidents. When recovery is fast, the cost of a failure drops dramatically — which makes teams more willing to deploy, which drives further improvement.

---

## 1.4 Key CI/CD Tools

The CI/CD landscape has many tools. Here are the most important ones:

| Tool | Type | Strengths | Typical Use |
|------|------|-----------|-------------|
| GitHub Actions | Cloud (hosted) | Tight GitHub integration, huge marketplace, free for public repos | Open source projects, GitHub-hosted repos |
| GitLab CI | Cloud + Self-hosted | Full DevOps platform, free self-hosted tier, built-in registry/security | Enterprise, GitLab-hosted repos |
| Jenkins | Self-hosted | Extremely customizable, thousands of plugins, mature ecosystem | On-premise requirements, legacy systems |
| CircleCI | Cloud (hosted) | Fast, Docker-native, good orbs (reusable config) | Startups, Docker-heavy workflows |
| Tekton | Kubernetes-native | Cloud-native primitives, CRD-based, highly composable | K8s-heavy environments |
| Argo Workflows | Kubernetes-native | DAG-based task orchestration, great UI | Data pipelines, complex K8s workflows |

### Which should you learn?

- **GitHub Actions** is the best starting point for most developers. It is free for public repos, has the largest action marketplace, and is deeply integrated with GitHub — where most open source and SaaS development lives.
- **GitLab CI** is essential if your organization uses GitLab, and it is the leading choice for companies that want a fully self-hosted DevOps platform.
- **Jenkins** remains dominant in enterprises with on-premise infrastructure and large amounts of legacy configuration.

This course provides deep coverage of GitHub Actions, solid coverage of GitLab CI and Jenkins, and platform-agnostic design principles you can apply anywhere.

---

## 1.5 What This Course Focuses On

- **Depth on GitHub Actions** — the most widely adopted modern CI/CD tool
- **Practical coverage of GitLab CI and Jenkins** — both remain essential in professional environments
- **Platform-agnostic pipeline design** — concepts, patterns, and tradeoffs that transfer across tools
- **Real-world workflows** — every example is drawn from actual production use cases, not toy demos

---

## 1.6 A Pipeline in 5 Minutes

The best way to understand CI/CD is to see a complete example. Here is a minimal but real GitHub Actions workflow for a Node.js project:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test
```

### Line-by-line walkthrough

**`name: CI`** — The display name that appears in the GitHub UI under the Actions tab.

**`on:`** — Declares what events trigger this workflow. This workflow runs on:
- Any push to the `main` branch
- Any pull request targeting the `main` branch

**`jobs:`** — A workflow contains one or more jobs. Jobs run in parallel by default unless you declare dependencies between them.

**`test:`** — The job identifier (must be unique within the workflow, kebab-case by convention).

**`runs-on: ubuntu-latest`** — Specifies the runner — the machine that will execute this job. `ubuntu-latest` is a GitHub-hosted virtual machine running the latest Ubuntu LTS. A fresh VM is provisioned for every job run.

**`steps:`** — An ordered list of steps that run sequentially inside the job.

**`- uses: actions/checkout@v4`** — Runs the official `checkout` action, which clones your repository into the runner so subsequent steps can access your code. The `@v4` pins the version.

**`- uses: actions/setup-node@v4`** — Installs Node.js on the runner. The `with:` block passes configuration — in this case, the Node version.

**`- run: npm ci`** — Runs a shell command directly. `npm ci` installs dependencies from `package-lock.json` exactly (faster and more reproducible than `npm install`).

**`- run: npm test`** — Runs the project's test suite. If this command exits with a non-zero status code, the step (and the job) fails, the PR is blocked, and the developer is notified.

That's it. From this simple foundation, you can build arbitrarily sophisticated pipelines.

---

## Summary

- CI/CD automates the path from code commit to production, replacing manual, error-prone processes
- **CI** catches bugs early by running builds and tests on every commit
- **CD** ensures every passing build is always releasable; Continuous Deployment removes the human from the final release step
- The measurable impact is enormous: faster deployments, fewer failures, faster recovery
- GitHub Actions is the primary tool in this course; GitLab CI and Jenkins are also covered
- A complete CI pipeline is a YAML file in your repository — it checks out code, installs dependencies, and runs tests

---

## Knowledge Check

1. What is "integration hell" and how does CI address it?
2. What is the difference between Continuous Delivery and Continuous Deployment?
3. A developer says "deployments are scary so we only deploy once a month." How does CI/CD change this calculus?
4. In the example workflow, what happens if `npm test` exits with code 1?
5. Why does `runs-on: ubuntu-latest` provision a fresh VM for every job run instead of reusing one?

---

## Hands-On Exercise

1. Fork any public Node.js repository on GitHub (e.g., a simple todo app or starter project)
2. Create the directory `.github/workflows/` in the repository
3. Add a file named `ci.yml` with exactly the workflow shown in section 1.6
4. Commit and push the file to the `main` branch
5. Navigate to the **Actions** tab in your forked repository
6. Watch the workflow run — observe each step executing in real time
7. Make a small deliberate change (e.g., rename a function without updating its test) and push again
8. Watch the workflow fail and identify exactly which step failed and why

**Goal:** Experience the CI feedback loop firsthand — commit, automatic trigger, pass or fail result.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./00-index.md">← Previous: Index</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./02-core-concepts.md">Next: Core Concepts & Architecture →</a>
</div>
