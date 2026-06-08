# Chapter 6 — Best Practices: Small, Fast, Secure Images

## 1. Introduction

By now you can build and run multi-service applications. The difference between a hobbyist
and a professional shows up in *how* those images are built. A naive image for a small Go
service can be 900 MB; a well-built one can be 8 MB. That 100× difference affects pull
times, storage costs, deploy speed, cold-start latency, and — crucially — **security
exposure**: every package in an image is a potential vulnerability, so a smaller image is
literally a smaller attack surface.

This chapter is the craft chapter. We cover the techniques that consistently produce small,
fast-building, secure images: **minimal base images**, **multi-stage builds**, **disciplined
layer caching**, **`.dockerignore`**, and a first serious pass at **security** (non-root
users, least privilege, secret handling, scanning, pinning). These aren't optional polish —
in production they're table stakes.

---

## 2. Learning Goals

By the end of this chapter you will be able to:

- Choose an appropriate **base image** (full vs slim vs alpine vs distroless) and justify
  the trade-offs.
- Write **multi-stage builds** that ship only the runtime artifact, not the build toolchain.
- Apply layer-caching discipline to keep rebuilds fast.
- Run containers as a **non-root user** and reduce privileges.
- Handle **secrets** correctly (and recognize the wrong ways).
- **Scan** images for vulnerabilities and **pin** dependencies for reproducibility.

---

## 3. Concepts Explained

### 3.1 Choosing a base image

The base image is the foundation, and it dominates size and security exposure.

| Base | Size (approx) | Trade-off |
|---|---|---|
| `ubuntu` / `python:3.12` (full) | large (100s of MB) | Familiar, has shells/tools; bloated, larger attack surface |
| `*-slim` (e.g. `python:3.12-slim`) | medium | Stripped of extras; good default for most apps |
| `alpine` (musl libc) | tiny | Smallest with a package manager; but musl≠glibc can break some wheels/binaries |
| **distroless** (Google) | tiny | Only your app + runtime, *no shell/package manager*; most secure, hardest to debug |
| `scratch` (empty) | ~0 | Nothing at all; for fully static binaries (Go/Rust) |

> **Default advice:** start with `-slim`. Move to alpine/distroless/scratch when you need
> the size/security and have validated compatibility. Beware alpine's musl libc surprises
> with some Python/Node native modules.

### 3.2 Multi-stage builds — the biggest single win

Most apps need a **build environment** (compilers, dev headers, full SDKs) that the
**runtime** doesn't. A multi-stage build uses one stage to build and a second, minimal stage
that copies only the finished artifact — discarding the entire toolchain.

```dockerfile
# ---- Stage 1: build ----
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/app

# ---- Stage 2: runtime (tiny) ----
FROM gcr.io/distroless/static:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

The final image contains *only* the compiled binary and a minimal runtime — often a few MB —
while the heavy `golang` SDK never ships. The same pattern applies to Node (build with full
image, copy `dist/` + production `node_modules` into slim) and Java (build a jar, copy into a
JRE-only image).

### 3.3 Layer caching discipline (recap + extend)

From Ch 3: order least- to most-frequently-changing, copy manifests before source. Extend
with:
- **Combine related `RUN` steps** and clean up in the same layer (`apt-get … && rm -rf
  /var/lib/apt/lists/*`) so the cleanup actually reduces image size (deleting in a *later*
  layer doesn't shrink the earlier one — the bytes are still in the lower layer).
- **Use `--no-install-recommends`** (apt) and `--no-cache` (apk) / `--no-cache-dir` (pip) to
  avoid dragging in extras and caches.

### 3.4 Security basics: the principle of least privilege

By default, containers run as **root** (UID 0) inside, and that root maps to a powerful user
on the host (unless user namespaces/rootless are in play — Ch 10). If an attacker
compromises a root container, they're far closer to compromising the host. So:

- **Run as a non-root user.** Create one in the Dockerfile and switch to it with `USER`.
- **Drop Linux capabilities** you don't need (`--cap-drop ALL`, add back only what's
  required).
- **Make the root filesystem read-only** (`--read-only`) where the app allows, mounting a
  `tmpfs` for any scratch paths.
- **Don't bake secrets into images** (they live forever in layers/history).

```dockerfile
FROM python:3.12-slim
RUN useradd --create-home --uid 10001 appuser
WORKDIR /app
COPY --chown=appuser:appuser requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY --chown=appuser:appuser . .
USER appuser
CMD ["python", "main.py"]
```

### 3.5 Secrets — the right and wrong ways

**Wrong:** `ENV API_KEY=...`, `ARG TOKEN=...`, or `COPY .env` — all persist in image layers
and metadata, readable by anyone with the image.

**Right (at build time):** BuildKit secret mounts, which are available during a `RUN` but
never stored in a layer:
```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```
```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
```

**Right (at run time):** inject via environment from a secrets manager, mounted files, or
orchestrator secrets (Docker/K8s secrets) — never committed to the image. Full treatment in
Ch 8.

### 3.6 Scanning and pinning

- **Scan** images for known CVEs (`docker scout`, Trivy, Grype) and fail builds on critical
  findings.
- **Pin** the base image by digest and lock application dependencies (lockfiles) so builds
  are reproducible and auditable. Generate an **SBOM** (software bill of materials) so you
  know exactly what's inside.

---

## 4. Internal Working / Deep Dive

### 4.1 Why deleting files in a later layer doesn't shrink the image

Layers are additive diffs (Ch 3/4). If layer 3 adds a 200 MB cache and layer 5 deletes it,
the bytes still exist in layer 3 — layer 5 just records a "whiteout" marking them removed in
the merged view. The image still ships all layers, so total size is unchanged. **You must
add and remove within the *same* `RUN`** to avoid persisting the bytes. This single fact
explains a huge share of accidentally-bloated images.

### 4.2 How multi-stage builds discard weight

Each `FROM` starts a fresh stage with its own layers. Only the **final stage** becomes the
output image; earlier stages exist solely to produce artifacts you `COPY --from`. BuildKit
is smart enough to skip building stages whose outputs aren't used. So the multi-GB build
toolchain genuinely never lands in your shipped image — it's a separate, discarded stage.

### 4.3 What "non-root" actually buys you

Container root is constrained (reduced capabilities, seccomp), but it's still root, and many
container escapes and privilege-escalation chains start from in-container root. Running as an
unprivileged UID means a compromised process can't write system paths, can't bind privileged
ports (<1024), and has far fewer escalation primitives. Combined with `--cap-drop ALL`,
`--read-only`, and a seccomp profile, you've turned a soft target into a hard one — defense
in depth (expanded in Ch 10).

### 4.4 Image size's hidden costs

Size isn't vanity. Larger images mean: slower registry pulls (worse autoscaling and cold
starts), more storage across every node and registry, longer CI, and more packages = more
CVEs to track and patch. A distroless image with no shell also denies attackers the tools
they'd use post-compromise. Small images are a performance *and* security strategy.

---

## 5. Examples

### Example 1 — Node multi-stage (full build → slim runtime)

```dockerfile
# ---- build ----
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ---- runtime ----
FROM node:20-slim
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build /app/dist ./dist
RUN useradd --uid 10001 appuser
USER appuser
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Example 2 — Clean apt layer (add + clean in one RUN)

```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl \
 && rm -rf /var/lib/apt/lists/*
```

### Example 3 — Pin base by digest + add provenance labels

```dockerfile
FROM python:3.12-slim@sha256:<digest-here>
LABEL org.opencontainers.image.source="https://github.com/myorg/api" \
      org.opencontainers.image.version="1.4.0"
```

### Example 4 — Scan before shipping

```bash
docker build -t myorg/api:1.4 .
docker scout cves myorg/api:1.4        # or: trivy image myorg/api:1.4
# fail the pipeline if criticals are found (Ch 8 wires this into CI)
```

### Example 5 — Run hardened

```bash
docker run -d --name api \
  --user 10001:10001 \
  --cap-drop ALL \
  --read-only --tmpfs /tmp \
  --security-opt no-new-privileges \
  -p 3000:3000 myorg/api:1.4
```

---

## 6. Real-World Use Cases

- **Faster autoscaling:** a 20 MB image pulls and starts far quicker than a 1 GB one, so
  scale-out and cold starts are snappier — critical under traffic spikes.
- **Lower cloud bills:** smaller images mean less registry storage and less data egress
  across a fleet.
- **Audit & compliance:** pinned digests + lockfiles + SBOM + scanning produce an auditable,
  reproducible supply chain.
- **Hardened multi-tenant platforms:** non-root, read-only, capability-dropped containers
  reduce blast radius if one tenant's app is compromised.
- **Secure CI:** BuildKit secret mounts let CI use private registry/npm credentials without
  leaking them into published images.

---

## 7. Common Mistakes

- **Using a full base image** (`python:3.12`, `node:20`) for runtime when slim/distroless
  would do — needless size and CVEs.
- **Shipping the build toolchain** because you didn't use multi-stage — compilers and SDKs in
  the runtime image.
- **Deleting caches in a later layer** and expecting the image to shrink (it won't — same
  `RUN` only).
- **Baking secrets** via `ENV`/`ARG`/`COPY .env` — permanently embedded in layers/history.
- **Running as root by default** and never adding a `USER`.
- **Not pinning** base images/dependencies, so "the same build" drifts and CVEs sneak in.
- **Skipping scans,** then deploying images with known critical vulnerabilities.
- **Blindly switching to alpine** and getting cryptic musl/glibc failures with native
  modules.

---

## 8. Best Practices

- **Default to slim bases; graduate to distroless/scratch** when validated.
- **Always multi-stage** for compiled or build-step languages; copy only the artifact.
- **Add+clean in one `RUN`;** use `--no-install-recommends`/`--no-cache(-dir)`.
- **Order for cache hits** (manifests → install → source).
- **Create and switch to a non-root `USER`;** `COPY --chown` to set ownership.
- **Pin base images by digest; lock app deps; generate an SBOM.**
- **Scan in CI and gate on severity.**
- **Handle secrets via BuildKit mounts (build) and orchestrator/env injection (run)** —
  never in layers.
- **Run with `--cap-drop ALL`, `--read-only` + tmpfs, `--security-opt no-new-privileges`**
  where feasible.
- **Add OCI provenance labels** so images are traceable to source and version.

---

## 9. Hands-On Exercise

**Goal:** take a bloated image and make it small, fast, and hardened.

1. **Baseline.** Build any app naively (full base, `COPY . .` then install, run as root).
   Record the image size (`docker images`) and inspect with `docker history` to find the
   biggest layers.

2. **Slim + cache order.** Switch to a `-slim` base and reorder so dependencies install
   before source is copied. Rebuild; record the new size and confirm cache reuse on a
   source-only change.

3. **Multi-stage.** Convert to a multi-stage build that copies only the runtime artifact.
   Record the size again — aim for a dramatic drop. Note the before/after numbers.

4. **Clean layers.** Find any `RUN` that installs then deletes across layers; fold it into a
   single `RUN`. Confirm the size effect with `docker history`.

5. **Harden.** Add a non-root `USER`, run with `--cap-drop ALL --read-only --tmpfs /tmp
   --security-opt no-new-privileges`, and confirm the app still works. If it breaks, diagnose
   which privilege it needed and add back the minimum.

6. **Scan.** Run `docker scout cves` (or Trivy) on both the baseline and final images and
   compare the vulnerability counts.

**Deliverable:** a small table — image variant vs size vs vulnerability count — and 2–3
sentences on which change had the biggest impact and why.

---

## 10. Quiz Questions

1. Why is a smaller image a security benefit, not just a performance one?
2. Explain a multi-stage build and what it keeps out of the final image.
3. You `RUN apt-get install …` in one layer and `RUN rm -rf /var/cache …` in the next. Did
   the image shrink? Why or why not?
4. Give two reasons to run a container as a non-root user.
5. Why is `ENV API_KEY=…` a bad way to provide a secret? Give one correct alternative for
   build time and one for run time.
6. What does pinning the base image by digest (plus lockfiles + SBOM) buy you?
7. When might alpine cause problems despite its small size?
8. What's the purpose of `--security-opt no-new-privileges`?

<details>
<summary>Answer key</summary>

1. Fewer packages means fewer known vulnerabilities to track/patch and a smaller attack
   surface; minimal images (e.g. distroless) also lack a shell/tools attackers would use
   post-compromise.
2. Use a heavy "build" stage to compile/produce an artifact, then a minimal final stage that
   `COPY --from` only the artifact — the compilers/SDKs/build deps never ship.
3. No. The installed bytes still live in the earlier layer; the later `rm` only adds a
   whiteout. You must install and clean within the same `RUN`.
4. A compromised non-root process can't write system paths or bind privileged ports and has
   far fewer privilege-escalation primitives, reducing blast radius.
5. It's baked into image layers/history forever, readable by anyone with the image. Build
   time: BuildKit `--mount=type=secret`. Run time: inject via env/file from a secrets
   manager or orchestrator secret.
6. Reproducible, auditable builds: the exact base content is fixed, dependencies are locked,
   and the SBOM enumerates everything inside for vulnerability tracking and compliance.
7. Alpine uses musl libc instead of glibc, which can break prebuilt binaries/wheels and some
   native modules (Python/Node), causing cryptic runtime failures.
8. It prevents processes in the container from gaining new privileges (e.g. via setuid
   binaries), closing a common escalation path.
</details>

---

## 11. Chapter Summary

- The base image dominates size and exposure: **default to slim, graduate to
  distroless/scratch** when validated; beware alpine's musl quirks.
- **Multi-stage builds** are the biggest single win — ship only the runtime artifact, not the
  toolchain.
- Layers are additive: **add and clean in the same `RUN`**; deleting later doesn't shrink the
  image.
- Security basics = **least privilege**: non-root `USER`, `--cap-drop ALL`, `--read-only` +
  tmpfs, `no-new-privileges`.
- **Never bake secrets**; use BuildKit secret mounts (build) and injected env/files (run).
- **Pin** base digests, **lock** dependencies, **scan** for CVEs, and produce an **SBOM** for
  a reproducible, auditable supply chain.

Next: **Chapter 7 — Advanced Topics**, where we go deep on network drivers, storage drivers,
advanced Compose patterns, and production-grade healthchecks.

---

## 12. Further Reading

- Docker docs: "Best practices for building images" and "Multi-stage builds."
- Google "distroless" project; "scratch" usage for static binaries.
- Docker Scout, Trivy, and Grype documentation; SBOM formats (SPDX, CycloneDX).
- BuildKit "build secrets" documentation.
- CIS Docker Benchmark (a thorough security checklist).
