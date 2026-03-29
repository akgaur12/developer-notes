# 🚀 GitHub CLI (`gh`) – Complete Guide

The GitHub CLI (`gh`) is an official command-line tool that lets you interact with GitHub directly from your terminal. It simplifies workflows like creating repositories, managing pull requests, issues, releases, and even running GitHub Actions.

---

## 📦 Installation

### Linux
```bash
sudo apt install gh        # Debian/Ubuntu
sudo dnf install gh        # Fedora
brew install gh            # Linuxbrew
```

### macOS
```bash
brew install gh
```

### Windows
```powershell
winget install --id GitHub.cli
```

Verify installation:
```bash
gh --version
```

---

## 🔐 Authentication

Login to GitHub:
```bash
gh auth login
```

Check auth status:
```bash
gh auth status
```

Logout:
```bash
gh auth logout
```

---

## 🏗 Repository Management

Create a new repo:
```bash
gh repo create

    or
    
gh repo create my-repo
```

Create & clone:
```bash
gh repo create my-repo --public --clone
```

Clone an existing repo:
```bash
gh repo clone owner/repo
```

View repo:
```bash
gh repo view
```

Fork a repo:
```bash
gh repo fork owner/repo
```

Delete a repo:
```bash
gh repo delete owner/repo
```

---

## 🔀 Pull Requests (PR)

Create PR:
```bash
gh pr create
```

List PRs:
```bash
gh pr list
```

View PR:
```bash
gh pr view 23
```

Checkout PR:
```bash
gh pr checkout 23
```

Merge PR:
```bash
gh pr merge 23
```

---

## 🐛 Issues

Create issue:
```bash
gh issue create
```

List issues:
```bash
gh issue list
```

View issue:
```bash
gh issue view 12
```

Close issue:
```bash
gh issue close 12
```

---

## 📦 Releases

Create release:
```bash
gh release create v1.0.0
```

List releases:
```bash
gh release list
```

Download release:
```bash
gh release download v1.0.0
```

Delete release:
```bash
gh release delete v1.0.0
```

---

## ⚙️ GitHub Actions

List workflows:
```bash
gh workflow list
```

Run workflow:
```bash
gh workflow run build.yml
```

List runs:
```bash
gh run list
```

Watch run:
```bash
gh run watch <run-id>
```

---

## 🌐 Open in Browser

```bash
gh repo view --web
gh pr view --web
gh issue view --web
```

---

## 🔧 Config & Aliases

View config:
```bash
gh config list
```

Set config:
```bash
gh config set editor vim
```

Create alias:
```bash
gh alias set prc "pr create"
```

---

## 🧩 Extensions

List extensions:
```bash
gh extension list
```

Install extension:
```bash
gh extension install owner/extension
```

Remove extension:
```bash
gh extension remove owner/extension
```

---

## 📌 TL;DR

`gh` brings GitHub into your terminal. Faster PRs, easier issues, simple releases, and complete GitHub Actions control.
