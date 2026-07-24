# Chapter 19: Production Deployment

> "A graph that only runs in your notebook isn't a product yet — it's a demo with good intentions." — every engineer who has shipped an agent to real users

## Learning Objectives

By the end of this chapter, you will be able to:

- Lay out a LangGraph project the way the framework expects it: graph-definition modules separated from serving code, a `langgraph.json` manifest, and a dependency file
- Write a complete, annotated `langgraph.json`, explaining exactly what the `graphs`, `dependencies`, and `env` fields do and how the CLI/Platform consume them
- Compare the two production deployment shapes — self-hosted (embedding a compiled graph inside your own FastAPI app) versus the LangGraph Platform/CLI (`langgraph dev` / `langgraph up`) — and articulate the concrete tradeoffs of each
- Build a FastAPI service that wraps a compiled graph with a `PostgresSaver` checkpointer and exposes a Server-Sent Events streaming chat endpoint keyed by an authenticated user's `thread_id`
- Containerize a LangGraph + FastAPI service with Docker, correctly handling the Postgres connection string and LLM API keys as environment variables rather than baked-in secrets
- Separate dev/staging/prod configuration — checkpointer backend, model provider, feature flags — using a single codebase with environment-driven behavior
- Explain why a Postgres-backed checkpointer makes your app layer stateless across instances, and why `MemorySaver` cannot survive a multi-instance or multi-restart deployment

---

## Prerequisites for the Chapter

This chapter is the synthesis point of the entire "Phase 4: Production" arc, and it assumes you arrive with the following already solid:

- **Checkpointing & Durable Execution (Chapter 9)**: you should already know what a checkpointer is, the `BaseCheckpointSaver` interface, and specifically how `PostgresSaver`/`AsyncPostgresSaver` persist state keyed by `thread_id` and `checkpoint_id`. This chapter does not re-derive checkpointing from scratch — it puts the Postgres checkpointer into a real serving layer.
- **Streaming (Chapter 11)**: familiarity with `graph.astream()` and the different `stream_mode` values (`"values"`, `"updates"`, `"messages"`). This chapter reuses `stream_mode="messages"` heavily to build a token-streaming chat endpoint.
- **Compilation & Execution (Chapter 7)**: you should be comfortable with `.compile()`, `.ainvoke()`, and passing a `config` dict with `configurable.thread_id` to scope execution to a conversation.
- **FastAPI fluency**: routes, dependency injection, `async def` handlers, `Depends()`, lifespan events (`@asynccontextmanager` app lifespans), and `StreamingResponse`. This chapter assumes you can read and write idiomatic FastAPI without hand-holding — the value-add here is *what's different* when a compiled LangGraph graph sits behind your routes, not a FastAPI tutorial.
- **Docker basics**: writing a `Dockerfile`, understanding layers/caching, and passing environment variables to a container (`-e`, `--env-file`, or a Compose file). No Kubernetes knowledge is assumed.
- **Error Handling & Resilience (Chapter 18)**: the retry/fallback/timeout patterns from that chapter apply directly inside the nodes of the graph you're about to deploy — this chapter doesn't repeat them, but a production deployment without them is incomplete.

If any of the checkpointing material feels shaky, revisit **Chapter 9** before continuing — everything in this chapter's worked example assumes `PostgresSaver` is already a known quantity, not a new concept.

---

## 1. Why Deployment Is a Different Problem Than "Does the Graph Run"

Everything through Chapter 18 answered "does this graph produce the right output, correctly, resiliently?" This chapter answers a different question: "how does a *process* — one that might be killed and restarted, scaled to five replicas behind a load balancer, and hit by a thousand different users with a thousand different conversations at once — serve that graph correctly?"

Three properties define whether a LangGraph application is production-ready, and none of them are about the graph's internal logic:

1. **Statelessness at the app layer.** Any of your `N` running instances must be able to handle a request for *any* `thread_id`, because a load balancer doesn't know or care which instance last touched a given user's conversation. This is only possible if state doesn't live in that instance's memory — it must live in a shared, external store all instances can reach. That store is the checkpointer's backing database.
2. **A stable public contract.** Client applications (a chat UI, a mobile app, another backend service) need HTTP endpoints with predictable request/response and streaming shapes — they should never need to know that "invoke" internally means "call `graph.ainvoke()` with a `thread_id` derived from a JWT."
3. **Reproducible builds and config.** The same code, run in dev, staging, and production, should behave identically except for externally injected configuration — which model provider to call, which checkpointer backend to use, which feature flags are on. Hardcoding any of that inside graph or node code is the single most common way deployments become fragile.

The rest of this chapter builds toward those three properties, first by fixing the project layout and configuration format LangGraph expects, then by building the actual serving layer two different ways.

---

## 2. Application Structure: Laying Out a Deployable Project

A LangGraph project that's going to be deployed — whether self-hosted or via the Platform — benefits from separating three concerns that are easy to tangle together during prototyping: **graph definitions**, **serving/API code**, and **configuration**. Here's the layout this chapter builds toward:

```
my-langgraph-app/
├── langgraph.json              # deployment manifest (Section 3)
├── pyproject.toml              # or requirements.txt — dependency pinning
├── .env                        # local secrets, NEVER committed (see .gitignore)
├── .env.example                # committed template showing required keys
├── Dockerfile                  # Section 6
├── docker-compose.yml          # app + Postgres for local/staging parity
│
├── src/
│   └── my_app/
│       ├── __init__.py
│       ├── graph.py            # builds and compiles the graph — no serving code here
│       ├── state.py            # State schema (TypedDict/Pydantic model)
│       ├── nodes/
│       │   ├── __init__.py
│       │   ├── router.py
│       │   ├── retrieve.py
│       │   └── generate.py
│       ├── config.py           # Settings object — reads env vars once, typed
│       └── checkpointer.py     # builds the checkpointer based on config
│
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI app: routes, lifespan, /invoke, /stream
│   ├── deps.py                 # auth dependency, thread_id derivation
│   └── schemas.py              # Pydantic request/response models for the API
│
└── tests/
    ├── test_graph.py
    └── test_api.py
```

The load-bearing design decision here is that **`src/my_app/graph.py` has no knowledge that FastAPI exists.** It exposes one thing: a function (or module-level variable) that returns a compiled graph. This is exactly the same discipline you already apply in LangChain Core work — keeping a chain's definition free of framework-specific serving concerns so it can be tested standalone, imported by a script, imported by a notebook, *or* imported by an API layer, without modification. LangGraph's own tooling (the CLI and Platform, Section 4.2) formalizes this separation: it literally imports `graph.py` and looks for a compiled graph object or a builder function — it never imports your FastAPI app.

```python
# src/my_app/graph.py
from langgraph.graph import StateGraph, START, END
from my_app.state import AppState
from my_app.nodes.router import route_node
from my_app.nodes.retrieve import retrieve_node
from my_app.nodes.generate import generate_node


def build_graph():
    """Builder function — returns an UNCOMPILED graph.

    Exposing the builder (rather than a pre-compiled graph) lets callers
    inject a checkpointer/store at compile time, which matters because
    the checkpointer differs between local dev, tests, and production.
    """
    builder = StateGraph(AppState)
    builder.add_node("router", route_node)
    builder.add_node("retrieve", retrieve_node)
    builder.add_node("generate", generate_node)

    builder.add_edge(START, "router")
    builder.add_conditional_edges(
        "router",
        lambda state: state["route"],
        {"needs_retrieval": "retrieve", "direct": "generate"},
    )
    builder.add_edge("retrieve", "generate")
    builder.add_edge("generate", END)
    return builder


# A ready-to-use, uncompiled graph for tooling (e.g., `langgraph dev`) that
# doesn't need a custom checkpointer wired in — see Section 3 for how
# langgraph.json references this.
graph = build_graph()
```

Two exposure styles are both legitimate and you'll see both in the wild:

- **Compiled graph, module-level**: `graph = build_graph().compile()`. Simplest, works well when the checkpointer is supplied later via `.compile(checkpointer=...)` overrides, or when you're fine with the default (no checkpointer, or a dev-only `MemorySaver`).
- **Builder function**: `build_graph()` returning the uncompiled `StateGraph`, with compilation (and checkpointer injection) happening in the code that consumes it — your FastAPI lifespan, or LangGraph Platform's own bootstrap. This is the more flexible option for production because the *serving layer* decides which checkpointer to attach, rather than baking that decision into the graph module.

The worked example in Section 5 uses the builder-function style for exactly that reason: the FastAPI app decides, based on environment configuration, whether to compile with a `PostgresSaver` (prod) or a `MemorySaver` (local dev/tests).

---

## 3. `langgraph.json` in Detail

`langgraph.json` is the manifest that tells LangGraph's tooling — the CLI (`langgraph dev`, `langgraph up`), and the Platform's build pipeline — three things: **which graphs exist and where to import them from**, **what to install to run them**, and **what environment variables/files they need**. It lives at the project root, alongside `pyproject.toml`/`requirements.txt`.

Here is a fully annotated example:

```jsonc
{
  // Schema version for the manifest format itself. Bump only when you
  // intentionally adopt a newer manifest schema; otherwise leave as-is.
  "dependencies": [
    ".",                          // install the local package (this project)
    "langchain-openai",           // additional third-party deps not already
    "langchain-anthropic"         // captured by the local package's own metadata
  ],

  // The graphs mapping: a logical NAME the CLI/Platform will expose,
  // mapped to an IMPORT PATH of the form "<module path>:<attribute>".
  // The attribute can be a compiled graph OR a zero-arg builder function
  // that returns one (LangGraph compiles it if it isn't already compiled).
  "graphs": {
    "support_agent": "./src/my_app/graph.py:graph",
    "support_agent_builder": "./src/my_app/graph.py:build_graph"
  },

  // Path to a .env file the CLI loads before importing your graph module,
  // so os.environ has API keys, DB URLs, etc. available at import time.
  // Never commit the referenced file itself — only .env.example.
  "env": ".env",

  // Optional: pin the Python version the build should target, to keep
  // local dev, CI, and any managed build environment consistent.
  "python_version": "3.11",

  // Optional: extra files/directories to include verbatim in the build
  // context beyond what dependency resolution would pull in automatically
  // (useful for prompt templates, few-shot example files, static configs).
  "include": [
    "./src/my_app/prompts"
  ]
}
```

Field-by-field detail:

- **`graphs`** — this is the mapping the reader should internalize most: a human-readable **name** (this becomes the identifier clients use when talking to a Platform deployment, and the name shown in `langgraph dev`'s local UI) mapped to an **import path**. The import path syntax `module:attribute` is the same convention Gunicorn/Uvicorn use for `app:app` — it's not new syntax, it's a pattern you already know from serving FastAPI itself. You can register multiple graphs in one manifest; a single deployable service can expose several distinct graphs (e.g., a customer-facing chat agent and an internal admin/triage agent) under different names.
- **`dependencies`** — a list of installable requirements. `"."` means "install this project itself" (so `pyproject.toml`'s own dependencies get resolved), and you can add extra entries for packages that aren't part of your package's declared dependencies but are needed at graph-import time. In most real projects, prefer declaring everything in `pyproject.toml`/`requirements.txt` and keeping this list to just `["."]` — duplicating dependency declarations in two places invites drift.
- **`env`** — a path to a dotenv file loaded into the process environment before your graph module is imported. This matters because module-level code (like instantiating a chat model client) often reads `os.environ` at import time — if the env file loads *after* import, you'll get confusing `None`/missing-key errors. In the self-hosted FastAPI shape (Section 4.1), you don't need this field at all — FastAPI/your process manager loads env vars its own way (e.g., `python-dotenv`, or the container runtime injecting them) — `langgraph.json`'s `env` field is specifically consumed by the LangGraph CLI/Platform tooling.
- **`python_version` / `include`** — optional knobs for controlling the build environment when using CLI/Platform-based builds (Section 4.2); irrelevant if you're only ever building your own Docker image by hand (Section 6), since you fully control the build steps there.

A subtlety worth calling out explicitly: **`langgraph.json` is only read by LangGraph's own tooling** (`langgraph dev`, `langgraph up`, Platform builds). If you go the pure self-hosted-FastAPI route and never invoke the CLI, this file is optional — you could omit it entirely and everything in Section 5 would still work, since your FastAPI app imports `build_graph` directly in Python, not through this manifest. It becomes mandatory the moment you want `langgraph dev`'s local debug server or a Platform deployment, which is precisely why Section 4 treats it as one of two alternate paths rather than a universal requirement.

---

## 4. Two Deployment Shapes

There are two legitimate ways to get a compiled LangGraph graph in front of real traffic, and picking between them is an architecture decision, not a style preference.

### 4.1 Shape 1 — Self-Hosted: Embed the Graph Inside Your Own FastAPI App

You already run FastAPI services. In this shape, a compiled LangGraph graph is just another dependency your app imports and calls — conceptually identical to how you'd embed an LCEL chain today. You own:

- The HTTP surface (`/invoke`, `/stream`, auth, rate limiting, request validation) — all standard FastAPI.
- The checkpointer lifecycle (opening/closing the Postgres pool alongside the app's lifespan).
- Deployment mechanics (your existing Docker/Kubernetes/ECS pipeline, unchanged).

This is the shape Section 5's worked example builds, and it's the natural default for teams who already have FastAPI infrastructure, CI/CD, observability, and auth conventions they don't want to abandon just because one internal component is now a LangGraph graph instead of an LCEL chain.

### 4.2 Shape 2 — LangGraph Platform / CLI-Based Deployment

LangGraph ships a CLI with two commands that stand up a serving layer *for you*, generated directly from `langgraph.json`:

- **`langgraph dev`** — a local development server with hot-reload, a built-in visual debugger (LangGraph Studio) for stepping through graph execution, and a REST API automatically exposing every graph declared in `graphs`. It's the fastest way to interactively poke at a graph while building it — no serving code to write at all.
- **`langgraph up`** — builds and runs the same service in a production-oriented mode (typically via a generated Docker image), giving you HTTP endpoints for invoking/streaming your graphs, run and thread management, and built-in persistence, without you writing a single FastAPI route. This is the local/self-managed on-ramp to the same serving layer that LangGraph Platform runs at managed scale in the cloud.

What you get "for free" that you'd otherwise hand-roll: thread/run management endpoints, background/async run execution with polling, built-in webhook support on run completion, and out-of-the-box integration with LangSmith tracing (Chapter 20). What you give up: control over the exact HTTP contract (you adapt to LangGraph's API shape rather than defining your own `/invoke`/`/stream`), and — if you go all the way to the managed Platform — where the service physically runs.

**Comparison table:**

| Concern | Self-hosted FastAPI (4.1) | LangGraph CLI/Platform (4.2) |
|---|---|---|
| Serving code you write | All of it (routes, streaming, auth) | None — generated from `langgraph.json` |
| HTTP contract | Fully custom, matches your existing API conventions | LangGraph's own API shape (runs, threads, streaming) |
| Fits into existing infra (auth, gateway, CI/CD) | Naturally — it's just another FastAPI service | Requires adapting existing infra to a new service shape |
| Local iteration speed | Normal FastAPI dev loop (`uvicorn --reload`) | `langgraph dev` + Studio visual debugger — faster for graph-only iteration |
| Thread/run management, webhooks | Build yourself | Included |
| Mixing graph logic with non-graph business logic (billing, other DB models, unrelated endpoints) in one deployable | Trivial — it's one app | Awkward — the deployable is scoped to graphs |
| Where it runs | Wherever you already deploy FastAPI | Self-managed via `langgraph up`, or LangGraph's managed Platform |
| Best fit | Teams with existing FastAPI/API infra and custom contract needs | Teams that want the serving layer solved and are comfortable adopting LangGraph's API shape |

**The practical decision rule:** if the graph is one endpoint inside a larger existing service (e.g., an internal support tool that already has user management, billing, and a dozen unrelated endpoints), embed it (4.1) — you don't want two different serving paradigms in one product. If the graph *is* the product — a standalone conversational agent service with no other responsibilities — the CLI/Platform path (4.2) removes a meaningful amount of boilerplate and gives you thread/run management and tracing integration without writing it. Many teams prototype with `langgraph dev` (4.2) for the tight debug loop, then move the finished graph into a self-hosted FastAPI wrapper (4.1) for the actual production deployment once it needs to sit next to existing services — that migration is cheap precisely because Section 2's layout keeps `graph.py` serving-agnostic.

---

## 5. Worked Example: FastAPI + PostgresSaver + SSE Streaming Chat Endpoint

This is the centerpiece of the chapter: a self-hosted (Shape 4.1) FastAPI service wrapping a compiled graph, backed by `PostgresSaver`, exposing a streaming chat endpoint over Server-Sent Events, with `thread_id` derived from an authenticated session rather than trusted from client input.

### 5.1 Configuration (`src/my_app/config.py`)

Centralize every environment-dependent value in one typed object, read once at process start. This is the single most important discipline for Section 7's dev/staging/prod story.

```python
# src/my_app/config.py
from functools import lru_cache
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    environment: str = "dev"                 # "dev" | "staging" | "prod"

    # Checkpointer backend selection — see Section 7.
    checkpointer_backend: str = "memory"      # "memory" | "postgres"
    database_url: str | None = None           # postgresql://user:pass@host:5432/db

    # Model provider config — never hardcode keys in code.
    openai_api_key: str | None = None
    anthropic_api_key: str | None = None
    default_model: str = "claude-sonnet-4-5"

    # Feature flags — plain booleans, toggled per environment.
    enable_retrieval_node: bool = True
    enable_verbose_tracing: bool = False

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"


@lru_cache
def get_settings() -> Settings:
    """Cached so we parse env vars exactly once per process."""
    return Settings()
```

### 5.2 Checkpointer factory (`src/my_app/checkpointer.py`)

```python
# src/my_app/checkpointer.py
from contextlib import asynccontextmanager
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from my_app.config import Settings


@asynccontextmanager
async def build_checkpointer(settings: Settings):
    """Yields a checkpointer appropriate to the environment.

    Using an async context manager here (rather than returning the
    object directly) matters because AsyncPostgresSaver owns a
    connection pool that must be opened and closed cleanly alongside
    the FastAPI app's own lifespan — see main.py.
    """
    if settings.checkpointer_backend == "postgres":
        if not settings.database_url:
            raise RuntimeError(
                "checkpointer_backend=postgres requires DATABASE_URL to be set"
            )
        async with AsyncPostgresSaver.from_conn_string(
            settings.database_url
        ) as checkpointer:
            # Idempotent — safe to call on every startup; only does real
            # work the first time or after a schema change.
            await checkpointer.setup()
            yield checkpointer
    else:
        # Dev/test fallback: in-process, lost on restart, NOT shared
        # across instances. See Section 8 for why this cannot be used
        # in any multi-instance deployment.
        yield MemorySaver()
```

### 5.3 Auth and thread_id derivation (`app/deps.py`)

The thread_id must never be a value the client hands you directly and unchecked — otherwise any authenticated user could read or continue any other user's conversation just by guessing or supplying their `thread_id`. Derive it from the authenticated identity instead.

```python
# app/deps.py
from fastapi import Depends, Header, HTTPException
from my_app.config import Settings, get_settings


class AuthenticatedUser:
    def __init__(self, user_id: str, session_id: str):
        self.user_id = user_id
        self.session_id = session_id

    @property
    def thread_id(self) -> str:
        # One thread per (user, session) — e.g., one thread per browser tab
        # or per distinct conversation the user starts. Swap this scheme
        # for "one thread per user_id" if you want a single continuous
        # conversation per user instead of one per session.
        return f"user:{self.user_id}:session:{self.session_id}"


async def get_current_user(
    authorization: str = Header(...),
    x_session_id: str = Header(...),
    settings: Settings = Depends(get_settings),
) -> AuthenticatedUser:
    # Stand-in for real auth: validate a bearer token against your
    # identity provider / JWT verifier and extract a stable user_id.
    # Never trust a client-supplied user_id header in production.
    token = authorization.removeprefix("Bearer ").strip()
    user_id = verify_jwt_and_get_subject(token)  # raises on invalid token
    if not user_id:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    return AuthenticatedUser(user_id=user_id, session_id=x_session_id)


def verify_jwt_and_get_subject(token: str) -> str | None:
    # Replace with real verification (e.g., python-jose against your
    # identity provider's JWKS endpoint). Kept as a stub here since JWT
    # verification specifics are outside this chapter's scope.
    raise NotImplementedError
```

### 5.4 Request/response schemas (`app/schemas.py`)

```python
# app/schemas.py
from pydantic import BaseModel


class ChatRequest(BaseModel):
    message: str


class ChatResponse(BaseModel):
    thread_id: str
    reply: str
```

### 5.5 The FastAPI app (`app/main.py`)

```python
# app/main.py
import json
from contextlib import asynccontextmanager

from fastapi import Depends, FastAPI
from fastapi.responses import StreamingResponse
from langchain_core.messages import HumanMessage

from my_app.config import get_settings
from my_app.checkpointer import build_checkpointer
from my_app.graph import build_graph
from app.deps import AuthenticatedUser, get_current_user
from app.schemas import ChatRequest, ChatResponse

settings = get_settings()


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Open the checkpointer (and its Postgres connection pool, in prod)
    # exactly once per process, and compile the graph exactly once —
    # NOT per request. Recompiling per request would be the equivalent
    # of rebuilding an LCEL chain on every call: wasteful and, worse,
    # it would silently open a new DB pool per request under load.
    async with build_checkpointer(settings) as checkpointer:
        compiled_graph = build_graph().compile(checkpointer=checkpointer)
        app.state.graph = compiled_graph
        yield
    # Pool is closed automatically on exiting the `async with` block.


app = FastAPI(lifespan=lifespan)


@app.post("/invoke", response_model=ChatResponse)
async def invoke(
    request: ChatRequest,
    user: AuthenticatedUser = Depends(get_current_user),
):
    """Non-streaming call — waits for the full graph run to finish."""
    config = {"configurable": {"thread_id": user.thread_id}}
    result = await app.state.graph.ainvoke(
        {"messages": [HumanMessage(content=request.message)]},
        config=config,
    )
    reply = result["messages"][-1].content
    return ChatResponse(thread_id=user.thread_id, reply=reply)


@app.post("/stream")
async def stream(
    request: ChatRequest,
    user: AuthenticatedUser = Depends(get_current_user),
):
    """Streaming call — Server-Sent Events, one event per token/chunk."""
    config = {"configurable": {"thread_id": user.thread_id}}

    async def event_generator():
        try:
            async for msg_chunk, metadata in app.state.graph.astream(
                {"messages": [HumanMessage(content=request.message)]},
                config=config,
                stream_mode="messages",
            ):
                # msg_chunk is an AIMessageChunk; metadata carries the
                # originating node name, useful for multi-node graphs
                # where the UI wants to show "which step is talking."
                payload = {
                    "node": metadata.get("langgraph_node"),
                    "content": msg_chunk.content,
                }
                yield f"data: {json.dumps(payload)}\n\n"
        except Exception as exc:
            # Never let an unhandled exception silently close the stream —
            # emit a terminal error event the client can act on.
            error_payload = {"error": str(exc)}
            yield f"data: {json.dumps(error_payload)}\n\n"
        finally:
            # A conventional sentinel telling the client the stream is done —
            # matches the pattern most SSE clients (EventSource, fetch-event-source)
            # already expect from LLM streaming APIs.
            yield "data: [DONE]\n\n"

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",  # disable proxy buffering (e.g., nginx)
        },
    )
```

A few details worth pulling out explicitly, because they're the parts most likely to bite a FastAPI engineer new to this integration:

- **The graph is compiled once, in `lifespan`, not per-request.** Compiling on every request would work functionally but would be equivalent to re-establishing a fresh database connection pool per request — a serious performance and connection-exhaustion risk once you're under real traffic.
- **`thread_id` is never accepted from the request body.** It's derived entirely server-side from the authenticated user (Section 5.3). This is the single most important security property of this endpoint: a malicious or buggy client cannot read or continue another user's conversation by supplying a different `thread_id`, because the client never gets to supply one at all.
- **`X-Accel-Buffering: no`** matters if you sit behind nginx or a similar reverse proxy — without it, the proxy may buffer the whole SSE response before forwarding it, defeating the entire purpose of streaming. This is not a LangGraph-specific detail, but it's the single most common reason "my streaming endpoint works locally but comes through all at once in production."
- **`stream_mode="messages"`** (covered in depth in Chapter 11) yields `(chunk, metadata)` tuples specifically because it's built for token-level LLM output; if your graph needs to stream intermediate state updates rather than token deltas, switch to `stream_mode="updates"` and adjust the event payload shape accordingly.

---

## 6. Docker Containerization

A representative `Dockerfile` for this service, using a multi-stage build to keep the final image lean and never bake secrets into a layer:

```dockerfile
# Dockerfile
FROM python:3.11-slim AS builder

WORKDIR /build

# Install build-time dependencies for packages with C extensions
# (e.g., psycopg for Postgres) before copying application code, so this
# layer is cached across code changes that don't touch dependencies.
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml poetry.lock* requirements.txt* ./
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-slim AS runtime

# Runtime-only system dependency: the Postgres client library, without
# the full build toolchain from the builder stage.
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Bring over only the installed Python packages, not the build toolchain.
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

COPY src/ ./src/
COPY app/ ./app/

# Run as a non-root user — standard container hardening practice,
# unrelated to LangGraph specifically but non-negotiable in production.
RUN useradd --create-home appuser
USER appuser

# NOTE: no secrets are set here (no OPENAI_API_KEY, no DATABASE_URL).
# Every environment-specific value is injected at `docker run` /
# orchestrator level — see Section 7. Baking a real key into a layer
# means it's recoverable from the image history forever, even after
# a later layer "removes" it.
ENV ENVIRONMENT=prod \
    CHECKPOINTER_BACKEND=postgres \
    PYTHONUNBUFFERED=1

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

And a `docker-compose.yml` for local/staging parity, wiring the app to a real Postgres instance so the checkpointer path is exercised the same way it will be in production:

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      ENVIRONMENT: staging
      CHECKPOINTER_BACKEND: postgres
      DATABASE_URL: postgresql://langgraph:langgraph@db:5432/langgraph
      OPENAI_API_KEY: ${OPENAI_API_KEY}          # from host shell / CI secret
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}    # from host shell / CI secret
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: langgraph
      POSTGRES_PASSWORD: langgraph
      POSTGRES_DB: langgraph
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U langgraph"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

Two points about secret handling worth stating directly, since they're the most common ways this goes wrong:

- **Never `COPY` a `.env` file with real secrets into the image.** `.env` should be `.gitignore`d *and* Docker-ignored (`.dockerignore`); real values are injected at `docker run -e` / Compose `environment:` / your orchestrator's secret manager (Kubernetes `Secret`, AWS Secrets Manager, etc.), never baked into a layer.
- **`--env-file` at `docker run` time is for local/staging convenience, not a production secrets strategy.** In real production, prefer your platform's native secrets mechanism (mounted as files or injected as env vars by the orchestrator at container start) so secrets never touch a CI log, a `docker inspect` output, or version control.

---

## 7. Environment/Config Management: Dev, Staging, Prod

The `Settings` object from Section 5.1 is the single seam every environment-specific difference passes through. The discipline is: **the code never branches on `if environment == "prod"` — it branches on the *specific setting* that differs**, so adding a fourth environment later doesn't require touching graph or route code.

| Setting | Dev | Staging | Prod |
|---|---|---|---|
| `checkpointer_backend` | `memory` (fast local iteration, no DB needed) | `postgres` (parity with prod) | `postgres` |
| `database_url` | unset | staging Postgres instance | prod Postgres instance (often a managed service, e.g., RDS/Cloud SQL) |
| `default_model` | a cheap/fast model for quick iteration | same as prod, to catch model-specific issues early | production-grade model |
| `enable_verbose_tracing` | `true` | `true` | `false` (or sampled) — full tracing on every request at prod volume gets expensive fast |
| `enable_retrieval_node` | toggled freely while building | mirrors prod | stable, tested value |
| Secrets source | local `.env` file | CI/CD secret store injected as env vars | orchestrator secrets manager |

Concretely, this is why Section 5.1's `Settings` reads `checkpointer_backend` as a plain string rather than the code importing `MemorySaver` directly: swapping dev↔staging↔prod is a **deployment-time environment variable change**, never a code change or a redeploy of different source. The same Docker image, given different environment variables, behaves correctly in all three environments — which is precisely the "reproducible builds and config" property named as a goal in Section 1.

Feature flags deserve the same treatment as backend selection: `enable_retrieval_node` gates whether a node is even wired into the graph (or gates a conditional edge that skips it), read from `Settings` at graph-build time, not hardcoded. This lets you test a new node in staging behind a flag before flipping it on in prod, without maintaining two divergent versions of `graph.py`.

---

## 8. Scaling Considerations

### 8.1 Why `MemorySaver` cannot survive a real deployment

`MemorySaver`, the checkpointer you reach for in Chapter 9's early examples, stores every checkpoint in a plain Python dictionary **inside the process**. That's fine for local development and unit tests, but it has two fatal properties for production:

1. **It doesn't survive a restart.** Every deploy, every crash, every autoscaling scale-down event wipes every in-flight conversation for every user whose thread happened to live on that instance.
2. **It doesn't survive having more than one instance.** The moment you run two replicas behind a load balancer — which any serious production deployment does, for both throughput and availability — a `thread_id` that started on instance A and gets routed to instance B on its next request finds *no history at all*. From the user's perspective, the assistant randomly "forgets" the conversation depending on which instance the load balancer happened to route them to.

### 8.2 Why Postgres fixes both

Swapping to `PostgresSaver`/`AsyncPostgresSaver` moves checkpoint state out of process memory and into a shared, durable database every instance can reach identically. This is the same architectural move you already know from plain web development: moving session state out of in-process memory and into Redis/Postgres/Memcached so any web server behind a load balancer can serve any user's session. LangGraph's checkpointer abstraction is that same pattern, applied to graph execution state instead of HTTP session state.

```mermaid
flowchart TB
    LB["Load Balancer"]
    A1["App Instance 1<br/>(compiled graph, no local state)"]
    A2["App Instance 2<br/>(compiled graph, no local state)"]
    A3["App Instance 3<br/>(compiled graph, no local state)"]
    PG[("Postgres<br/>checkpoints table<br/>(shared, durable)")]

    LB --> A1
    LB --> A2
    LB --> A3
    A1 <--> PG
    A2 <--> PG
    A3 <--> PG

    U1["User A, thread_id=user:A:session:1"] -.request 1.-> LB
    U1 -.request 2 (different instance!).-> LB
```

Because the checkpoint for `thread_id="user:A:session:1"` lives in Postgres and not in any one instance's memory, it does not matter which instance the load balancer routes request 2 to — whichever instance receives it loads the exact same checkpoint from the shared database and continues the conversation correctly. This is precisely the **statelessness at the app layer** named as goal #1 in Section 1: the instances themselves hold no durable state; the database does.

### 8.3 Connection pooling

Once every instance talks to the same Postgres database, connection management becomes a real capacity concern. A few concrete guidelines:

- **Use `AsyncPostgresSaver`'s connection pooling, and size it deliberately.** Each app instance opens its own pool (established once, in `lifespan`, per Section 5.5) — with `N` instances and a pool size of `P` each, you can reach `N × P` concurrent connections to Postgres. Check that figure against your Postgres instance's `max_connections` before scaling instance count blindly.
- **Prefer a pooler in front of Postgres at real scale** (e.g., PgBouncer, or your managed Postgres provider's built-in pooling) once `N × P` starts approaching `max_connections` — this decouples "how many app instances I run" from "how many raw Postgres connections exist," by having the pooler multiplex many logical app-side connections onto fewer real backend connections.
- **Keep the pool lifecycle tied to the app's lifespan, not per-request.** Opening a new connection (or worse, a new pool) per request is the single most common way a LangGraph-behind-FastAPI service silently exhausts its database's connection limit under moderate load — this is exactly why Section 5.5 opens the checkpointer once in `lifespan` and stores the compiled graph on `app.state`.
- **Set conservative pool sizes per instance and scale instance count, not pool size, to handle more traffic** — a smaller number of connections per instance, multiplied by more instances behind the load balancer, is easier to reason about and cap than a few instances each holding huge pools.

---

## Examples

**Minimal `langgraph.json` for a single-graph project** (the smallest viable manifest, useful as a starting template):

```json
{
  "dependencies": ["."],
  "graphs": {
    "chat_agent": "./src/my_app/graph.py:graph"
  },
  "env": ".env"
}
```

**Health check endpoint**, worth adding to any deployed service so your orchestrator (Kubernetes liveness/readiness probes, an ALB target group health check) can verify the app — and, transitively, its database connectivity — is actually healthy, not just that the process is running:

```python
# app/main.py (addition)
@app.get("/healthz")
async def healthz():
    try:
        # A cheap way to confirm the checkpointer's DB connection is alive:
        # list checkpoints for a throwaway thread_id and expect no error.
        async for _ in app.state.graph.checkpointer.alist(
            {"configurable": {"thread_id": "__healthcheck__"}}
        ):
            break
        return {"status": "ok"}
    except Exception as exc:
        return {"status": "degraded", "detail": str(exc)}
```

**Deriving `thread_id` differently depending on product requirements** — this chapter's worked example uses one thread per (user, session), but the right scoping is a product decision, not a technical constraint:

```python
# One continuous conversation per user, across all their sessions/devices:
return f"user:{self.user_id}"

# One thread per explicit "conversation" the user creates (like ChatGPT's
# sidebar of past conversations), where conversation_id comes from the
# client after the server first mints and returns one:
return f"user:{self.user_id}:conv:{self.conversation_id}"
```

---

## Diagrams

```mermaid
flowchart TD
    subgraph Client
        UI["Chat UI / mobile app"]
    end

    subgraph "FastAPI Service (any of N instances)"
        AUTH["Auth dependency<br/>verify JWT → thread_id"]
        INV["/invoke endpoint"]
        STR["/stream endpoint (SSE)"]
        G["Compiled graph<br/>(built once at lifespan startup)"]
    end

    subgraph "Shared Infrastructure"
        PG[("Postgres<br/>PostgresSaver checkpoints")]
        LLM["LLM Provider API<br/>(OpenAI / Anthropic)"]
    end

    UI -->|"Bearer token + message"| AUTH
    AUTH --> INV
    AUTH --> STR
    INV --> G
    STR --> G
    G <-->|"load / save checkpoint by thread_id"| PG
    G -->|"model calls"| LLM
    STR -->|"text/event-stream chunks"| UI
    INV -->|"JSON response"| UI
```

---

## Real-World Scenarios

**Scenario 1 — the silent memory loss during a rolling deploy.** A team ships a customer support agent using `MemorySaver` in production because "it worked fine in staging with one instance." The first rolling deploy — a routine code push, not an incident — recycles every pod one at a time. Every user mid-conversation at the moment their pod was replaced finds the assistant has no memory of anything they said moments earlier. Support tickets spike immediately after every deploy, correlated exactly with deploy timestamps once someone finally checks. The fix is exactly Section 8's move: swap to `AsyncPostgresSaver` backed by a managed Postgres instance, verify via the health check in Examples that the checkpointer is genuinely reachable from every pod, and confirm via a load test that killing one pod mid-conversation no longer loses state (the next request, routed to a surviving pod, loads the same checkpoint from Postgres and continues correctly).

**Scenario 2 — thread_id trusted from the client.** An early prototype accepts `thread_id` as a field in the request body because it was the fastest way to get streaming working during a hackathon. It ships to production unchanged. A security review later finds that any authenticated user can read or continue *any other user's* conversation simply by guessing or brute-forcing `thread_id` values (which, in the prototype, were sequential integers). The fix is Section 5.3's pattern exactly: `thread_id` is never client-supplied — it's derived server-side from the verified identity in the auth token, so there is no value a client can pass that would let it address someone else's thread.

**Scenario 3 — choosing CLI/Platform, then outgrowing it.** A small team building a standalone agent product starts with `langgraph dev` for the tight iteration loop and Studio's visual debugger, then deploys via `langgraph up` to ship an MVP fast with zero hand-written serving code. Six months later, the product has grown to include billing, a admin dashboard, and unrelated non-graph endpoints that don't naturally fit the Platform's graph-scoped API shape. The team migrates the graph into a self-hosted FastAPI app (Shape 4.1) that lives alongside the rest of their existing services — a low-cost migration precisely because their `graph.py` (Section 2) never had any serving-layer code baked into it; only the wrapper around it changed.

---

## Best Practices

- **Keep `graph.py` free of FastAPI, HTTP, or serving-layer imports.** It should be importable and testable in complete isolation — from a script, a test file, `langgraph dev`, or a FastAPI lifespan — with zero modification for any of those callers.
- **Compile the graph and open the checkpointer once, in the app's lifespan — never per-request.** Per-request compilation or connection creation is a hidden performance and connection-exhaustion bug waiting for production load to expose it.
- **Never accept `thread_id` as client-supplied input.** Derive it deterministically from the authenticated identity, so there is no way for a client to address a thread that isn't theirs.
- **Centralize all environment-dependent values in one typed `Settings` object**, read once, and branch behavior on individual settings (`checkpointer_backend`, `default_model`) rather than on a single `environment == "prod"` check scattered through the codebase.
- **Never bake secrets into a Docker image layer.** Inject `DATABASE_URL`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc. at container-run time via your orchestrator's secrets mechanism, and keep `.env` out of both version control and the Docker build context (`.dockerignore`).
- **Add a real health check** that exercises the checkpointer's actual database connectivity, not just "the process is alive" — a process can be running and completely unable to reach Postgres, and a shallow health check will report it as healthy anyway.
- **Match pool size × instance count against your database's connection limit deliberately**, and prefer a pooler (PgBouncer or a managed equivalent) once that product approaches the limit, rather than discovering the ceiling during an incident.
- **Choose the deployment shape (Section 4) based on whether the graph is the whole product or one component of a larger existing service** — don't default to hand-rolling FastAPI serving code out of habit if the CLI/Platform path removes real boilerplate for your case, and don't adopt the Platform's API shape if it fights your existing infrastructure.

---

## Common Mistakes

- **Using `MemorySaver` in any deployment with more than one instance, or any deployment subject to restarts** (i.e., every real production deployment). It silently loses all conversation state on restart or when a request lands on a different instance than the one that handled the previous turn — with no error, just quietly "forgotten" context.
- **Recompiling the graph, or reopening the checkpointer/connection pool, on every request.** This works at low traffic and then causes connection exhaustion or serious latency degradation the moment traffic increases — a classic "worked in the demo" bug.
- **Trusting a client-supplied `thread_id`.** This turns the checkpointer's per-thread isolation — which is supposed to be a security boundary between users' conversations — into no boundary at all.
- **Baking API keys or database URLs into a Docker image** (via a `COPY .env` or a hardcoded `ENV` instruction with a real value), which leaves the secret recoverable from the image's layer history indefinitely, even if a later layer appears to remove it.
- **Buffering SSE responses through a reverse proxy without disabling buffering** (e.g., missing `X-Accel-Buffering: no` behind nginx), producing a streaming endpoint that streams correctly in local testing but delivers everything in one lump in production.
- **Scattering `if environment == "prod"` checks through node and route code** instead of centralizing environment-dependent behavior behind a single `Settings` object — this makes adding a new environment (or auditing what actually differs between environments) far harder than it needs to be.
- **Assuming `langgraph.json`'s `env` field does anything for a self-hosted FastAPI deployment.** It's consumed by LangGraph's own CLI/Platform tooling only; a hand-rolled FastAPI app needs to load its own environment (via `python-dotenv`, the container runtime, or your orchestrator) regardless of what this manifest says.
- **Sizing connection pools per instance without accounting for total instance count**, quietly exceeding the database's `max_connections` the moment autoscaling adds a few more replicas under load.

---

## Summary

- A deployable LangGraph project separates **graph definitions** (framework-agnostic, in `graph.py`) from **serving code** (FastAPI routes, auth, streaming) — the same discipline as keeping LCEL chains free of API-layer concerns.
- **`langgraph.json`** is the manifest LangGraph's own CLI/Platform tooling reads: `graphs` maps a name to a `module:attribute` import path, `dependencies` lists what to install, and `env` points at a dotenv file loaded before your graph module is imported. It is not required for a pure self-hosted FastAPI deployment.
- There are two legitimate deployment shapes: **self-hosted FastAPI**, where you own the full serving layer and it's just another dependency your app imports, and **LangGraph CLI/Platform** (`langgraph dev`/`langgraph up`), which generates a serving layer — including thread/run management — directly from `langgraph.json`. The right choice depends on whether the graph is the whole product or one piece of a larger existing service.
- A production chat endpoint compiles the graph and opens the checkpointer **once, at process startup**, streams via SSE using `stream_mode="messages"`, and derives `thread_id` **from the authenticated user**, never from client-supplied input.
- Docker images should never contain secrets — inject `DATABASE_URL` and LLM API keys at container-run time, and keep environment-specific behavior (checkpointer backend, model choice, feature flags) behind a single typed `Settings` object rather than scattered conditionals.
- **`PostgresSaver`/`AsyncPostgresSaver` makes the app layer stateless**: because checkpoint state lives in a shared database rather than any one instance's memory, any instance can serve any `thread_id`, which is what makes multi-instance, restart-safe deployment possible — something `MemorySaver` can never provide.
- Connection pooling must be sized with total instance count in mind (`N instances × P pool size` against the database's `max_connections`), with a pooler like PgBouncer as the standard mitigation once that product grows large.

---

## Knowledge Check

1. Why does `graph.py` in this chapter's recommended layout have no FastAPI imports at all, and what concrete capability would you lose if you compiled the graph directly inside a route handler instead of in the app's `lifespan`?
2. Walk through what `langgraph.json`'s `graphs` field maps, using the syntax `"name": "./path/to/module.py:attribute"` — what can `attribute` be, and what does the tooling do differently depending on which it is?
3. A teammate proposes accepting `thread_id` as a field in the `/stream` request body "to make testing easier with curl." Explain precisely what security property this breaks and how you'd derive `thread_id` instead.
4. You're comparing self-hosted FastAPI versus `langgraph up` for a new project that is purely a standalone conversational agent with no other product surface. Which shape would you lean toward, and what would change your mind?
5. Explain, in terms of where state physically lives, why two replicas of the same FastAPI app using `MemorySaver` will produce inconsistent conversation behavior for a single user, while two replicas using `AsyncPostgresSaver` will not.
6. Your service runs 6 app instances, each configured with a connection pool of 20. Your managed Postgres instance has `max_connections = 100`. What's wrong with this configuration, and name two ways to fix it.

---

## Hands-on Exercises

1. **Build the layout.** Starting from an existing graph you built in an earlier chapter (or a simple two-node graph you write for this exercise), restructure it into the project layout from Section 2: a `graph.py` with a `build_graph()` builder function, a `Settings` object, and a `langgraph.json` manifest with at least one entry in `graphs`. Verify the graph module has zero imports from `fastapi`.

2. **Wire up the FastAPI wrapper.** Using Section 5 as a template, build `/invoke` and `/stream` endpoints for your graph from Exercise 1. Start with `MemorySaver` to get it working end-to-end, then swap in `AsyncPostgresSaver` against a local Postgres instance (via `docker-compose up db` using this chapter's Compose file). Confirm that a conversation started, then continued in a second request with the same derived `thread_id`, retains context — and that restarting the app process does NOT lose that context once you're on Postgres (unlike with `MemorySaver`, where restarting does lose it).

3. **Simulate multi-instance behavior.** Run two instances of your FastAPI app on two different local ports (e.g., `8000` and `8001`), both pointed at the same Postgres database. Send request 1 to port `8000`, then send request 2 (same derived `thread_id`) to port `8001`. Confirm the second instance continues the conversation correctly despite never having seen request 1 — this is the hands-on proof of Section 8's core claim about app-layer statelessness.

---

## Further Reading

- [LangGraph Application Structure Guide](https://docs.langchain.com/oss/python/langgraph/application-structure) — the canonical reference for project layout and `langgraph.json`
- [LangGraph CLI Reference](https://docs.langchain.com/oss/python/langgraph/local-server) — `langgraph dev`, `langgraph up`, and local server options
- [LangGraph Platform Deployment Documentation](https://docs.langchain.com/langgraph-platform) — managed deployment, thread/run management APIs
- [LangGraph Persistence / Checkpointers Documentation](https://docs.langchain.com/oss/python/langgraph/persistence) — `PostgresSaver`/`AsyncPostgresSaver` API details (builds on Chapter 9)
- [FastAPI Lifespan Events Documentation](https://fastapi.tiangolo.com/advanced/events/) — the `lifespan` pattern used to own the checkpointer's connection pool
- [MDN: Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events) — the wire format underlying the `/stream` endpoint in this chapter
- [PgBouncer Documentation](https://www.pgbouncer.org/) — connection pooling for Postgres at scale, relevant once `instances × pool size` approaches your database's connection limit
- [The Twelve-Factor App: Config](https://12factor.net/config) — the underlying philosophy behind Section 7's environment-variable-driven configuration approach

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./18-error-handling-and-resilience.md">← Previous: Error Handling & Resilience</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./20-observability-and-monitoring.md">Next: Observability & Monitoring →</a>
</div>
