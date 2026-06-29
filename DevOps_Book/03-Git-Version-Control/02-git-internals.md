# Chapter 02 — Git Internals: How Git Really Works

## Learning Objectives

By the end of this chapter, you will be able to:

- Describe the structure of the `.git` directory and the purpose of each item inside it
- Explain Git's four object types: blob, tree, commit, and tag
- Understand how SHA-1 content-addressed storage ensures data integrity
- Decode a commit object's structure (tree pointer, parent, author, committer, message)
- Use `git cat-file` to inspect raw Git objects
- Explain the Directed Acyclic Graph (DAG) model of commit history
- Describe what a branch really is at the filesystem level
- Clarify what HEAD is and what "detached HEAD" means
- Explain the role of the staging area (index) between the working tree and the repository
- Describe how Git compresses objects into packfiles

---

## 2.1 Why Internals Matter

Most Git tutorials teach you commands. This chapter teaches you *why* those commands do what they do. That distinction matters enormously.

When you understand Git's internals:
- `git reset --hard` stops being terrifying — you understand exactly what state it restores
- `git rebase` becomes intuitive — you see it as replaying commits onto a new base
- "Detached HEAD" no longer causes panic — you know HEAD is just a pointer
- You can debug any strange state because you know where to look

Git's data model is deceptively simple: a small set of immutable objects linked together by content-addressed hashes. Everything Git does — branching, merging, rebasing, history — is built on top of that foundation.

---

## 2.2 The .git Directory

When you run `git init` in a directory, Git creates a `.git` subdirectory. This single directory *is* the repository. The files around it (your project files) are just the working tree — a checked-out view of the data stored in `.git`.

Run `ls -la .git/` in a freshly initialized repository:

```
.git/
├── HEAD                   # Pointer to the current branch (or commit in detached state)
├── config                 # Repository-local configuration
├── description            # Used by GitWeb; irrelevant for most users
├── hooks/                 # Client-side hook scripts (pre-commit, commit-msg, etc.)
│   ├── pre-commit.sample
│   ├── commit-msg.sample
│   └── ...
├── info/
│   └── exclude            # Like .gitignore but not committed to the repo
├── objects/               # The object database — blobs, trees, commits, tags
│   ├── info/
│   └── pack/
└── refs/                  # Named pointers to commits (branches and tags)
    ├── heads/             # Local branches (e.g., refs/heads/main)
    └── tags/              # Tags (e.g., refs/tags/v1.0)
```

After a few commits, you will also see:

```
.git/
├── COMMIT_EDITMSG         # The message from the last commit (for reference)
├── FETCH_HEAD             # The tip of the last fetched branch
├── MERGE_HEAD             # The commit being merged (during a merge in progress)
├── ORIG_HEAD              # The previous HEAD before a dangerous operation (reset, merge)
├── REBASE_HEAD            # The commit being rebased (during a rebase in progress)
├── index                  # Binary file — the staging area
├── logs/
│   ├── HEAD               # Reflog for HEAD movements
│   └── refs/
│       └── heads/
│           └── main       # Reflog for the main branch
└── objects/
    ├── 4b/
    │   └── 825dc642cb6eb9a060e54bf8d69288fbee4904  # An object (hash split: 2-char dir + 38-char file)
    └── pack/
```

**Key files to know:**

| File/Dir | Purpose |
|---|---|
| `HEAD` | Points to the current branch ref or directly to a commit (detached) |
| `config` | Repo-local git config (overrides `~/.gitconfig`) |
| `objects/` | Content-addressed store for all Git data |
| `refs/heads/` | One file per local branch, containing the commit SHA |
| `refs/tags/` | One file per tag |
| `index` | Binary staging area file |
| `logs/` | History of where HEAD and branches have pointed (reflog) |
| `ORIG_HEAD` | Safety snapshot before merge/rebase/reset |
| `hooks/` | Scripts that Git runs at lifecycle events |

---

## 2.3 The Git Object Model

Git is, at its core, a **content-addressed key-value store**. You put any content in, and Git gives you back a SHA-1 hash that you can later use to retrieve that content. Everything in Git is built on four object types.

### SHA-1 Content Addressing

When Git stores any object, it:

1. Takes the raw content
2. Prepends a header: `<type> <byte-length>\0`
3. Computes the SHA-1 hash of the result (40 hex characters)
4. Compresses the content with zlib
5. Writes it to `.git/objects/<first-2-chars>/<remaining-38-chars>`

The critical property: **the same content always produces the same hash**. Git never stores the same data twice. If you have two identical files in different directories, they share one blob object.

### Object Type 1: Blob

A **blob** (Binary Large OBject) stores the content of a single file. It contains *nothing* except the raw file bytes — no filename, no permissions, no path. A blob is pure content.

```
blob 14\0Hello, World!\n
     ^                   SHA-1 → e965047ad7c57865823c7d992b1d046ea66edf78
```

### Object Type 2: Tree

A **tree** stores a directory snapshot. It is analogous to a filesystem directory. Each entry in a tree contains:

- A mode (file permissions: `100644` for regular file, `100755` for executable, `040000` for directory)
- An object type (blob or tree)
- The SHA-1 of the referenced object
- The filename

```
tree 73\0
100644 blob a8c2f... README.md
100644 blob 3f4e1... main.go
040000 tree 7ab3c... src/
```

Trees can reference other trees (subdirectories), creating a recursive structure that represents an entire directory hierarchy.

### Object Type 3: Commit

A **commit** is a snapshot of the entire project at a point in time. It contains:

- A pointer to the **root tree** object (the top-level directory state)
- Zero or more **parent commit** SHA-1s (zero for the first commit; two for a merge commit)
- **Author**: name, email, timestamp (when the change was originally made)
- **Committer**: name, email, timestamp (when it was committed — can differ after rebase)
- **Commit message**

```
commit 250\0
tree   4b825dc642cb6eb9a060e54bf8d69288fbee4904
parent 7ec0a5f8e47b03ad52af3b00d37bdc5553ebfd81
author Jane Doe <jane@example.com> 1714000000 +0000
committer Jane Doe <jane@example.com> 1714000000 +0000

Add user authentication module

Implements JWT-based auth with refresh token rotation.
Closes #42
```

The SHA-1 of a commit is a hash of *all* of this content, including the parent hash. This means that if you change any commit in history, all subsequent commits necessarily get different hashes — it is cryptographically impossible to silently alter history.

### Object Type 4: Tag

An **annotated tag** (as opposed to a lightweight tag) is a full Git object that contains:

- A pointer to the tagged object (usually a commit)
- The tagger's name, email, and timestamp
- A tagging message
- An optional GPG signature

```
tag 155\0
object 9fceb02d0ae598e95dc970b74767f19372d61af8
type   commit
tag    v2.0.0
tagger Jane Doe <jane@example.com> 1714100000 +0000

Release version 2.0.0

Includes OAuth2 integration and API v2 endpoints.
```

Lightweight tags are just a ref file (a pointer) — they are not objects in the database.

### Object Type Summary

| Object | Contains | Analogous To |
|---|---|---|
| blob | File content (bytes) | A file on disk |
| tree | List of (mode, type, sha, name) entries | A directory |
| commit | tree pointer, parent(s), author, message | A snapshot + metadata |
| tag | commit pointer, tagger, message, signature | A named milestone |

---

## 2.4 Exploring Objects with git cat-file

`git cat-file` is the plumbing command for inspecting raw Git objects.

### Setup: Create a simple repo

```bash
mkdir git-internals-demo
cd git-internals-demo
git init
echo "Hello, World!" > hello.txt
git add hello.txt
git commit -m "Initial commit"
```

### Inspect object types

```bash
# List all objects
find .git/objects -type f

# Example output (your hashes will differ):
# .git/objects/8a/b686eafeb1f44702738c8b0f24f2567c36da6d  ← commit or tree
# .git/objects/a0/423896973644771497bdc03eb99d5281615b51  ← blob

# Check the type of a specific object
git cat-file -t 8ab686ea
# commit

git cat-file -t a0423896
# blob
```

### Inspect blob content

```bash
git cat-file -p a0423896
# Hello, World!
```

### Inspect a commit

```bash
# Get the current HEAD commit hash
git rev-parse HEAD
# e.g., 7ec0a5f8e47b03ad52af3b00d37bdc5553ebfd81

git cat-file -p HEAD
# tree 4b825dc642cb6eb9a060e54bf8d69288fbee4904
# author Jane Doe <jane@example.com> 1714000000 +0000
# committer Jane Doe <jane@example.com> 1714000000 +0000
#
# Initial commit
```

### Inspect a tree

```bash
# From the commit output above, get the tree hash
git cat-file -p 4b825dc6
# 100644 blob a0423896973644771497bdc03eb99d5281615b51    hello.txt
```

You have now traced the full chain: **HEAD → commit → tree → blob → file content**.

### Verify SHA-1 yourself

```bash
# Git computes: sha1("blob 14\0Hello, World!\n")
echo -e "Hello, World!" | git hash-object --stdin
# a0423896973644771497bdc03eb99d5281615b51
```

---

## 2.5 The Directed Acyclic Graph (DAG)

Git's commit history forms a **Directed Acyclic Graph (DAG)** — a graph where:
- **Nodes** are commits
- **Edges** are parent pointers (a commit points to its parent)
- The graph is **directed** (edges go in one direction — child to parent)
- The graph is **acyclic** (you can never follow parent pointers and arrive back at the same commit)

### Visualizing the DAG

```
A ← B ← C ← D    (main branch)
            ↑
            └─ E ← F    (feature branch)

Legend:
  A = root commit (no parent)
  B, C, D = linear commits on main
  E, F = commits on a feature branch that diverged from D
```

### Merge commits in the DAG

```
A ← B ← C ← D ← M    (main, after merge)
                ↑↑
                ││
            E ← F   (feature)
```

`M` (the merge commit) has **two parents**: `D` and `F`.

### Branches are just pointers

This is one of Git's most elegant design decisions. A **branch** is literally just a file containing a 40-character SHA-1 hash:

```bash
cat .git/refs/heads/main
# 7ec0a5f8e47b03ad52af3b00d37bdc5553ebfd81
```

When you make a new commit, Git:
1. Writes the commit object to `.git/objects/`
2. Updates the current branch file to point to the new commit hash

Creating a branch is nearly free — it only creates a 41-byte file.

```bash
git branch feature-login

cat .git/refs/heads/feature-login
# 7ec0a5f8e47b03ad52af3b00d37bdc5553ebfd81  (same as main at creation time)
```

---

## 2.6 HEAD: The Current Position Pointer

`HEAD` is a special ref that tells Git: "this is where I am right now." It is almost always an **indirect reference** — it points to a branch name, and that branch name points to a commit.

```bash
cat .git/HEAD
# ref: refs/heads/main
```

When you make a commit:
1. Git reads `HEAD` → finds it points to `refs/heads/main`
2. Reads `refs/heads/main` → finds the parent commit hash
3. Creates the new commit object with that parent
4. Updates `refs/heads/main` to the new commit hash

HEAD itself does not change — it still says `ref: refs/heads/main`.

### Detached HEAD

When you check out a specific commit hash (instead of a branch name), HEAD points directly to that commit hash, not to a branch:

```bash
git checkout 4b825dc6

cat .git/HEAD
# 4b825dc642cb6eb9a060e54bf8d69288fbee4904
```

This is **detached HEAD** state. You can look around, run tests, and even make commits — but those commits will not be referenced by any branch. If you switch away without creating a branch, those commits become unreachable and will eventually be garbage-collected.

```
# Normal state
HEAD → refs/heads/main → commit D

# Detached HEAD state
HEAD → commit C  (directly)
```

To recover from detached HEAD:

```bash
# Create a new branch here (saves your work)
git branch recovery-branch

# Or, if you did not make any commits, just switch back
git checkout main
```

---

## 2.7 The Staging Area (Index)

The staging area, also called the **index**, is a binary file at `.git/index`. It serves as the intermediary between your working tree and the repository.

The index stores:
- The SHA-1 of each staged blob
- The filename and path
- File permissions (mode)
- Timestamps and size (for performance — helps Git quickly detect if a file has changed)

When you run `git add hello.txt`:
1. Git computes the SHA-1 of `hello.txt`'s content and writes a blob object
2. Git updates `.git/index` to record: "file `hello.txt` is now represented by blob `<hash>`"

When you run `git commit`:
1. Git reads all entries from the index
2. Constructs tree objects representing the directory structure
3. Creates a commit object pointing to the root tree
4. Moves the branch pointer to the new commit
5. The index is NOT cleared — it continues to mirror the just-committed state

This is why, immediately after a commit, `git status` shows nothing staged — the index matches the last commit.

### Why the index exists

The staging area lets you craft precise commits. You might have three modified files but only want to commit two of them. You stage exactly those two. The third stays modified in the working tree but is not included in the commit. This level of control is what makes `git add -p` (interactive patching) so powerful — you can commit individual *hunks* of a file, not just entire files.

---

## 2.8 Packfiles

Initially, Git stores each object as a separate compressed file (a "loose object"). For a repository with thousands of commits, this creates thousands of small files — inefficient for disk I/O and network transfer.

Git periodically (or when you run `git gc`) consolidates loose objects into a **packfile**: a single binary file in `.git/objects/pack/`. Packfiles use delta compression — instead of storing every version of a file independently, they store a base version and the differences (deltas) between versions.

```bash
# Manually trigger garbage collection and repacking
git gc

# After gc, inspect the pack directory
ls .git/objects/pack/
# pack-abc123...def.idx    ← index file (for fast lookup)
# pack-abc123...def.pack   ← the actual pack file
```

The index (`.idx`) file allows Git to find objects inside the pack file by SHA-1 without reading the entire pack. On a large repository, packfiles can reduce disk usage by 10–50× compared to loose objects.

---

## 2.9 Why This Mental Model Matters

Every high-level Git operation maps directly to low-level object manipulation:

| Operation | What Actually Happens |
|---|---|
| `git add` | Writes blob object(s), updates index |
| `git commit` | Writes tree objects + commit object, moves branch pointer |
| `git branch foo` | Creates `.git/refs/heads/foo` with current commit hash |
| `git checkout foo` | Updates HEAD to point to `refs/heads/foo`, updates working tree |
| `git merge` | Creates a new commit object with two parent hashes |
| `git rebase` | Creates new commit objects (with different parent hashes) and moves the branch pointer |
| `git reset --hard <hash>` | Moves the branch pointer to `<hash>`, resets index and working tree |
| `git tag v1.0` | Creates `.git/refs/tags/v1.0` with a commit hash |
| `git gc` | Packs loose objects into packfiles, prunes unreachable objects |

Understanding this table means you can reason about any Git operation without memorizing its behavior — you just think about which objects are created or modified, and where the pointers move.

---

## Summary

| Concept | Key Takeaway |
|---|---|
| `.git` directory | The entire repository; your files are just a working copy |
| Content-addressed storage | SHA-1(content) = address; same content = same hash |
| Blob | File content only; no name, no path |
| Tree | Directory snapshot; maps names to blobs/trees |
| Commit | Snapshot + metadata + parent pointer |
| Annotated tag | Named milestone object with message |
| DAG | Commits form a directed acyclic graph via parent pointers |
| Branch | A 41-byte file containing a commit SHA-1 |
| HEAD | Pointer to the current branch (or directly to a commit in detached state) |
| Detached HEAD | HEAD points to a commit, not a branch; new commits are not saved |
| Index (staging area) | Binary file recording what will go into the next commit |
| Packfiles | Delta-compressed bundles of many objects for efficiency |

---

## Knowledge Check

1. What is stored in `.git/refs/heads/main`?
2. What is the difference between a blob and a tree?
3. Why does changing one commit in history change the SHA-1 of all subsequent commits?
4. A friend says "I detached HEAD and made 3 commits, then switched back to main and now they're gone!" What happened, and how could they have prevented it?
5. What is the difference between the author and the committer in a commit object?
6. After `git add` but before `git commit`, where exactly is your change stored? (Name the Git concept and the file.)
7. What command reveals the raw content of any Git object?

---

## Hands-On Exercise

### Exercise 2.1: Explore Objects After Commits

1. Create a new Git repository: `mkdir internals-lab && cd internals-lab && git init`
2. Create two files: `echo "Alpha" > alpha.txt && echo "Beta" > beta.txt`
3. Stage and commit both files: `git add . && git commit -m "Add alpha and beta"`
4. Run `find .git/objects -type f` and note the hashes.
5. Use `git cat-file -t <hash>` on each object. Identify which are blobs, trees, and commits.
6. Use `git cat-file -p <commit-hash>` to read the commit.
7. From the commit's tree hash, use `git cat-file -p <tree-hash>` to see the tree.
8. From the tree's blob hashes, use `git cat-file -p <blob-hash>` to verify you get "Alpha" and "Beta".
9. Read `cat .git/HEAD` and `cat .git/refs/heads/main`. Confirm they chain correctly.
10. Make a second commit and re-run step 9. Notice how `refs/heads/main` changed.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./01-introduction-to-git.md">← Previous: Introduction to Git</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./03-essential-commands.md">Next: Essential Commands →</a>
</div>
