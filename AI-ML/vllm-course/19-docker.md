# Docker

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain why `vllm/vllm-openai` is the right default artifact for most deployments — what it bundles, and why
  building your own vLLM+CUDA image from scratch is usually wasted effort
- Explain what the NVIDIA Container Toolkit does, why a container can't see a GPU without it, and launch
  `vllm/vllm-openai` with `docker run --gpus all ...` against a real model
- Mount a host directory (or named volume) onto the container's Hugging Face cache path so multi-gigabyte model
  weights survive a container restart instead of re-downloading every time
- Pass `HF_TOKEN` (or `HUGGING_FACE_HUB_TOKEN`) into a container correctly for gated/private model repos, and
  explain why it must never be baked into a committed `Dockerfile`
- Check host driver/CUDA compatibility with `nvidia-smi` before assuming a "works on my machine" container
  failure is a vLLM bug
- Wire the confirmed, unauthenticated `/health` endpoint into a Docker `HEALTHCHECK` instruction, and explain how
  that same endpoint feeds a Kubernetes readiness/liveness probe (Chapter 20)
- Distinguish container-level memory limits (host RAM, `docker run --memory`) from GPU VRAM governed by
  `--gpu-memory-utilization` (Chapter 10) — and explain why setting one does nothing to the other
- Assemble a complete, production-shaped `docker run` command: GPU passthrough, a cache volume, `HF_TOKEN`, port
  mapping, and `--api-key`

---

## Prerequisites for This Chapter

This chapter assumes the general Docker fluency the course index calls out as a prerequisite: you already know
what an image is, what a container is, how `docker run`, `-v`, `-p`, and `-e` work, and the difference between a
bind mount and a named volume. None of that is re-taught here — this chapter is entirely about what's
**specific** to running vLLM and GPU workloads in a container, not Docker fundamentals in general.

It also builds directly on two earlier chapters:

- **Chapter 4 (The OpenAI-Compatible Server)** — the `vllm serve` flags this chapter's `docker run` examples pass
  as the container's command (`--api-key`, `--served-model-name`, etc.) and the confirmed endpoint table
  (`/v1/chat/completions`, `/health`, `/metrics`, and which ones `--api-key` gates) are assumed background here.
- **Chapter 10 (Memory Management)** — `--gpu-memory-utilization`, `--max-model-len`, and `--max-num-seqs` are
  assumed known. This chapter's memory section is specifically about *not confusing* those flags with Docker's
  own `--memory` flag, not about re-deriving what they do.

This chapter does not re-explain what an image, layer, volume, or `docker run` flag is in general — only what
changes when the workload behind the container is a multi-gigabyte GPU-resident model server.

---

## 1. Why a Prebuilt Image, Not a Bespoke Build

A working vLLM installation is not just a `pip install`. It's a specific PyTorch build, a matching CUDA toolkit,
compiled CUDA/C++ kernels for attention and quantization, and a set of Python dependencies that all have to agree
with each other and with the host's GPU driver. Getting all of that to line up by hand, in your own `Dockerfile`,
is real work — and it's work the vLLM project has already done for you and tests continuously.

The official image, **`vllm/vllm-openai`** (fact sheet §14), bundles:

- The vLLM engine itself (the V1 engine covered throughout this course)
- The OpenAI-compatible server (Chapter 4) as the image's default entrypoint — the container's `CMD` is set up so
  that arguments you pass to `docker run` after the image name are forwarded to `vllm serve`
- A matching CUDA toolkit and pre-compiled CUDA kernels for the attention backends, quantization kernels, and
  other GPU-side code this course has covered (PagedAttention's block-structured kernels in Chapter 7,
  quantization kernels in Chapter 13, etc.)

The practical upshot: for the overwhelming majority of deployments, you do not build your own vLLM+CUDA image.
You pull `vllm/vllm-openai`, mount what needs to persist (Section 4), pass what needs to be configured (env vars,
`vllm serve` flags), and run it. Reaching for a custom `Dockerfile` that reinstalls vLLM from source makes sense
only for real edge cases — a patched/forked vLLM build, an unsupported hardware backend, or a security policy
that forbids pulling third-party images at all — and even then, the usual pattern is to start `FROM
vllm/vllm-openai:<tag>` and layer your changes on top, rather than starting from a bare CUDA base image and
reinstalling everything vLLM's maintainers already solved.

> **Version-illustration note:** this chapter shows concrete tags like `vllm/vllm-openai:v0.26.0` purely as
> illustrations of the pattern. vLLM ships roughly every two weeks (course-wide convention, fact sheet §0/§1) —
> check `hub.docker.com/r/vllm/vllm-openai/tags` or the GitHub releases page for the current tag before pulling
> for anything beyond a one-off experiment.

---

## 2. The NVIDIA Container Toolkit

### 2.1 Why a container can't see a GPU by default

A Docker container is, by default, isolated from the host's devices. That isolation is exactly what makes
containers portable — and exactly what stands between a container and a physical GPU. A GPU isn't just a device
file; using it correctly requires the container's process to talk to the **host's** NVIDIA kernel driver, which
means the container needs the right device nodes exposed *and* a matching set of host driver libraries visible
inside its filesystem, without you manually copying driver `.so` files into every image (which would break the
instant the host updates its driver).

The **NVIDIA Container Toolkit** is the piece that solves this. It's a runtime hook that:

- Exposes the host's GPU device nodes into the container
- Bind-mounts the host's NVIDIA driver libraries into the container at runtime (not baked into the image), so the
  container's CUDA toolkit (whatever version it was built against) talks to the driver the host actually has
- Is invoked automatically when you pass `--gpus all` (or a more specific `--gpus` selector) to `docker run`

This is **not vLLM-specific** — it's the standard prerequisite for running *any* CUDA-based container on Docker,
the same toolkit you'd need for a PyTorch, TensorFlow, or CUDA-samples container. But it's necessary in practice
for every GPU-based vLLM deployment, so it's worth confirming explicitly rather than assuming a fresh host has it
installed.

### 2.2 Installing it

Installation is a host-level, one-time setup step (not something you do inside a Dockerfile) — follow NVIDIA's
current installation guide for your host OS and Docker version (Further Reading has the link). After installing
it, the toolkit registers itself with the Docker daemon so `--gpus` becomes a recognized `docker run` flag.

A quick sanity check that the toolkit is correctly wired up, before touching vLLM at all:

```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

If this prints the same GPU table you'd see running `nvidia-smi` directly on the host, GPU passthrough is
working and you're ready to run `vllm/vllm-openai`. If it fails — an error about no GPUs found, or the toolkit
runtime not being recognized — that's a host/toolkit configuration problem to fix *before* debugging anything
vLLM-specific, since vLLM (or any CUDA container) cannot work around a container that genuinely can't see a GPU.

### 2.3 Launching vLLM with GPU passthrough

```bash
docker run --gpus all \
  -p 8000:8000 \
  vllm/vllm-openai:v0.26.0 \
  --model Qwen/Qwen2.5-1.5B-Instruct \
  --dtype auto
```

- `--gpus all` is what invokes the NVIDIA Container Toolkit and exposes every GPU on the host to the container.
  You can also scope it to specific devices (e.g. `--gpus '"device=0,1"'`) — the same selection semantics you'd
  use with any CUDA container, not something vLLM changes.
- `-p 8000:8000` maps the container's default server port (Chapter 4) to the host.
- Everything after the image name — `--model ... --dtype auto` — is forwarded to `vllm serve` inside the
  container, because the image's entrypoint is the server, not a shell.

This is a bare-minimum example deliberately kept small to isolate the GPU-passthrough concept. Section 9 builds
the complete, production-shaped version with a cache volume, credentials, and auth.

---

## 3. Model Cache Volumes — Don't Re-Download Multi-GB Weights Every Restart

### 3.1 The problem

Inside the container, vLLM (via the underlying `huggingface_hub` library) downloads model weights to a cache
directory under the container filesystem's home directory — the same `~/.cache/huggingface` location it would
use on a bare-metal host. A container's filesystem, by default, is ephemeral: when the container is removed
(`docker rm`, or replaced by a new deployment), anything written inside it — including that multi-gigabyte cache
— is gone with it. Restart the container, or replace it during a redeploy, and vLLM re-downloads the entire model
from the Hugging Face Hub from scratch, even though nothing about the model changed.

For a small model this is an annoying delay. For a 70B-class checkpoint at dozens of gigabytes, this is minutes
of wasted startup time, repeated egress bandwidth, and — if you're iterating on container config and restarting
frequently during setup — a genuinely painful development loop.

### 3.2 The fix: mount a host path onto the cache directory

```bash
docker run --gpus all \
  -v /path/on/host/hf-cache:/root/.cache/huggingface \
  -p 8000:8000 \
  vllm/vllm-openai:v0.26.0 \
  --model Qwen/Qwen2.5-1.5B-Instruct
```

Mounting a host directory (or a named Docker volume, if you'd rather not manage a host path directly) onto the
container's `~/.cache/huggingface` means the downloaded weights live outside the container's own writable layer.
The first run downloads the model as before; every subsequent run — even against a freshly `docker run`-created
container — finds the weights already present in the mounted cache and skips the download entirely, going
straight to loading them onto the GPU.

> **Note on the home directory path.** The exact home directory inside `vllm/vllm-openai` depends on which user
> the container process runs as (commonly `root`, i.e. `/root/.cache/huggingface`, in this image as of this
> writing). Confirm the actual path for your installed image tag — e.g. by running `docker run --rm --entrypoint
> env vllm/vllm-openai:<tag>` to inspect `HOME`, or checking the image's Dockerfile in the vLLM repo — rather than
> assuming it never changes across releases.

This is an easy detail to miss precisely because it's invisible when it's wrong: a container without the cache
volume mounted works *correctly*, just slowly and repeatedly, and nothing in the logs screams "you're
re-downloading 15GB you already have." It only becomes visible as an annoyingly slow restart loop or an
unexpectedly large egress bill.

### 3.3 Named volume alternative

If you'd rather not manage a host directory path directly (common in CI or when the host filesystem layout isn't
yours to control), a named volume achieves the same persistence:

```bash
docker volume create vllm-hf-cache

docker run --gpus all \
  -v vllm-hf-cache:/root/.cache/huggingface \
  -p 8000:8000 \
  vllm/vllm-openai:v0.26.0 \
  --model Qwen/Qwen2.5-1.5B-Instruct
```

Functionally identical to the bind mount for this purpose — the choice between a bind mount and a named volume
here is the same general Docker trade-off you already know (host-path visibility/portability vs. Docker-managed
lifecycle), not something specific to vLLM.

---

## 4. HuggingFace Credentials — `HF_TOKEN` Without Baking Secrets Into the Image

Gated model repos (many Llama, Gemma, and other license-gated checkpoints on the Hugging Face Hub) and private
repos require an authenticated download. The standard way to supply that credential to any Hugging Face
Hub–based tool, vLLM included, is an environment variable — `HF_TOKEN` (the current standard name) or
`HUGGING_FACE_HUB_TOKEN` (an older, still-recognized alias) — read by the `huggingface_hub` library at download
time.

```bash
docker run --gpus all \
  -v vllm-hf-cache:/root/.cache/huggingface \
  -e HF_TOKEN=hf_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
  -p 8000:8000 \
  vllm/vllm-openai:v0.26.0 \
  --model meta-llama/Llama-3.1-8B-Instruct
```

`-e HF_TOKEN=...` passes the token into the container's environment at `docker run` time, which vLLM's download
path picks up the same way it would if you'd exported it in a shell before running vLLM bare-metal.

### 4.1 The security-hygiene rule this section exists to state

**Never bake a token into a committed `Dockerfile`** — that is, never write something like:

```dockerfile
# Do not do this
ENV HF_TOKEN=hf_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

An `ENV` instruction becomes part of the image's layer history. Anyone who can pull or inspect that image —
including, if it's ever pushed to a registry by mistake, anyone with registry read access — can extract the
token with `docker history` or by inspecting the layer, regardless of whether the running container later
overrides the variable. Baking a secret into an image is functionally equivalent to committing it to source
control: it doesn't matter that the value isn't shown in a `docker inspect` of a *running* container if the
image itself already contains it.

The credential belongs at **run time**, not **build time**:

- Locally or in simple deployments: `-e HF_TOKEN=...` at `docker run`, as above, ideally sourced from a shell
  variable or a `.env` file that is itself gitignored, not typed into a shared shell history.
- In production: a secrets manager appropriate to your orchestrator — a Kubernetes `Secret` mounted as an env var
  (Chapter 20 goes deeper on this), a cloud provider's secrets service (AWS Secrets Manager, GCP Secret Manager,
  etc.), or an equivalent — injected into the container's environment at deploy time, never present in any
  version-controlled file at all.

The underlying principle is the same one you already apply to any other credential (a database password, a
third-party API key) — this section exists only to flag that model-serving containers are not a special case
just because the "secret" is a Hugging Face token rather than a database credential.

---

## 5. CUDA / Driver Compatibility

### 5.1 Why "works on my machine" container failures happen

A CUDA container ships a specific CUDA toolkit version baked into its layers — for `vllm/vllm-openai`, whatever
CUDA version that particular image tag was built against. The **host's NVIDIA driver**, by contrast, lives
outside the container entirely and is whatever version was installed on that machine. The NVIDIA Container
Toolkit (Section 2) bridges the two by exposing the host driver into the container, but it cannot make an
incompatible pairing work — the host driver has to be new enough to support the CUDA version the container was
built against. An older driver paired with a container built against a newer CUDA toolkit than the driver
supports is a genuine incompatibility, not a configuration bug the toolkit can paper over.

This is the single most common source of "it ran fine on one GPU box and immediately failed on another" — two
hosts with the same Docker setup and the same image tag, but different driver versions, one of which is too old
for the CUDA version inside the image.

### 5.2 Checking compatibility before you debug anything else

Run `nvidia-smi` **on the host** (outside any container):

```bash
nvidia-smi
```

The output header reports both the installed driver version and the **maximum CUDA version that driver
supports** — this is the number to compare against the CUDA version baked into your chosen `vllm/vllm-openai`
tag. If the image's CUDA version exceeds what the host driver supports, that's your failure, and no amount of
vLLM-level flag tuning fixes it — the fix is either upgrading the host's NVIDIA driver or choosing/building an
image tagged against an older, driver-compatible CUDA version.

> **Verify against current docs**: exact CUDA-version-to-minimum-driver-version tables change as NVIDIA ships new
> CUDA releases and vLLM's published images move to newer CUDA toolkits. Treat `nvidia-smi`'s reported "max
> supported CUDA version" line as the authoritative, host-specific check, rather than memorizing a
> version-compatibility table that will drift.

When a container fails immediately on GPU initialization (rather than, say, mid-generation), checking this
pairing is the fastest first diagnostic step — faster than assuming it's a vLLM bug, a bad model checkpoint, or a
config mistake in your `docker run` command.

---

## 6. Health Checks — Wiring `/health` Into Docker and Orchestrators

Chapter 4 confirmed the server's `/health` endpoint: unauthenticated, a simple liveness signal ("is the process
up and able to answer"), deliberately not gated by `--api-key` so infrastructure-internal callers don't need an
application secret just to ask if the process is alive.

### 6.1 A Docker `HEALTHCHECK`

If you're building your own thin image `FROM vllm/vllm-openai:<tag>` (Section 1) or otherwise controlling the
Dockerfile, you can wire `/health` in directly:

```dockerfile
FROM vllm/vllm-openai:v0.26.0

HEALTHCHECK --interval=30s --timeout=5s --start-period=120s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
```

- `--start-period=120s` (or longer, depending on model size) matters more here than in a typical web-service
  image: vLLM's startup includes loading multi-gigabyte weights onto the GPU and CUDA graph capture (Chapter 6's
  and Chapter 9's territory) before the server can answer at all — a health check with too short a grace period
  will report the container unhealthy while it's still legitimately loading, not because anything is wrong.
- Because `/health` requires no `Authorization` header, the `curl` call above needs no credential — one less
  thing to wire through the healthcheck definition.

If you're running the stock `vllm/vllm-openai` image unmodified (no custom Dockerfile), the equivalent healthcheck
can be attached at `docker run` time instead:

```bash
docker run --gpus all \
  --health-cmd="curl -f http://localhost:8000/health || exit 1" \
  --health-interval=30s \
  --health-timeout=5s \
  --health-start-period=120s \
  --health-retries=3 \
  -p 8000:8000 \
  vllm/vllm-openai:v0.26.0 \
  --model Qwen/Qwen2.5-1.5B-Instruct
```

### 6.2 Foreshadowing Kubernetes

`docker run --health-cmd`/`HEALTHCHECK` is the single-container-runtime equivalent of what a Kubernetes
**readiness probe** and **liveness probe** do at the orchestrator level — both fundamentally poll the same
`/health` endpoint on an interval and act on the result (Docker marks the container `unhealthy`; Kubernetes stops
routing traffic to a not-ready pod, or restarts a pod that fails liveness). Chapter 20 (Production Serving) covers
wiring `/health` into an actual Kubernetes `readinessProbe`/`livenessProbe` and `/metrics` into a Prometheus
scrape config — the concept introduced here (an unauthenticated, cheap, "is this process alive" endpoint) is
exactly what both mechanisms are built on, whether the orchestrator is `docker run`'s own health-check machinery
or a full Kubernetes deployment.

---

## 7. Container Memory vs. GPU VRAM — Two Different Limits Governed by Two Different Knobs

This is a subtle point worth stating explicitly, because the two concepts are easy to conflate and the failure
mode when you do is confusing rather than obviously wrong.

### 7.1 What `docker run --memory` actually limits

Docker's `--memory` flag (and the related `--memory-swap`, `--memory-reservation`) constrains the container's use
of **host system RAM** — ordinary CPU-addressable memory, enforced via the Linux kernel's cgroups. This is the
memory used by the server process's CPU-side work: the Python process itself, request/response handling and
JSON (de)serialization, tokenization (turning input text into token IDs and back, Chapter 1's territory), and any
CPU-side buffering the frontend does before/after a request touches the GPU.

### 7.2 What actually governs GPU memory

The model's weights, the KV cache (Chapter 6), and activation memory all live in **GPU VRAM**, a completely
separate memory space from host RAM, and Docker's `--memory` flag has **no visibility into or control over it
whatsoever**. The knob that governs GPU memory usage is `--gpu-memory-utilization` (Chapter 10) — the vLLM engine
flag that controls what fraction of the GPU's total VRAM the engine is allowed to reserve for weights, KV cache
blocks, and activations.

### 7.3 Why conflating them produces confusing failures

Set `docker run --memory 8g` on a container running a model whose weights alone are 16GB in VRAM, and nothing
about VRAM usage changes — the container simply may not have run out of the RAM limit yet at all, because the
16GB of GPU memory was never counted against that 8GB host-RAM ceiling in the first place. Conversely, an
`--gpu-memory-utilization` set too high for the actual GPU (Chapter 10's OOM-diagnosis territory) causes a CUDA
out-of-memory error that has nothing to do with any `--memory` value you set on the container — because that
failure happens entirely inside the GPU's own memory manager, a resource Docker's cgroup-based memory accounting
doesn't touch.

The practical rule: size `docker run --memory` (if you set it at all) for the CPU-side footprint of the server
process — tokenizer overhead, request queuing, response buffers — a modest, roughly constant amount that doesn't
scale with model size the way VRAM does. Size GPU memory usage entirely through `--gpu-memory-utilization`,
`--max-model-len`, and `--max-num-seqs` (Chapter 10), independent of whatever host-RAM limit you did or didn't set
on the container.

---

## 8. Putting It Together — Container to GPU

```mermaid
flowchart TD
    A["docker run --gpus all vllm/vllm-openai ..."] --> B["Docker Engine"]
    B --> C["NVIDIA Container Toolkit<br/>(runtime hook)"]
    C --> D["Host NVIDIA Driver<br/>(bind-mounted into container)"]
    D --> E["Physical GPU(s)"]

    F["Host directory or named volume"] -.->|"-v ...:/root/.cache/huggingface"| G["vLLM container filesystem"]
    G -->|"weights cached, skip re-download"| H["vLLM engine loads weights onto GPU"]
    H --> E

    I["-e HF_TOKEN=... (run time only)"] -.->|"never baked into image"| G
```

Two paths converge on the same container: the **compute path** (top) — `--gpus all` triggers the NVIDIA
Container Toolkit, which is what actually lets the containerized vLLM process talk to the host's real GPU
hardware through the host's driver — and the **data/persistence path** (bottom) — a mounted cache volume and a
run-time-injected token, both of which exist specifically so the container's ephemeral filesystem doesn't force a
multi-gigabyte re-download or a leaked credential.

---

## 9. Full Worked Example

Putting every piece from this chapter together — GPU passthrough, a persistent model cache, a gated-model token,
port mapping, and server authentication (Chapter 4):

```bash
docker run -d \
  --name vllm-server \
  --gpus all \
  --restart unless-stopped \
  -v vllm-hf-cache:/root/.cache/huggingface \
  -e HF_TOKEN="${HF_TOKEN}" \
  -p 8000:8000 \
  --health-cmd="curl -f http://localhost:8000/health || exit 1" \
  --health-interval=30s \
  --health-timeout=5s \
  --health-start-period=120s \
  --health-retries=3 \
  vllm/vllm-openai:v0.26.0 \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --dtype auto \
  --api-key "${VLLM_API_KEY}" \
  --served-model-name llama3.1-8b \
  --gpu-memory-utilization 0.9 \
  --max-model-len 8192
```

Walking through what each piece is doing and why, tying back to the relevant section:

| Piece | Purpose | Section |
|---|---|---|
| `--gpus all` | Exposes host GPU(s) to the container via the NVIDIA Container Toolkit | 2 |
| `-v vllm-hf-cache:/root/.cache/huggingface` | Persists downloaded weights across container restarts | 3 |
| `-e HF_TOKEN="${HF_TOKEN}"` | Authenticates the gated `meta-llama/...` download, sourced from the shell/CI secret store at run time, never baked into the image | 4 |
| `-p 8000:8000` | Maps the OpenAI-compatible server's port (Chapter 4) to the host | — |
| `--health-cmd=...` | Wires the confirmed `/health` endpoint into Docker's own health tracking | 6 |
| `--model`, `--dtype`, `--served-model-name` | Ordinary `vllm serve` flags (Chapter 4), forwarded by the image's entrypoint | 1, Ch. 4 |
| `--api-key "${VLLM_API_KEY}"` | Gates `/v1/*` (Chapter 4) — note this secret is *also* sourced from the environment at run time, the same discipline as `HF_TOKEN` | 4, Ch. 4 |
| `--gpu-memory-utilization`, `--max-model-len` | GPU VRAM sizing — deliberately *not* a Docker `--memory` value (Section 7) | 7, Ch. 10 |

Note that both secrets in this command (`HF_TOKEN` and `VLLM_API_KEY`) are referenced as shell variables
(`"${HF_TOKEN}"`), not literal values typed into the command — the same run-time-injection discipline Section 4
describes, extended to the server's own API key.

### 9.1 A `docker-compose.yml` equivalent

For local development or single-host deployments, the same configuration expressed as Compose is often easier to
version-control and reproduce than a long `docker run` invocation:

```yaml
services:
  vllm:
    image: vllm/vllm-openai:v0.26.0
    restart: unless-stopped
    ports:
      - "8000:8000"
    volumes:
      - vllm-hf-cache:/root/.cache/huggingface
    environment:
      - HF_TOKEN=${HF_TOKEN}
    command:
      - --model
      - meta-llama/Llama-3.1-8B-Instruct
      - --dtype
      - auto
      - --api-key
      - ${VLLM_API_KEY}
      - --served-model-name
      - llama3.1-8b
      - --gpu-memory-utilization
      - "0.9"
      - --max-model-len
      - "8192"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 5s
      start_period: 120s
      retries: 3
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

- `HF_TOKEN` and `VLLM_API_KEY` are read from the shell environment or a gitignored `.env` file sitting alongside
  `docker-compose.yml` (Compose's standard `${VAR}` substitution) — the file committed to version control
  contains no secret values, only variable references.
- The `deploy.resources.reservations.devices` block is Compose's way of expressing GPU passthrough — the
  Compose-level equivalent of `docker run --gpus all`, still backed by the same NVIDIA Container Toolkit
  underneath.

---

## Real-World Scenario

A team is deploying a self-hosted Llama-3.1-8B-Instruct model behind their internal agent stack (the same
migration scenario Chapter 4 introduced). They provision two GPU hosts from their cloud provider — ostensibly
identical instance types — and hit two separate container failures, one per host, on the same day.

**Host A** fails immediately on container start with a CUDA initialization error. Running `nvidia-smi` directly
on the host (Section 5.2) shows a driver version whose "max supported CUDA version" line is older than the CUDA
toolkit baked into the `vllm/vllm-openai:v0.26.0` tag they pulled — the instance was provisioned from an older
base image with a stale NVIDIA driver that was never updated. The fix is a host-level driver upgrade (or, as a
stopgap, pulling an older vLLM image tag built against a CUDA version the existing driver does support), not
anything inside the container or the `vllm serve` command.

**Host B** starts cleanly but the on-call engineer notices every redeploy during their rollout testing takes
several extra minutes and re-saturates the host's network egress — they'd forgotten to mount a cache volume
(Section 3) on this particular host's `docker run` command, copy-pasted from an older internal doc before the
team standardized on always including `-v vllm-hf-cache:/root/.cache/huggingface`. Every redeploy re-downloads
the full 8B-parameter checkpoint from the Hugging Face Hub from scratch, because the container's writable layer
— including the cache directory nothing was mounted over — is discarded along with the old container.

Separately, a third teammate flags a support ticket: "the container OOMs under load, I bumped `docker run
--memory` to 64g and it didn't help." The actual failure is a CUDA out-of-memory error from
`--gpu-memory-utilization` being set too high relative to the GPU's actual VRAM under concurrent load (Chapter
10's diagnosis territory) — the host has plenty of system RAM headroom the whole time; the `--memory` flag was
never the relevant limit (Section 7), and raising it did nothing because host RAM was never the bottleneck.

All three are configuration-and-understanding fixes, not vLLM bugs: check `nvidia-smi` before assuming a broken
image, always mount the cache volume, and never reach for Docker's `--memory` flag when the actual symptom is
GPU VRAM exhaustion.

---

## Best Practices

- **Never bake secrets into an image.** `HF_TOKEN`, `VLLM_API_KEY`, and anything similar belong at `docker run`/
  Compose run time (`-e`, `${VAR}` substitution) or in a production secrets manager — never as a Dockerfile `ENV`
  instruction in a committed file (Section 4.1).
- **Always mount a model cache volume.** Treat `-v <volume>:/root/.cache/huggingface` (or the equivalent named
  volume) as a default, not an optimization you add later — the cost of forgetting it (re-downloading multi-GB
  weights on every restart) is easy to miss and easy to avoid (Section 3).
- **Pin image tags in production; don't run `latest`.** A pinned tag (`vllm/vllm-openai:v0.26.0`) gives you a
  reproducible, auditable deployment and lets you control exactly when a vLLM version bump (and any accompanying
  flag/behavior change, per this course's versioning conventions) reaches production, rather than inheriting
  whatever the `latest` tag happens to point to on a given day.
- **Check `nvidia-smi` on the host before debugging a container GPU failure as if it were a vLLM problem**
  (Section 5.2) — driver/CUDA mismatches produce failures that look like application bugs but aren't.
- **Give health checks a realistic `start-period`.** Model loading and CUDA graph capture take real time
  (Section 6.1); a too-short grace period reports "unhealthy" during ordinary startup, not during an actual
  failure.
- **Size `--gpu-memory-utilization`/`--max-model-len` for GPU VRAM and `docker run --memory` (if used at all) for
  host RAM — never use one to try to control the other** (Section 7).
- **Confirm the container's cache path (`~/.cache/huggingface`) resolves to the user the process actually runs
  as** inside your specific image tag, rather than assuming `/root/...` is permanent across every future release
  (Section 3.2).

---

## Common Mistakes

- **Forgetting `--gpus all`** (or the equivalent Compose GPU reservation block) — the container starts, vLLM
  initializes, and either fails outright with "no GPU found" or silently falls back to a CPU code path far slower
  than intended, depending on version and configuration (Section 2.3).
- **Not mounting a cache volume**, then being confused why every restart or redeploy re-downloads the full model
  and takes minutes longer than expected (Section 3) — this is the single easiest-to-miss mistake in this
  chapter because the container runs *correctly* without the volume, just wastefully.
- **Baking `HF_TOKEN` (or any credential) into a Dockerfile `ENV` instruction** rather than injecting it at run
  time — the token becomes extractable from the image's layer history regardless of what the running container's
  environment shows (Section 4.1).
- **Assuming a container GPU failure is a vLLM bug** without first checking `nvidia-smi` on the host for a
  driver/CUDA mismatch (Section 5) — this is the "works on my machine" failure mode of GPU containers
  specifically.
- **Conflating `docker run --memory` with GPU VRAM limits** — raising a host-RAM limit does nothing for a CUDA
  out-of-memory error, and the actual VRAM-governing flag is `--gpu-memory-utilization` (Chapter 10), not
  anything Docker-level (Section 7).
- **Running `latest` in production** instead of a pinned tag, inheriting an unplanned version bump (and any
  accompanying flag/behavior change) on whatever day the tag happens to move.
- **Setting a health check `start-period` too short for the model size being loaded**, causing false-positive
  "unhealthy" reports during ordinary large-model startup (Section 6.1).

---

## Summary

- `vllm/vllm-openai` bundles the engine, the OpenAI-compatible server, and pre-compiled CUDA kernels — build a
  custom image only for real edge cases, and prefer `FROM vllm/vllm-openai:<tag>` over starting from scratch even
  then (Section 1).
- The NVIDIA Container Toolkit is what lets `--gpus all` actually expose the host's GPU and driver to a
  container; it's a standard CUDA-container prerequisite, not vLLM-specific, and worth sanity-checking with a
  bare `nvidia/cuda` container before debugging vLLM itself (Section 2).
- Mount a host directory or named volume onto the container's Hugging Face cache path so model weights persist
  across restarts — an easy detail to miss because its absence doesn't error, it just wastes time and bandwidth
  (Section 3).
- Pass `HF_TOKEN`/`HUGGING_FACE_HUB_TOKEN` as a run-time environment variable for gated/private repos; never bake
  a credential into a committed Dockerfile (Section 4).
- CUDA-toolkit-in-image and NVIDIA-driver-on-host must be compatible; `nvidia-smi` on the host reports the
  driver's maximum supported CUDA version — check this before assuming a container failure is an application bug
  (Section 5).
- Wire the unauthenticated `/health` endpoint into a Docker `HEALTHCHECK` (or `--health-cmd`) with a startup
  grace period long enough for weight loading — the same endpoint Chapter 20 wires into Kubernetes readiness/
  liveness probes (Section 6).
- Container-level memory limits (`--memory`, host RAM, CPU-side overhead) and GPU VRAM (`--gpu-memory-utilization`,
  Chapter 10) are two separate resources governed by two separate mechanisms — don't reach for one to fix a
  symptom caused by the other (Section 7).
- A complete production `docker run` combines GPU passthrough, a cache volume, run-time credentials, port
  mapping, a health check, and `--api-key` — Section 9 has the full worked command and a Compose equivalent.

---

## Knowledge Check

1. What three things does the `vllm/vllm-openai` image bundle, and under what circumstances would building your
   own vLLM+CUDA image from scratch actually be justified?
2. What specifically does the NVIDIA Container Toolkit do, and why does a plain `docker run` without `--gpus all`
   fail to see any GPU even on a host that clearly has one installed?
3. You redeploy a vLLM container and notice it re-downloads the full model every time, adding several minutes to
   every restart. What's the most likely missing piece in the `docker run` command, and what's the fix?
4. Why is baking `HF_TOKEN` into a Dockerfile's `ENV` instruction a security problem even if the running
   container's environment is never inspected by an attacker?
5. A container fails immediately with a CUDA initialization error on one GPU host but works fine on another,
   apparently identical, host. What's the first thing to check, and what command reveals it?
6. A teammate raises `docker run --memory` to fix a CUDA out-of-memory error under load. Explain why this won't
   work and what the actual fix is.

---

## Hands-On Exercise

1. **Confirm the NVIDIA Container Toolkit is working** by running `docker run --rm --gpus all
   nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi` and comparing the output to running `nvidia-smi` directly on
   the host.
2. **Launch `vllm/vllm-openai`** with a small instruct model (adjust for your available VRAM), including
   `--gpus all`, a mounted cache volume, port mapping, and `--api-key` — following Section 9's pattern (omit
   `HF_TOKEN` if your chosen model isn't gated).
3. **Verify `/health`** responds without any auth header: `curl http://localhost:8000/health`.
4. **Send a chat completion request** to `/v1/chat/completions` with your `--api-key`'s bearer token, confirming
   the container-based server behaves identically to the bare-metal server from Chapter 4.
5. **Stop and remove the container** (`docker rm -f`), then launch a fresh container with the *same* cache volume
   mounted, and confirm from the logs that it skips re-downloading the model and loads directly from the mounted
   cache.
6. **Deliberately omit the cache volume** on one more run and time how much longer startup takes versus step 5,
   to make Section 3's cost concrete rather than theoretical.
7. **Optional:** add a `HEALTHCHECK`/`--health-cmd` to your run and use `docker inspect --format='{{json
   .State.Health}}' <container>` to watch its status transition from `starting` to `healthy`.

---

## Further Reading

- [vLLM Docs — Deploying with Docker](https://docs.vllm.ai/en/latest/deployment/docker.html) — the authoritative
  reference for the `vllm/vllm-openai` image and its supported flags/tags
- [vLLM Docs — Deployment overview](https://docs.vllm.ai/en/latest/deployment/) — how Docker fits alongside the
  Kubernetes/Helm path covered in Chapter 20
- [NVIDIA Container Toolkit — Installation Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
  — host-level setup referenced in Section 2.2
- [NVIDIA Container Toolkit — User Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/index.html)
  — `--gpus` selection syntax and troubleshooting
- [Hugging Face Hub — Authentication](https://huggingface.co/docs/huggingface_hub/en/quick-start#authentication)
  — `HF_TOKEN` / `HUGGING_FACE_HUB_TOKEN` behavior referenced in Section 4
- `github.com/vllm-project/vllm/releases` and `hub.docker.com/r/vllm/vllm-openai/tags` — check before pinning any
  specific image tag for production
- Related chapter in this course: [Chapter 4 — The OpenAI-Compatible Server](./04-openai-compatible-server.md) —
  `vllm serve` flags forwarded through the container's entrypoint, and the confirmed endpoint table
- Related chapter in this course: [Chapter 10 — Memory Management](./10-memory-management.md) — the
  `--gpu-memory-utilization`/`--max-num-seqs` flags this chapter's Section 7 distinguishes from Docker's own
  memory limits
- Related chapter in this course: [Chapter 20 — Production Serving](./20-production-serving.md) — wiring
  `/health`/`/metrics` into Kubernetes probes and Prometheus, and the `vllm-project/production-stack` Helm path

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./18-performance-tuning.md">← Previous: Performance Tuning</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./20-production-serving.md">Next: Production Serving →</a>
</div>
