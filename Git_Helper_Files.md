# Git Special Helper Files 📁

This document explains **special helper files used by Git** and what they do, in a simple and practical way.

---

## 1. `.gitignore` 🚫

**Purpose:** Tells Git which files/folders to ignore.

### Common use cases

* Environment files
* Build artifacts
* Cache files

```gitignore
.env
__pycache__/
*.log
.venv/
```

---

## 2. `.gitattributes` 📖

**Purpose:** Tells Git **how to treat files** (text, binary, merge rules, line endings).

### Example

```gitattributes
* text=auto
*.sh text eol=lf
*.png binary
```

---

## 3. `.gitkeep` 📂

**Purpose:** Keeps empty folders in Git.

Git does not track empty directories by default.

```
logs/.gitkeep
```

---

## 4. `.gitmodules` 🧩

**Purpose:** Tracks Git submodules (repos inside repos).

Auto-created when you add a submodule.

```ini
[submodule "libs/common"]
	path = libs/common
	url = https://github.com/org/common.git
```

---

## 5. `.gitconfig` ⚙️

**Purpose:** Stores Git user configuration (usually global, not in repo).

```ini
[user]
  name = Akash Gaur
  email = akash@example.com
```

---

## 6. `.github/` 🤖

**Purpose:** GitHub-specific automation and templates.

```
.github/
 ├── workflows/        # CI/CD pipelines
 ├── ISSUE_TEMPLATE/  # Issue templates
 └── PULL_REQUEST_TEMPLATE.md
```

---

## 7. `README.md` 📘

**Purpose:** Explains the project.

Usually contains:

* What the project does
* How to run it
* Examples

---

## 8. `LICENSE` ⚖️

**Purpose:** Defines how others can use your code.

Common licenses:

* MIT
* Apache 2.0
* GPL

---

## 9. `CODEOWNERS` 👥

**Purpose:** Automatically request reviews from owners.

Location: `.github/CODEOWNERS`

```
/backend/  @akash
```

---

## 10. `.editorconfig` ✏️

**Purpose:** Keeps formatting consistent across editors.

```ini
root = true
[*]
indent_style = space
indent_size = 4
```

---

## 11. `.dockerignore` 🐳

**Purpose:** Prevents files from being copied into Docker images.

```dockerignore
.git
__pycache__/
.env
```

---

## Quick Summary Table

| File             | Purpose             |
| ---------------- | ------------------- |
| `.gitignore`     | Ignore files        |
| `.gitattributes` | File behavior rules |
| `.gitkeep`       | Track empty folders |
| `.gitmodules`    | Submodules          |
| `.gitconfig`     | User config         |
| `.github/`       | GitHub automation   |
| `README.md`      | Project info        |
| `LICENSE`        | Usage rights        |
| `CODEOWNERS`     | Review rules        |
| `.editorconfig`  | Formatting rules    |
| `.dockerignore`  | Docker build ignore |

---

### One-line takeaway

> **Git helper files tell Git, editors, and platforms how to behave around your code.**
