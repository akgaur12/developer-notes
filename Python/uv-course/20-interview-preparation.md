# Interview Preparation

You've built the full stack over nineteen chapters: what uv actually replaces and why it's architected around a real resolver and a global content-addressable cache (Ch 1–3), Python version management without pyenv (Ch 4), project creation and structure (Ch 5–6), dependency management and lock-file reproducibility (Ch 7–8), running code with `uv run` including PEP 723 inline scripts (Ch 9), the dev-dependency-vs-global-tool distinction (Ch 10–11), workspaces and monorepos (Ch 12), the full FastAPI + SQLAlchemy + Alembic developer loop (Ch 13), Docker integration (Ch 14), CI/CD integration (Ch 15), publishing packages (Ch 16), a consolidated best-practices checklist, the common failure modes ([Chapter 18](./18-common-mistakes-and-pitfalls.md)), and a set of [capstone projects](./19-capstone-projects.md) that assembled, split, containerized, and published pieces of ExpenseFlow end to end. This final chapter is not new material — it is a rehearsal. Its job is to take everything from Chapters 1–19 and compress it into the exact shape a technical interviewer asks for: a crisp conceptual answer delivered in under a minute, a calm, structured diagnosis under scenario pressure, a defensible system-design walkthrough with justified trade-offs, correct `pyproject.toml`/Dockerfile/CI-workflow snippets written under a shared editor, and a war story that proves you've actually shipped a uv-managed project to production, not just read about it. Work through this chapter the way you'd rehearse for a real interview loop: read a question, form your own answer before reading the model answer, and treat any gap between the two as a pointer back to the specific earlier chapter you need to revisit tonight.

---

## Learning Objectives

By the end of this chapter, you will be able to:

- Answer 16 core uv conceptual interview questions confidently and instructively, spanning what uv replaces, its resolver, the lock/sync/run distinction, tool management, workspaces, Python version management, PEP 723, and Docker integration
- Diagnose realistic production scenarios — a dependency conflict that only reproduces on one machine, a bloated Docker image, a confusing cache-miss pattern, an ad hoc `pip install` that desynced an environment, and a Python-version-specific CI failure — using the same diagnostic discipline taught in Chapters 8, 14, and 18
- Write correct `pyproject.toml` dependency-group configuration, Dockerfile snippets, GitHub Actions workflow YAML, PEP 723 scripts, and workspace configuration from a plain-English problem statement under interview conditions
- Deliver a structured, interview-shaped system-design answer for a multi-service Python monorepo with shared internal libraries built on uv workspaces
- Recognize composite, illustrative production case studies grounded in ExpenseFlow that show how this course's concepts play out as real incidents
- Run a full 45-minute mock interview against yourself and honestly self-grade the result
- Walk into a backend/tooling-focused interview able to state assumptions, name trade-offs, and justify every dependency-management decision instead of reciting command syntax

---

## Prerequisites for This Chapter

This is the capstone review chapter of the entire course. It assumes you have completed, or are comfortable quickly skimming back through, **all of Chapters 1–19**:

- **Ch 1–3**: what uv replaces, the standards it builds on (PEP 621/508/723), and its resolver/cache/Python-interpreter architecture
- **Ch 4–6**: Python version management, project creation (`uv init`, `pyproject.toml` anatomy), and virtual environments
- **Ch 7–9**: dependency management, lock files and `--frozen`/`--locked`, and running code with `uv run`
- **Ch 10–11**: dev dependencies vs. global tools (`uv tool`/`uvx`)
- **Ch 12**: workspaces and monorepos
- **Ch 13–14**: the full FastAPI/SQLAlchemy/Alembic developer loop, and Docker integration
- **Ch 15–16**: CI/CD integration and publishing packages
- **Ch 17–18**: the consolidated best-practices checklist and known failure modes
- **Ch 19**: the seven capstone projects, including ExpenseFlow's own tooling journey and its Dockerized, CI-gated, and published forms

Every answer below is instructive on its own, but if any of it feels unfamiliar rather than "oh right, I remember this," that's your signal to reopen the relevant chapter before the interview — not during it.

---

## 1. Conceptual Q&A

Unlike the "Knowledge Check" sections in earlier chapters, which deliberately withhold answers so you self-test honestly, every question in this section comes with a full model answer — because that's exactly what an interview demands of you in real time.

### Q1. What is uv, and what does it replace?

uv is a single, Rust-based Python package and project manager built by Astral (the makers of Ruff). It exists to replace a whole cluster of tools Python developers have historically had to stitch together by hand: `pip` for installing packages, `virtualenv` for isolating environments, `pip-tools` for compiling a locked requirements file, `pyenv` for managing multiple Python interpreter versions, `pipx` for running CLI tools in their own isolated environments, and — for teams that had adopted it — much of what `Poetry` does for project and dependency management. The reason it consolidates all of this into one tool isn't just convenience: it's a proper resolver (a PubGrub-style algorithm, not pip's historically weaker backtracking), a single global content-addressable cache shared across every project on the machine (so a package already installed for one project is nearly instant to install into another via hardlinks/reflinks instead of a fresh copy every time), and a consistent, standards-based (`pyproject.toml`, PEP 621/508/723) configuration surface across every one of those concerns instead of five separate tools each with their own file format and mental model (Ch 1, Ch 2).

### Q2. How does uv's resolver differ from pip's?

pip's classic resolver (even its newer 2020+ resolver) works by picking a candidate version for a dependency, trying it, and backtracking when a conflict is discovered later — a correctness model that historically could be slow and, in some conflict shapes, could still resolve to something inconsistent without a clear explanation of why. uv's resolver is based on PubGrub, the same family of algorithm used by Cargo (Rust) and Dart's package manager: it reasons about the *entire* set of version constraints across the whole dependency graph at once, and when a conflict exists, it can produce a clear, human-readable explanation of exactly which packages' requirements are incompatible with each other, rather than a deep, confusing backtracking trace. Combined with being written in Rust rather than pure Python, and backed by a persistent, shared cache instead of resolving from scratch against the network every time, the practical result is a resolver that's both meaningfully faster and meaningfully easier to reason about when it fails (Ch 2, Ch 3).

### Q3. What's the difference between `uv lock`, `uv sync`, and `uv run`?

These are three distinct operations that are easy to blur together because `uv run` quietly triggers the other two when needed. `uv lock` resolves your project's dependencies — reading `pyproject.toml`'s constraints and writing the exact, fully-pinned result (versions, hashes, platform markers) to `uv.lock` — without touching any virtual environment at all. `uv sync` takes an existing `uv.lock` and makes your `.venv` match it exactly: installing what's missing, removing what shouldn't be there, without re-resolving anything (assuming the lockfile is already up to date). `uv run <command>` is the actual day-to-day entry point: before running your command, it ensures the environment is in sync with the lockfile (re-locking first if `pyproject.toml` has changed since the lockfile was written, then syncing), and only then executes your command inside that guaranteed-correct environment. In short: `lock` resolves, `sync` applies, `run` guarantees-then-executes (Ch 8, Ch 9).

### Q4. What's the difference between a project dependency and a uv tool?

A **project dependency** is versioned per-project, declared in that project's `pyproject.toml`, pinned in that project's `uv.lock`, and installed into that project's own `.venv` — every teammate and every CI run resolving that same lockfile gets the exact same version, which is the entire point. A **uv tool** (`uv tool install`, or run ad hoc with `uvx`) is installed once, globally, in its own isolated environment completely separate from any project, and is meant for CLI applications you use *across* projects rather than *within* one specific project's code — things like `cookiecutter`, `httpie`, or a code-generation CLI. The distinction that trips people up in practice: `ruff`, `mypy`, and `pytest` are almost always the *first* category for a real team, even though they're also individually installable as global tools — a team needs every developer and every CI run to use the exact same pinned version of its linter and test runner, which only a project dev-dependency (not a globally installed tool that could silently drift between machines) actually guarantees (Ch 10, Ch 11).

### Q5. When would you reach for a workspace?

A uv workspace is worth its complexity once you genuinely have more than one importable or deployable unit that needs to share code and be resolved/locked together — for example, a FastAPI API and a background worker that both import the same Pydantic schemas and constants from an internal shared library. The workspace mechanism (`[tool.uv.workspace]` at the root, each member with its own `pyproject.toml`, one shared `uv.lock` resolving everything at once) exists specifically to keep that shared code and its consumers from drifting out of sync with each other the way they would if they were separate, independently-locked projects. It is *not* worth reaching for just because a repository happens to contain more than one Python project, or as a way to "future-proof" a single-service project that doesn't have a second consumer yet — the correct default for a genuinely standalone project is still a single, ordinary project, and adding workspace structure prematurely adds real coordination overhead (a shared lockfile every member's changes can affect) for no real benefit yet (Ch 12).

### Q6. How does uv manage Python versions without pyenv?

uv downloads and manages prebuilt, standalone Python interpreters (via the `python-build-standalone` project) directly — `uv python install 3.13` fetches a working interpreter without compiling from source and without needing pyenv, a system package manager, or any other tool involved at all. `uv python list` shows installed and installable versions, `uv python pin` writes a `.python-version` file recording which version a specific project expects, and `uv python find` resolves which interpreter uv would actually use for a given context. Multiple versions can be installed side by side with no conflict, and `uv run`/`uv sync` automatically pick up the project's pinned version without you needing to activate anything — the net effect is that an entire class of "which Python am I even running" confusion, and the separate tool pyenv historically existed to solve, is handled by the same single binary that also manages your dependencies (Ch 4).

### Q7. What's PEP 723, and when would you use an inline script instead of a full project?

PEP 723 defines a way to embed a script's own dependency metadata directly inside the script file itself, as a specially formatted comment block (`# /// script` ... `# dependencies = [...]` ... `# ///`). `uv run script.py` reads that block and runs the script in an ephemeral, isolated environment built just for those declared dependencies — no `pyproject.toml`, no `uv.lock`, no project structure required at all. The right time to reach for this is a genuinely standalone, one-off script that has nothing to do with your main project's dependency set — ExpenseFlow's `backfill_currency.py` maintenance script, which needs `httpx` for one outbound call but has no reason to make `httpx` a dependency of the entire FastAPI application, is exactly this case. The alternative — adding the dependency to the main project just because it's convenient in the moment — is precisely the kind of dependency-set bloat this pattern exists to avoid; if a script instead grows real ongoing importance, tests, or more than a couple of dependencies, that's the signal to promote it into a real project instead (Ch 9, Ch 17).

### Q8. How do you correctly Dockerize a uv project?

Use the official pattern: copy the `uv`/`uvx` binaries in from `ghcr.io/astral-sh/uv`'s image (`COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/`), then, in a multi-stage build, copy only `pyproject.toml` and `uv.lock` *before* your application source code and run `uv sync --frozen --no-dev --no-install-project` — this produces a dependency-only layer that Docker's build cache can reuse across builds as long as your dependencies haven't changed, even if your source code has. Only after that do you copy the application source and run a final `uv sync --frozen --no-dev` to install the project itself. Two environment variables matter specifically inside a container: `UV_COMPILE_BYTECODE=1` (precompiles `.pyc` files at build time rather than on first run) and `UV_LINK_MODE=copy` (because uv's default hardlink-based cache installs don't survive across a Docker `COPY` layer boundary the way they do on a single local filesystem — without this, you'd either get an error or, worse, a confusingly larger-than-expected image as uv silently falls back to copying). `--frozen` guarantees the lockfile is used exactly as committed with no re-resolution, and `--no-dev` guarantees `pytest`/`ruff`/`mypy` never end up in the shipped image (Ch 14).

### Q9. What's the difference between `--frozen` and `--locked`, and when do you use each?

Both exist to stop `uv sync` from silently re-resolving your dependencies, but they fail differently. `--locked` asserts the lockfile is already fully up to date with `pyproject.toml`'s constraints — if it isn't (someone changed a version constraint and forgot to re-run `uv lock`), the command fails immediately with a clear error rather than quietly re-resolving something different than what's committed. `--frozen` goes further: it skips checking whether the lockfile matches `pyproject.toml` at all, and simply installs exactly what's already in `uv.lock` as-is, even if it's technically stale relative to `pyproject.toml`. The practical guidance: use `--locked` in CI, where you *want* to be told loudly if a developer forgot to commit an updated lockfile — that's a real bug to catch. Use `--frozen` in a Docker build, where you deliberately want to install precisely the locked versions without spending time re-verifying anything, since the image is being built from a commit that (if CI is doing its job) has already been validated (Ch 8, Ch 14).

### Q10. How does uv's cache work, and why does that matter specifically inside Docker?

uv keeps a single global cache, keyed by package content (content-addressable), shared across every project on the machine. When you install a package that's already cached from some other project, uv doesn't re-download or re-copy the package's files — it hardlinks (or, on filesystems that support it, uses a copy-on-write reflink) straight from the cache into the new project's `.venv`, which is why installing an already-cached package into a brand-new environment can be close to instant. Inside a Docker build, this exact mechanism is also the source of a common gotcha: a hardlink only works within a single filesystem, and a Docker `COPY` between build stages (or the layer-committing process generally) doesn't preserve that hardlink relationship — so a naive assumption that "the cache will just work the same way it does locally" can produce a surprisingly large or slow image if `UV_LINK_MODE` isn't explicitly set to `copy` for the containerized build (Q8, Ch 3, Ch 14).

### Q11. What standards does uv build on, and why does that matter compared to Poetry?

uv's project configuration lives in the standard, PEP 621-defined `[project]` table of `pyproject.toml` — the same section any standards-compliant Python build tool understands — plus PEP 508 for dependency version specifiers and PEP 723 for inline script metadata. This matters concretely because a `pyproject.toml` written for uv is, for the most part, portable: another standards-compliant tool can read the same `[project]` metadata without translation. Poetry, historically, stored its project and dependency configuration in a proprietary `[tool.poetry]` section with its own non-standard dependency-specifier syntax, which meant a Poetry-managed project's metadata wasn't directly usable by other tools without extra conversion — a real, if narrower now than it used to be, form of tooling lock-in. Standards-based configuration isn't just philosophically nicer — it's the practical reason migrating away from (or interoperating with) uv later is a smaller, more mechanical exercise than migrating away from a tool with its own bespoke format ever was (Ch 2).

### Q12. What's the difference between a dependency group and an optional dependency/extra?

A **dependency group** (`[dependency-groups]`, most commonly the `dev` group added via `uv add --dev`) is for dependencies needed to *work on* the project — testing, linting, type-checking — that should never ship in whatever gets built or deployed from the project; they're installed by default during local development (`uv sync` includes them unless told otherwise) but explicitly excluded with `--no-dev` for a production build. An **optional dependency / extra** (`[project.optional-dependencies]`) is for dependencies needed to *use* the project in a particular configuration — for example, a library that supports multiple database drivers might define a `postgres` extra pulling in `asyncpg` and a `mysql` extra pulling in a different driver, installed selectively with `uv sync --extra postgres`. The distinguishing question to ask: is this something a *consumer* of the finished project might or might not need (extra), or something only a *developer working on* the project needs and a consumer should never see (dependency group)? Conflating the two is a common and avoidable source of bloated production installs (Ch 7).

### Q13. How do you correctly set up dev dependencies and tooling for a real team?

Add `pytest`, `ruff`, `mypy`, and `pre-commit` via `uv add --dev`, so they're versioned in `pyproject.toml`'s dev dependency group and pinned in the shared `uv.lock` — every developer and every CI run then installs the exact same versions by construction, not by convention. Run them consistently through `uv run` (`uv run pytest`, `uv run ruff check`, `uv run ruff format`, `uv run mypy`) rather than any globally installed equivalent, and wire `pre-commit`'s own hooks to invoke `uv run` internally as well, so even a pre-commit hook can't accidentally use a different, unpinned version of the tool it's running. A `Makefile` or `justfile` wrapping the common sequence (`uv run ruff check && uv run mypy && uv run pytest`) is a reasonable convenience for a team, but it's exactly that — a convenience, not a substitute for every individual command being correct and reproducible on its own (Ch 10).

### Q14. What is `uv build`/`uv publish`, and what's trusted publishing?

`uv build` produces a standard Python package distribution from a project — both a source distribution (sdist) and a wheel — reading `[build-system]`/`[project]` metadata from `pyproject.toml` exactly the way any PEP 517-compliant build tool would. `uv publish` uploads that built distribution to a package index (PyPI, TestPyPI, or a private/internal index). **Trusted publishing** is the modern, token-free authentication mechanism for that upload: instead of storing a long-lived PyPI API token as a CI secret (which can leak, needs rotation, and is scoped by hand), you configure PyPI to trust a specific GitHub Actions workflow (identified by repository and workflow file) via OpenID Connect (OIDC) — the workflow proves its identity cryptographically at publish time with no stored secret involved at all. This is the recommended approach for any new CI release workflow specifically because it removes an entire category of credential-leak risk (Ch 16).

### Q15. How would you migrate an existing pip/Poetry project to uv?

For a plain pip + `requirements.txt` project: run `uv init` inside the existing directory (careful not to overwrite an existing `pyproject.toml` if one already exists in some minimal form), then translate the dependency list into `[project.dependencies]` — `uv add` each package one at a time is usually cleaner than a bulk import, since it lets uv actually resolve the whole set correctly rather than trusting whatever pip previously happened to have installed. For a Poetry project, the migration is mostly a metadata translation: Poetry's `[tool.poetry.dependencies]` needs to become the standard `[project.dependencies]`/`[project.optional-dependencies]` shape, and Poetry's own lock file (`poetry.lock`) doesn't carry over — you'll run `uv lock` fresh once the dependency list is correctly expressed in the standard format. In both cases, the practical validation step is the same: after migrating, run `uv run pytest` (or the project's existing test suite) and confirm nothing behaves differently, and commit the newly generated `uv.lock` as the project's new source of truth going forward (Ch 2, Ch 7, Ch 8).

### Q16. What's the difference between an application project and a library project in uv (`--app`/`--lib`/`--package`)?

`uv init --app` (the default) scaffolds a project meant to be *run*, not imported by other projects — ExpenseFlow's own FastAPI service is this shape. `uv init --lib` scaffolds a project meant to be *imported* — it produces an installable, `src/`-layout package with the build metadata needed for other projects (or a workspace member) to depend on it, exactly the shape `expenseflow-shared` needs. `--package` is a related flag ensuring a project is buildable/installable regardless of whether it's primarily an app or a library, which matters most for workspace members that need to be installed into a sibling member's environment. Getting this choice right at project creation time saves a later restructuring: an application scaffolded without library-appropriate packaging metadata will need that metadata added retroactively the moment another project needs to depend on it, exactly the situation Chapter 12's workspace split runs into with `expenseflow-shared` (Ch 5, Ch 12).

---

## 2. Scenario-Based Questions

### Scenario 1: "CI passes but a teammate's laptop fails with a dependency conflict — what do you check first?"

First, check whether the teammate actually has an up-to-date `uv.lock` and ran a lockfile-respecting sync — `git status`/`git log` on `uv.lock` to confirm it matches what CI is using, and ask exactly which command they ran (`uv sync` vs. `uv sync --locked` vs. an ad hoc `uv pip install` outside the managed workflow entirely). The most common root cause: CI is running `uv sync --locked` (or `--frozen`) against the committed lockfile, while the teammate's local environment somehow diverged from it — either they ran a bare `uv sync` after locally editing a version constraint without re-locking, or they used `uv pip install <package>` directly at some point, which installs into the environment without ever touching `pyproject.toml`/`uv.lock`, silently drifting the environment out of sync with what's actually declared (Ch 18's mixing-`uv-pip`-with-managed-workflow mistake, applied). The fix is almost always to delete their `.venv` and run `uv sync --locked` fresh from the committed lockfile, confirming it now matches CI exactly — and, if the lockfile genuinely needed updating, to run `uv lock` deliberately, review the diff, and commit it as its own change rather than letting an environment silently drift (Ch 8, Ch 18).

### Scenario 2: "A production Docker image is way bigger than expected after switching to uv — what did the team likely get wrong?"

Check, in order: first, whether the Dockerfile passes `--no-dev` on every `uv sync` call in the image build — forgetting it is the single most common way `pytest`, `ruff`, `mypy`, and their own transitive dependencies end up shipped into a production image that never needed them. Second, whether `UV_LINK_MODE` is set to `copy` inside the container build — if uv's default hardlink-based install is silently falling back to full copies across a `COPY` layer boundary (or, worse, if the cache itself is being copied into the final image rather than only the installed `.venv`), the image balloons in a way that looks confusing until you understand the hardlink-across-layers limitation specifically. Third, whether the Dockerfile is even using a proper multi-stage build at all — a single-stage build that leaves the `uv` binary, build tools, and the entire uv cache directory sitting in the final image is a very common, very avoidable source of bloat that a correctly separated builder/final-stage pattern eliminates entirely (Ch 14, Ch 18).

### Scenario 3: "A fresh clone runs `uv sync` and it reinstalls everything from scratch, even though CI's cache should have made this instant — why?"

This is almost always a cache-scoping mismatch, not a uv bug. Check first whether the CI workflow is actually restoring uv's cache directory correctly between runs — `astral-sh/setup-uv` supports cache restoration, but if the cache key isn't keyed appropriately (e.g., keyed only on the runner OS and not on `uv.lock`'s hash, or pointed at the wrong path), a "fresh clone" in CI is functionally a cache miss every time regardless of what ran before. Locally, the equivalent check is whether the developer's global uv cache directory itself was cleared or is pointed somewhere unexpected (`UV_CACHE_DIR` overridden, or running inside an ephemeral container without a persistent volume for the cache) — in both cases, uv's install step isn't "broken," it's correctly falling back to downloading and building from scratch because the cache it would have reused genuinely isn't there this time (Ch 3, Ch 15).

### Scenario 4: "A developer ran `pip install` directly inside the project's venv instead of `uv add` — what's now wrong, and how do you fix it?"

What's wrong: the package is now installed in `.venv`, but it exists nowhere in `pyproject.toml` or `uv.lock` — the environment and the project's declared dependencies have silently diverged. The next teammate (or CI run) doing a `uv sync` from the lockfile will not have that package at all, and code depending on it will fail somewhere else, at a point disconnected from the actual root cause. The fix: identify exactly what was installed this way (`uv pip list` compared against `uv.lock`'s contents is the fastest way to spot the discrepancy), then add it properly with `uv add <package>` so it's declared and locked correctly, or remove it if it was genuinely experimental and shouldn't be a dependency at all. The broader fix is a team habit, not a one-off cleanup: treat `uv pip install` (uv's pip-compatible low-level interface) as an escape hatch for genuinely unusual situations, never as an equivalent, interchangeable way to add a dependency to a managed project (Ch 18).

### Scenario 5: "Two developers with the same `pyproject.toml` get different transitive dependency versions locally — what's the fastest way to confirm the cause?"

Confirm first whether both developers actually have the identical `uv.lock` committed and checked out — a git-level check (`git log -1 --format=%H -- uv.lock` on each machine) rules out the simplest explanation instantly: one of them is on an older commit, or has local uncommitted changes to the lockfile. If the lockfile genuinely is identical on both machines, the next check is whether both actually ran `uv sync` (or `uv sync --locked`) against it, rather than one of them having an environment that predates the current lockfile and was never re-synced — this is exactly the "it worked on my machine" shape Chapter 8 warns about, and `uv sync --locked` on the machine showing different behavior will immediately reveal whether its environment doesn't actually match the committed lockfile at all (Ch 8, Ch 18).

### Scenario 6: "A GitHub Actions matrix job testing Python 3.11/3.12/3.13 fails only on the 3.11 leg — what do you check first?"

Check first whether the failure is genuinely Python-version-specific application behavior (a language feature, standard-library change, or a dependency that has different behavior/availability across those versions) versus a uv/environment setup issue specific to that matrix leg — read the actual failing test's error message before assuming either. If it's a setup issue, confirm the workflow is actually installing Python 3.11 correctly per matrix leg (`uv python install ${{ matrix.python-version }}` or an equivalent `actions/setup-python` step matched to the matrix variable) rather than accidentally reusing whatever Python version the runner image ships with regardless of the matrix value — a surprisingly common mistake is a matrix that varies a workflow *input* without that input actually being wired into the step that installs/selects the interpreter. If the setup is correct and it's a genuine version-specific dependency issue, check whether `requires-python` in `pyproject.toml` and any version-conditional markers in `uv.lock` correctly account for the oldest supported version's actual behavior, rather than only ever having been tested against the newest one locally (Ch 4, Ch 15).

---

## 3. Practical & Configuration Challenges

### Challenge 1 — Correctly scope ExpenseFlow's runtime vs. dev dependencies in `pyproject.toml`

**Problem**: ExpenseFlow's `pyproject.toml` currently has `pytest` and `ruff` listed as plain project dependencies. Fix the scoping so they never ship into a production install.

```toml
[project]
name = "expenseflow"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.32",
    "sqlalchemy>=2.0",
    "alembic>=1.13",
    "asyncpg>=0.30",
    "pydantic-settings>=2.6",
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "ruff>=0.7",
    "mypy>=1.13",
    "pre-commit>=4.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

```bash
# Local development: dev group included by default
uv sync

# Production build / CI dependency-verification step: dev group excluded
uv sync --frozen --no-dev
```

**Why it's correct**: runtime dependencies stay in `[project.dependencies]`, exactly what a production install needs; `pytest`, `ruff`, `mypy`, and `pre-commit` live in the `dev` dependency group, included automatically for local development but explicitly excludable with `--no-dev` — this is the exact mechanism that guarantees a Docker image built with `--frozen --no-dev` never contains test/lint tooling (Q12, Q13, Ch 10, Ch 14).

### Challenge 2 — Write a correct multi-stage Dockerfile for ExpenseFlow

**Problem**: ExpenseFlow needs a production Dockerfile that caches its dependency layer correctly and never ships dev dependencies or the `uv` binary itself into the final image.

```dockerfile
# --- Stage 1: builder ---
FROM python:3.13-slim AS builder

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy

WORKDIR /app

# Copy only the dependency manifests first — this layer stays cached
# across builds as long as dependencies haven't changed.
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project

# Now copy the application source and install the project itself.
COPY src/ ./src/
RUN uv sync --frozen --no-dev

# --- Stage 2: final ---
FROM python:3.13-slim

WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY --from=builder /app/src /app/src

ENV PATH="/app/.venv/bin:$PATH"

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Why it's correct**: the dependency manifests are copied and synced *before* application source, so unrelated source-code changes never invalidate the (usually much slower) dependency-install layer; `--frozen --no-dev` both times guarantees the exact locked, production-only dependency set; `UV_LINK_MODE=copy` avoids the broken-hardlink-across-layers problem; and the final stage contains no `uv` binary, no build tools, and no dev dependencies at all — only the synced `.venv` and application source (Q8, Q9, Q10, Ch 14).

### Challenge 3 — Write a GitHub Actions CI workflow with correct cache restoration and lockfile enforcement

**Problem**: ExpenseFlow needs CI that installs uv, restores its cache between runs, and never silently re-resolves a different dependency set than what's committed.

```yaml
name: CI

on: [pull_request, push]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: expenseflow_test
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true
          cache-dependency-glob: "uv.lock"

      - name: Install Python
        run: uv python install 3.13

      - name: Sync dependencies (locked, never bare sync)
        run: uv sync --locked

      - name: Lint
        run: uv run ruff check .

      - name: Type check
        run: uv run mypy .

      - name: Test
        env:
          DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/expenseflow_test
        run: uv run pytest

      - name: Migration drift smoke check
        run: uv run alembic upgrade head
```

**Why it's correct**: `astral-sh/setup-uv` with `enable-cache: true` restores uv's cache keyed on `uv.lock`'s hash, so unchanged dependencies install almost instantly on subsequent runs; `uv sync --locked` fails loudly if `uv.lock` doesn't match `pyproject.toml`, rather than silently re-resolving something different than what a developer tested locally; and the migration-drift check bridges directly to the sibling Alembic course's own CI-integration chapter (Q9, Ch 8, Ch 15).

### Challenge 4 — Write a PEP 723 inline script for `backfill_currency.py`

**Problem**: ExpenseFlow needs a one-off maintenance script that calls an external exchange-rate API via `httpx`, without adding `httpx` to the main project's dependencies.

```python
# /// script
# requires-python = ">=3.13"
# dependencies = [
#     "httpx>=0.27",
#     "sqlalchemy>=2.0",
# ]
# ///

import httpx
from sqlalchemy import create_engine, text

engine = create_engine("postgresql+psycopg2://postgres:postgres@localhost:5432/expenseflow")

with engine.connect() as conn, httpx.Client() as client:
    rows = conn.execute(text("SELECT id, currency FROM expenses WHERE currency IS NULL"))
    for row in rows:
        response = client.get(f"https://api.example.com/currency-guess/{row.id}")
        guessed = response.json().get("currency", "USD")
        conn.execute(
            text("UPDATE expenses SET currency = :c WHERE id = :id"),
            {"c": guessed, "id": row.id},
        )
    conn.commit()
```

```bash
uv run scripts/backfill_currency.py
```

**Why it's correct**: the `# /// script` block declares this script's own dependencies (`httpx`, `sqlalchemy`) entirely independently of ExpenseFlow's `pyproject.toml`/`uv.lock` — `uv run` builds a throwaway, isolated environment just for this script's declared dependencies and runs it, with zero footprint on the main project's dependency set (Q7, Ch 9).

### Challenge 5 — Configure a uv workspace for `packages/api` + `packages/shared`

**Problem**: ExpenseFlow needs to be restructured so `packages/api` depends on `packages/shared` as a workspace member, not a version-pinned dependency.

```toml
# expenseflow-workspace/pyproject.toml (root)
[tool.uv.workspace]
members = ["packages/*"]
```

```toml
# packages/shared/pyproject.toml
[project]
name = "expenseflow-shared"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = ["pydantic>=2.9"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

```toml
# packages/api/pyproject.toml
[project]
name = "expenseflow-api"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = [
    "expenseflow-shared",
    "fastapi>=0.115",
    "uvicorn[standard]>=0.32",
]

[tool.uv.sources]
expenseflow-shared = { workspace = true }
```

```bash
# from the workspace root — resolves and locks both members together
uv sync
```

**Why it's correct**: `[tool.uv.workspace]` at the root discovers both members; `packages/api`'s `[tool.uv.sources]` entry explicitly marks `expenseflow-shared` as a workspace reference rather than a PyPI lookup, so both members are always resolved and locked together in one `uv.lock` at the root — a change to `expenseflow-shared` is immediately visible to `packages/api` without needing a version bump or republish (Q5, Ch 12).

### Challenge 6 — Write a release workflow that gates `uv publish` on tests passing

**Problem**: `expenseflow-shared` needs a tag-triggered release workflow where the publish step cannot run unless the test job succeeds first.

```yaml
name: Release

on:
  push:
    tags: ["v*"]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --locked
      - run: uv run ruff check .
      - run: uv run mypy .
      - run: uv run pytest

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  publish:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # required for OIDC trusted publishing
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - uses: astral-sh/setup-uv@v3
      - run: uv publish
```

**Why it's correct**: `build` declares `needs: test` and `publish` declares `needs: build`, so the publish step is mechanically incapable of running unless every prior job succeeded — GitHub Actions will not execute a job whose `needs` dependency failed; `permissions: id-token: write` is what enables OIDC trusted publishing with no stored API token secret at all; and triggering only on `v*` tags keeps releases deliberate rather than an automatic side effect of every merge (Q14, Ch 15, Ch 16).

---

## 4. System Design Discussion

### System Design: Structure a multi-service Python monorepo with shared internal libraries using uv workspaces

**Clarifying questions to ask first.** Before designing anything, ask: how many services are involved, and do they deploy independently or on a shared release train? What, concretely, needs to be shared — Pydantic schemas and constants (low-risk, ExpenseFlow's actual case), or something with heavier runtime behavior (a database client wrapper, business logic)? Is there already an ad hoc way code gets duplicated across services today (copy-paste, a private PyPI package, a git submodule), and what's actually painful about it? The answer below assumes the realistic ExpenseFlow-adjacent shape: an `api` service and a new `worker` service that both need the same Pydantic schemas and a couple of shared constants, deployed independently, with a small team owning both.

**The core problem to name explicitly.** Without a workspace, `api` and `worker` either duplicate the shared schemas outright (which drifts the moment one is updated and the other isn't) or depend on a separately published package that has to be version-bumped and republished for every single change, adding release-cycle friction to what might be a one-line schema tweak used entirely within the same team's two services.

**Recommended strategy.** A single uv workspace with three members: `packages/api`, `packages/worker`, and `packages/shared` — a root `pyproject.toml` declaring `[tool.uv.workspace] members = ["packages/*"]`, and both `api` and `worker` depending on `expenseflow-shared` via `[tool.uv.sources]`'s `workspace = true` marker rather than a version pin. This gives one `uv.lock` resolving all three members together — a change to `packages/shared` is immediately usable by both consumers with no publish step, no version bump, and no risk of the two services silently drifting onto different versions of the same schema. Each member still gets its own independent deployment: `packages/api` builds and ships as its own container image, `packages/worker` builds and ships as its own separate container image, and only `packages/shared`'s *code* — not its deployment — is actually shared.

**When to promote a shared library out of the workspace, and when not to.** If `expenseflow-shared` ever needs to be consumed by a service outside this specific team's workspace/repository — a third team's service, or an external partner — that's the signal to genuinely publish it as an independent, versioned package (Chapter 16's pattern, exercised in [Chapter 19's Project 7](./19-capstone-projects.md)), since a workspace path reference only works within one checked-out repository. Within a single team's monorepo, though, promoting every shared piece of code to a fully independent published package by default adds real friction — a version bump and a release for every change — for a coordination problem the workspace model already solves for free at that scope.

**Access control and blast radius.** Each service should still be deployed and run with its own database credentials and its own scoped permissions, regardless of how tightly their code is coupled at the workspace level — sharing a `pyproject.toml`-level resolution doesn't mean sharing a runtime identity or a blast radius. A bug in `packages/worker`'s code should not be able to reach `packages/api`'s database credentials simply because they share a lockfile.

**Alternative worth naming.** If the coordination the workspace provides ever starts feeling insufficient — for example, if `api` and `worker` genuinely need to evolve independently on entirely different release cadences, or different teams start owning each — that's the signal the workspace model has been outgrown, and the shared library should be extracted and published independently (Chapter 16's pattern again), with each service depending on it as an ordinary, version-pinned dependency instead. Naming this as an option, not necessarily the recommendation for the stated scope, is itself a signal of senior-level judgment: the right answer scales with how tightly coupled the teams and release cadences actually are, not with how large the codebase has grown (Ch 12, Ch 16, Ch 19 Project 5, Ch 19 Project 7).

```mermaid
flowchart TB
    subgraph WS["expenseflow-workspace/ (single repo, one uv.lock)"]
        subgraph API["packages/api"]
            APIDEP["depends on expenseflow-shared\n(workspace = true)"]
        end
        subgraph WORKER["packages/worker"]
            WORKERDEP["depends on expenseflow-shared\n(workspace = true)"]
        end
        subgraph SHARED["packages/shared"]
            SCHEMAS["Pydantic schemas, constants"]
        end
        APIDEP -.-> SHARED
        WORKERDEP -.-> SHARED
    end

    API -->|"independent build/deploy"| APIIMG["api container image"]
    WORKER -->|"independent build/deploy"| WORKERIMG["worker container image"]

    SHARED -.->|"outgrown workspace scope?\npromote + publish independently"| PUBLISHED["expenseflow-shared\non a package index (Ch 16)"]
```

---

## 5. Practical Troubleshooting Exercises

### Exercise 1 — "`uv sync --locked` fails in CI right after a teammate's PR merged, but their PR's own CI run was green"

**Symptom**: A PR that added a new dependency passed CI cleanly and merged. The very next CI run on `main` — for an unrelated, later commit — fails at `uv sync --locked` with a message that the lockfile is out of date.

**Diagnosis**: Check first whether the merged PR's `uv.lock` was actually committed in the same commit as the `pyproject.toml` change, or only `pyproject.toml` was changed with the lockfile update missed — a squash-merge or a manual conflict resolution during merge is a common way for the lockfile's final state to end up not matching what CI actually validated on the feature branch. Confirm by running `git log -p -- uv.lock pyproject.toml` around the merge commit and checking whether both files' final states are mutually consistent.

```bash
# Confirm locally whether the lockfile actually matches pyproject.toml right now
uv lock --check
```

**Fix**: run `uv lock` locally against the current `main`, review the diff (it should be small if this is genuinely just a merge-reconciliation gap), and commit the corrected `uv.lock` as its own follow-up commit. As a preventive measure, add a required CI check on `main` itself (not just per-PR) that runs `uv lock --check`, so a lockfile/manifest mismatch introduced by a merge is caught within minutes rather than surfacing on the next unrelated commit's CI run (Ch 8, Ch 15).

### Exercise 2 — "A Docker build is slow every single time, even though `pyproject.toml` hasn't changed in weeks"

**Symptom**: Every CI build re-runs the full `uv sync --frozen --no-dev` dependency-install step from scratch, taking several minutes, despite no dependency changes.

**Diagnosis**: Check the Dockerfile's `COPY` ordering first — if application source code is copied *before* (or in the same layer as) `pyproject.toml`/`uv.lock`, then any source change invalidates Docker's build cache for that entire layer, including the dependency-install step bundled inside it, even though the dependencies themselves haven't changed at all. Also check whether CI's own Docker layer caching is configured at all (e.g., `docker buildx` with a registry or GHA cache backend) — even a perfectly ordered Dockerfile gains nothing from layer caching if the CI runner starts from a completely clean Docker build cache on every single run.

```dockerfile
# Wrong order — invalidates the dependency layer on every source change
COPY . .
RUN uv sync --frozen --no-dev

# Correct order — dependency layer only invalidates when these two files change
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project
COPY src/ ./src/
RUN uv sync --frozen --no-dev
```

**Fix**: reorder the Dockerfile so manifests are copied and synced before source code, and configure CI's Docker build step to persist and restore its layer cache between runs (a registry-backed cache or the CI provider's native Docker cache action). Confirm the fix by making a source-only change and observing the dependency-install step now reports a cache hit instead of reinstalling from scratch (Ch 14, Ch 18).

### Exercise 3 — "`uvx some-tool` behaves differently than a teammate's version of the same tool"

**Symptom**: A developer runs `uvx ruff check` and gets different lint results than a teammate who ran the exact same command on the same code.

**Diagnosis**: `uvx` (equivalent to `uv tool run`) resolves and runs the *latest* available version of the requested tool at the time it's invoked, in its own isolated, ephemeral-by-default environment — it is not pinned to any specific version by the project at all, unlike a `dev` dependency group entry in `pyproject.toml`. Two developers running `uvx ruff check` days apart, after a new `ruff` release, will genuinely get two different `ruff` versions, and therefore potentially different lint results, entirely as designed — this isn't a bug, it's exactly the trade-off of using a global tool invocation for something that actually needs to be pinned per-project.

```bash
# Confirm which ruff version uvx actually resolved
uvx ruff --version

# Compare against the project's pinned version
uv run ruff --version
```

**Fix**: this is precisely the Q4/Q13 decision surfacing as a real incident — `ruff` (and `mypy`, `pytest`) belong in the project's `dev` dependency group, invoked via `uv run ruff check`, not via `uvx`, specifically so every developer and CI run resolves the exact same pinned version from `uv.lock`. Reserve `uvx` for genuinely one-off, version-agnostic tool invocations (a quick `uvx cookiecutter <template>` to scaffold something once) where "always get the latest version" is actually the desired behavior (Ch 10, Ch 11).

### Exercise 4 — "A workspace member's tests pass locally but fail in CI with an import error for the shared package"

**Symptom**: `uv run pytest` inside `packages/api` passes locally, importing `expenseflow_shared` without issue, but the same command in CI raises `ModuleNotFoundError: No module named 'expenseflow_shared'`.

**Diagnosis**: Check first whether CI is running `uv sync` from the workspace *root* (which resolves and installs the whole workspace, including `packages/shared`, into a shared environment) versus running it from inside `packages/api`'s own directory only — depending on how the CI job is scripted, syncing only a single member's directory in isolation can produce an environment that doesn't have the workspace-linked sibling package installed at all, even though local development happened to always run from the root.

```bash
# From the workspace root — resolves and installs every member together
cd expenseflow-workspace && uv sync

# From inside a single member only — may not see workspace siblings depending on setup
cd expenseflow-workspace/packages/api && uv sync
```

**Fix**: update the CI workflow to run `uv sync` (and subsequent `uv run` commands) from the workspace root, matching the exact command shape used in local development, rather than `cd`-ing into an individual member first. As a general rule, any CI job for a workspace-based project should mirror the root-level command shape documented for local development exactly — divergence between the two is where this class of bug hides (Ch 12, Ch 15).

### Exercise 5 — "A `pip install`-based Dockerfile migrated to uv now fails with a missing `uv.lock` error"

**Symptom**: A team migrating an existing Dockerfile from `pip install -r requirements.txt` to `uv sync --frozen` gets an immediate build failure: `error: uv.lock not found`.

**Diagnosis**: `--frozen` (and `--locked`) both require an existing, committed `uv.lock` to sync against — a project migrated from `requirements.txt` that never had `uv lock` run against its new `pyproject.toml` simply doesn't have one yet, and `--frozen` deliberately refuses to generate one on the fly (that's what a bare `uv sync` would do, which is exactly the behavior `--frozen` exists to prevent in an automated context).

**Fix**: run `uv lock` locally (not in the Dockerfile) once, as a one-time step of the pip-to-uv migration, review the generated `uv.lock`, and commit it to version control — only then does `uv sync --frozen` in the Dockerfile have something to sync against. This is a good moment to also confirm the migration correctly translated every `requirements.txt` entry into `[project.dependencies]` first (Q15), since `uv lock`'s output is only as correct as the dependency declarations it's resolving (Ch 7, Ch 8, Ch 14).

---

## 6. Real-World Production Case Studies

The following are illustrative, composite scenarios grounded in ExpenseFlow's tooling journey from Chapters 1–19 — reflecting well-known uv-adoption and CI/CD process patterns, not a citation of a specific company's confidential incident — but each is a realistic, commonly-reported shape of production issue.

**The Docker image that got smaller, then quietly got bigger again.** When ExpenseFlow's team first migrated its Dockerfile from `pip install -r requirements.txt` to uv, the image shrank noticeably and builds sped up — the team was proud of the win and moved on. Months later, during an unrelated review of the CI pipeline's total runtime, someone noticed the production image had crept back up to nearly its old size. The cause turned out to be two small, independent regressions that had each landed separately and gone unnoticed: a new dev-only dependency (`ipdb`, added for local debugging convenience) had been added with a plain `uv add` instead of `uv add --dev`, and a later Dockerfile edit — made by someone unfamiliar with the original layering rationale — had accidentally reordered a `COPY` step so application source now landed before the dependency sync, which itself hadn't caused bloat directly but had made the team stop noticing dependency-layer cache misses as a signal something was off, since builds were "slow anyway" by that point. The fix was mechanical once found: move `ipdb` to the `dev` group, restore correct `COPY` ordering, and rebuild. The lesson: an initial optimization win doesn't stay a win by default — image size and layer-cache health are worth an occasional deliberate re-check, the same way a team periodically reviews dependency versions, rather than something proven once and assumed forever (Ch 14, Ch 18).

**The lockfile that diverged one merge at a time.** Two different engineers, in two different weeks, each added a small dependency to ExpenseFlow and correctly ran `uv add` locally, generating an updated `uv.lock` each time — but both PRs happened to be reviewed and approved without a reviewer specifically re-running `uv sync --locked` against the merged result, trusting that "CI was green on each individual PR" was sufficient. It was, individually — but a subtle interaction between the two added dependencies' own transitive requirements produced a combination CI had never actually tested together, since each PR's CI run only ever saw its own branch's lockfile in isolation. The result surfaced as an obscure `ImportError` in production, two weeks after both merges, tracked down only after a lot of confused debugging that started from the wrong assumption ("nothing changed recently" — when in fact two small, individually-fine changes had). The team's fix was a small process change, not a tooling change: a required, non-bypassable CI check on `main` itself (not just per-PR) that re-runs `uv sync --locked` and the full test suite after every merge, specifically so a bad interaction between two individually-fine changes is caught within minutes of the second merge landing, rather than being discovered as an unrelated-looking production bug weeks later. The lesson: per-PR CI validates a branch against the `main` it forked from, not the `main` that will actually exist once it merges — the same shape of gap this course's sibling Alembic course describes for a multi-head migration collision, just showing up as a dependency conflict instead of a schema conflict.

**The workspace that saved a schema-drift incident before it happened.** After splitting into `packages/api` and `packages/shared`, a developer updated a Pydantic schema in `packages/shared` to add a new required field, intending it only for an upcoming `packages/worker` feature that hadn't shipped yet. Because the workspace resolves and locks everything together, `uv run pytest` from the workspace root immediately failed several of `packages/api`'s *existing* tests — the API's request-validation logic was now unexpectedly rejecting payloads that no longer matched the updated shared schema. Caught locally, in minutes, before the change was even pushed. The developer's own reflection afterward was the most valuable part: had `expenseflow-shared` instead been a separately published, version-pinned package (as it might reasonably become later, per Chapter 16 and this chapter's system-design discussion), the same mistake would have been *possible* to make invisibly — `packages/api` simply wouldn't have picked up the new schema version until its own dependency was explicitly bumped, and the incompatibility might not have surfaced until someone did that bump much later, disconnected from the original change. The lesson: a workspace's tight coupling is not only a convenience, it's a deliberate consistency guarantee — trading looser, independently-versioned coupling for immediate, whole-graph feedback is exactly the trade-off Chapter 12 and this chapter's system-design answer describe, and this incident is the concrete shape of what that trade-off buys you when it matters.

---

## Real-World Scenario

A mock 45-minute uv/Python-tooling technical interview, structured the way a real onsite or virtual loop typically runs — rehearse this end-to-end, out loud, with a timer.

| Time | Segment | Pull from |
|---|---|---|
| 0:00 – 0:05 | Warm-up / background | Briefly describe your [Chapter 19](./19-capstone-projects.md) capstone project (Project 5, 6, or 7) and one design decision you'd defend |
| 0:05 – 0:15 | Rapid conceptual Q&A | Pick 4-5 from Section 1: e.g., Q2 (uv's resolver vs. pip's), Q3 (`lock`/`sync`/`run`), Q5 (when to use a workspace), Q8 (Dockerizing correctly), Q14 (trusted publishing) |
| 0:15 – 0:20 | One scenario/debugging question | Section 2, Scenario 2 ("production Docker image is way bigger than expected") — narrate your diagnostic order, not just the answer |
| 0:20 – 0:35 | Live configuration challenge | Section 3, Challenge 2 or 6 (multi-stage Dockerfile, or a test-gated release workflow) — write it from scratch without looking, then check against the model solution |
| 0:35 – 0:44 | System design | Section 4 (multi-service monorepo with shared internal libraries) — walk through clarifying questions, the workspace strategy, and when to promote a library out of it, in under 9 minutes |
| 0:44 – 0:45 | Your questions for the interviewer | Have two ready: e.g., "how do you currently decide when a shared internal library gets its own workspace member versus its own published package" or "what does your CI actually gate before a Docker image gets pushed" |

Time yourself strictly. If you run long on any segment, note which one — running long on conceptual Q&A at the expense of the system design segment is the single most common way candidates mismanage this format.

---

## Best Practices

- **Always state a trade-off, never just a choice** — "I'd use `--locked` in CI and `--frozen` in the Docker build, because CI should fail loudly on a stale lockfile while the image build should trust an already-validated commit" is a materially stronger answer than "I'd use `--locked`."
- **Answer conceptual questions with the definition-mechanism-trade-off shape**: one sentence defining the concept, one sentence on the underlying mechanism (what uv or the resolver actually does), and one sentence on when it breaks down or costs something — this keeps answers tight (30-60 seconds) without sounding rehearsed.
- **In scenario/debugging questions, narrate your diagnostic order out loud** — an interviewer evaluating a "CI passes but a laptop fails" or "the Docker image got bigger" question is watching *how* you isolate the cause (check the lockfile commit first; check `--no-dev` and `COPY` ordering before assuming the base image changed), not just whether you eventually guess right.
- **In system design questions, ask clarifying questions before designing** — how many services, what's actually being shared, whether release cadences differ across teams all change the right strategy, and asking first signals senior-level judgment rather than pattern-matching to a memorized architecture.
- **Ground every answer in a mechanism, not a memorized rule** — being able to explain *why* a hardlink doesn't survive a Docker `COPY` boundary (Q10), or *why* PubGrub can explain a conflict where pip's older resolver could only report one (Q2), is worth far more than reciting "use uv, it's faster" without being able to say why.
- **Have one real (or realistic capstone-based) war story ready** — most interviewers eventually ask "tell me about a dependency or build issue you've seen or can imagine," and a concrete, specific answer (even hypothetical, reasoned from first principles, like the Section 6 case studies) outperforms a generic answer every time.
- **Practice the configuration challenges by hand, not by memorizing solutions** — interviewers frequently tweak the problem statement slightly (a different dependency shape, a private index instead of PyPI, a different CI provider), specifically to see whether you understand the mechanism or memorized an answer.
- **Verify a claimed fix rather than asserting it** — in a troubleshooting question, actually naming the command you'd run (`uv lock --check`, `uvx ruff --version` vs. `uv run ruff --version`) to confirm the diagnosis before proposing the fix reads as far more credible than jumping straight to "the fix is X."

---

## Common Mistakes

- **Confusing "it's fast" with "I understand why it's fast"** — describing uv as simply "the fast one" misses the entire point of Chapters 2–3, and an interviewer will immediately probe with "fast compared to what, and why specifically" to check whether you actually understand the resolver and cache architecture underneath (Q1, Q2, Q10).
- **Forgetting that `--frozen` and `--locked` solve different problems** — proposing one where the other is clearly called for (using `--frozen` in CI, where you'd actually want the loud failure `--locked` provides for a stale lockfile) is a factual gap Exercise 1 shows can surface as a real incident (Q9).
- **Not distinguishing a project dependency from a global tool when asked directly** — recommending `uvx ruff` or a global `uv tool install` for a team's linter, when the whole point of a shared, pinned version is what a `dev` dependency group guarantees, signals a gap Exercise 3 shows causing real, confusing inconsistency (Q4, Q13).
- **Reaching for a workspace or a published package prematurely** — proposing workspace structure for a single-service project, or proposing a fully independent published package for code only ever consumed within one team's monorepo, both reach for more coordination machinery than the stated scope needs (Q5, System Design).
- **Treating a Docker image's size as fixed once "uv made it smaller"** — as the first Section 6 case study shows, an initial win from switching to uv doesn't stay a win without ongoing vigilance about `--no-dev`, layer ordering, and dependency-group scoping.
- **Skipping clarifying questions in system design and diving straight into an architecture** — this is the single most common signal of junior-level pattern-matching versus senior-level engineering judgment, and interviewers weight it heavily.
- **Assuming per-PR CI is sufficient to catch every dependency-conflict or schema-drift scenario** — the second and third Section 6 case studies both show a class of incident that individually-green PR checks cannot catch, requiring a check against `main`'s actual current state instead.
- **Cancelling or reinstalling an environment without first confirming what's actually out of sync** — jumping straight to "just delete `.venv` and reinstall everything" without first checking whether the lockfile itself is the actual problem risks masking the real root cause rather than fixing it (Scenario 1, Scenario 4).

---

## Summary

This course started with a single question — what problem does uv actually solve, and why does a Rust-based, standards-driven tool replace an entire cluster of older ones — and built outward one load-bearing layer at a time. Chapters 1–3 gave you the motivation and uv's internal architecture: the resolver, the global cache, and Python interpreter management. Chapters 4–6 made you fluent in Python version pinning, project creation, and the automatic virtual environment model. Chapters 7–9 covered dependency management, lock-file reproducibility, and the `uv run` execution model, including PEP 723 inline scripts. Chapters 10–11 drew a firm, practical line between project dev dependencies and global tools. Chapter 12 took the system into workspaces and monorepos. Chapters 13–16 took it into production integration: the full FastAPI/SQLAlchemy/Alembic developer loop, Docker, CI/CD, and publishing. Chapters 17–18 consolidated everything into a professional best-practices checklist and a catalog of known failure modes. Chapter 19 asked you to build seven real things, culminating in a published, CI-gated internal package. And this chapter, Chapter 20, rehearsed all of it under interview conditions — conceptual answers, scenario diagnosis, hands-on configuration writing, system design, troubleshooting, and production case studies.

You are now equipped to:

- **Explain uv's architecture and its resolver/cache model precisely**, and contrast it with pip+virtualenv's older model in terms of both speed and correctness, not just "it's faster"
- **Reason correctly about the lock/sync/run distinction and the `--frozen`/`--locked` split**, and know exactly which one belongs in CI versus a Docker build
- **Draw a firm, defensible line between a project dependency and a global tool**, and explain the concrete team-level cost of getting that line wrong
- **Design and defend a workspace strategy for a multi-service monorepo**, including when to promote a shared library out of the workspace entirely and publish it independently
- **Write correct `pyproject.toml`, Dockerfile, CI workflow, and PEP 723 artifacts from a plain-English problem statement**, not from memorized snippets
- **Diagnose a dependency conflict, a bloated Docker image, or a workspace import failure methodically**, working from the cheapest, most information-dense check outward rather than guessing
- **Talk about uv the way someone who has actually shipped a uv-managed project to production talks about it** — in terms of mechanisms and trade-offs, not memorized command syntax

Congratulations on completing the course. Go back to the [course index](./00-index.md) and check off every box in the Milestones Checklist from memory — if any box gives you pause, that's your last-mile study list before an interview, not a sign you need to redo the whole course. This is the full arc: from "what problem does uv actually solve?" to a professional capable of designing, wiring, testing, containerizing, and publishing a real Python application's tooling stack in front of a whiteboard. Good luck.

---

## Knowledge Check

Rate your confidence (1-5) on each of the following, honestly, before your next interview:

1. Can you explain, from memory and without notes, exactly why uv's resolver can produce a clearer conflict explanation than pip's historically weaker backtracking resolver?
2. Can you state the precise difference between `uv lock`, `uv sync`, and `uv run`, and explain why `--frozen` is the right choice in a Dockerfile while `--locked` is the right choice in CI?
3. Can you write a correct, layer-cached, `--no-dev` multi-stage Dockerfile and a test-gated release workflow from a plain-English requirement in under 15 minutes, without referring back to this chapter's solutions?
4. Can you explain exactly when a project dependency belongs in the `dev` group versus when it should be a global `uv tool`, and what concretely breaks for a team if that decision is made incorrectly?
5. Can you deliver a full system design answer (clarifying questions → workspace strategy → when to promote out of it) for a multi-service Python monorepo you've never designed before, out loud, in under 12 minutes, stating your assumptions as you go?

---

## Hands-On Exercise

Run a full mock interview against yourself:

1. **Pick 3 conceptual questions** from Section 1 (try to pick across different areas — e.g., one on the resolver/cache, one on the lock/sync/run distinction, one on Docker integration).
2. **Pick 2 configuration challenges** from Section 3 (include at least one you find genuinely uncomfortable, not just the easiest ones).
3. **Pick the system design question** from Section 4.

Answer all six out loud or in writing — with a timer, under realistic time pressure — **without looking at the model answers first**. Only after you've committed to your own answer, compare it against the model answer in this chapter and self-grade honestly against these criteria: Did you name the underlying mechanism, not just the term? Did you state at least one trade-off? For the configuration challenges, is your `pyproject.toml`/Dockerfile/workflow actually correct against the stated requirement, and did you choose the right flag/pattern for the stated constraint rather than just something that looks plausible? For the system design question, did you ask clarifying questions before designing, and did you address when a shared library should be promoted out of the workspace, not just how to set the workspace up initially?

Repeat this exercise with a fresh set of questions in a day or two — the goal isn't to memorize this chapter's specific answers, but to build the reflex of structuring any uv/tooling question, seen or unseen, the same disciplined way. If you have access to a local project (any of [Chapter 19](./19-capstone-projects.md)'s capstone projects work), go one step further for the configuration challenges: actually write and run the `pyproject.toml`/Dockerfile/workflow you produced, then verify it behaves the way you predicted (`uv sync --locked` should fail loudly on a deliberately stale lockfile, a correctly layered Docker build should show a cache hit on a source-only change) — running your own answer against a real project catches syntax mistakes and wrong assumptions that reading a model answer alone never will.

---

## Further Reading

- [uv Official Documentation](https://docs.astral.sh/uv/) — the official reference; the Concepts and Guides pages are the ones you'll return to most both in interviews and on the job.
- [uv Guides (Docker, CI/CD, Scripts)](https://docs.astral.sh/uv/guides/) — worked recipes for exactly the kind of Dockerfile, CI workflow, and PEP 723 challenges this chapter draws on.
- [uv Reference (CLI, settings, environment variables)](https://docs.astral.sh/uv/reference/) — the canonical reference for every flag (`--frozen`, `--locked`, `--no-dev`, `UV_LINK_MODE`) referenced throughout this chapter.
- [PEP 723 – Inline script metadata](https://peps.python.org/pep-0723/) — the specification underlying Q7 and Challenge 4.
- [PyPI Trusted Publishers (OIDC publishing)](https://docs.pypi.org/trusted-publishers/) — the mechanism behind Q14 and Challenge 6's token-free release workflow.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./19-capstone-projects.md">← Previous: Capstone Projects</a>
  <a href="./00-index.md">🏠 Index</a>
  <span></span>
</div>
