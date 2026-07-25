# Production Deployment

> Seventeen chapters built and tested the agent. This one wires it into the FastAPI/Docker/Kubernetes service that has to stay up.

## Learning Objectives

By the end of this chapter, you will be able to:

- Build an idiomatic FastAPI streaming endpoint around `agent.astream(..., stream_mode="messages")`, piped into a Server-Sent-Events `StreamingResponse`, with `thread_id` derived correctly from your request/session model
- Wire a `PostgresSaver` checkpointer into a FastAPI app's lifespan/dependency-injection pattern, so it is constructed once at startup and shared across every request instead of being rebuilt per call
- Explain — precisely, and without hedging — why `.ainvoke`/`.astream` are the exact same `CompiledStateGraph` you already invoke synchronously, with no separate "async agent" construction path
- Apply your existing retry/rate-limiting discipline (tenacity, provider SDK retry config, `Runnable.with_retry`) to the `model=` you pass into `create_deep_agent()`, and distinguish that from application-level request-quota rate limiting
- Correlate LangSmith traces, structured application logs, and interrupt events around a single `thread_id`, so a production incident is debuggable across both layers at once
- Identify and apply the concrete cost-control levers available to a deployed deep agent: prompt-caching middleware, filesystem eviction thresholds, and summarization tuning
- Reason correctly about per-tenant isolation: request auth is your FastAPI app's job as always, but MCP server auth and `StoreBackend` namespacing are deepagents-specific multi-tenant concerns you must design deliberately
- Containerize a deep-agent FastAPI service and explain, with the underlying mechanism, why `PostgresSaver` (not `SqliteSaver`, and never `MemorySaver`) is required the moment you run more than one replica
- Set request timeouts and readiness/liveness probes deliberately around a service whose graph's `recursion_limit` is 9999, rather than assuming the graph's own limit is a usable safety net

## Prerequisites for This Chapter

- **Chapter 2 (Architecture & Internals)** — `create_deep_agent()` returns an ordinary `CompiledStateGraph` with `recursion_limit` raised to 9999; this chapter is where that number turns into an actual production design constraint (timeouts, worker concurrency) rather than an abstract fact.
- **Chapter 5 (Filesystem-Backed Context)** — `tool_token_limit_before_evict`, the eviction threshold this chapter revisits as a cost-control lever.
- **Chapter 6 (Backends & Storage Architecture)** — `StoreBackend`, `CompositeBackend`, and `NamespaceFactory`; this chapter's multi-tenant section assumes you can already construct a per-tenant namespace.
- **Chapter 7 (Memory & Persistence)** — `add_cache_control=True` on `MemoryMiddleware`-injected content, one of this chapter's concrete cost levers.
- **Chapter 9 (Human-in-the-Loop)** — `interrupt_on` and `Command(resume=...)`; this chapter treats "how does a paused run reach a human reviewer, and how does the resume call get triggered" as application architecture you now have to actually build.
- **Chapter 10 (Checkpointing & Durable Execution)** — the `MemorySaver`/`SqliteSaver`/`PostgresSaver` decision and `thread_id` discipline; this chapter is where that decision gets enforced by an actual multi-replica deployment instead of reasoned about hypothetically.
- **Chapter 11 (MCP Integration)** — `MultiServerMCPClient`; this chapter treats its connection lifecycle and auth as your application's responsibility, same as anywhere else you've used it.
- **Chapter 13's coverage of `AnthropicPromptCachingMiddleware` and a custom `wrap_model_call` middleware for token counting** — both get reused here as production cost-accounting tools, not re-derived from scratch.
- **Your own FastAPI, Docker, and Kubernetes production background.** This chapter does not explain what a `Depends()`, a `StreamingResponse`, a multi-stage Dockerfile, or a readiness probe is. It explains exactly what's different about putting a `create_deep_agent()` graph behind all of that, and nothing else.

---

## This Chapter Is a Synthesis, Not a New Primitive

Every other chapter in this course introduced something new: a middleware, a backend, a state mixin, a subagent shape. This one introduces nothing new. There is no `DeploymentMiddleware`, no `deepagents.serve()`, no framework-provided ASGI app. What you are deploying, mechanically, is precisely the object Chapter 2 already told you `create_deep_agent()` returns: a `CompiledStateGraph`, with a handful of specific middleware installed based on which parameters you passed, running behind whatever HTTP framework you choose. Deploying it to production is exactly the same exercise as deploying any other LangGraph service you've already shipped — the FastAPI route shapes, the Docker image, the Kubernetes manifests are all things you already know how to write.

What this chapter actually does is collect every deployment-relevant decision the previous seventeen chapters surfaced — checkpointer choice (Ch. 10), HITL's need for a durable resume path (Ch. 9), prompt-caching and eviction levers (Ch. 5, 7, 13), MCP connection lifecycle (Ch. 11), the raised `recursion_limit` (Ch. 2) — and shows where each one actually lands in a running FastAPI/Docker/Kubernetes service. If a fact from an earlier chapter doesn't show up here, it's because it was already a complete, self-contained fact; if it does show up here, it's because "knowing the fact" and "having wired it into a real service correctly" turn out to be different skills, and this chapter is about the second one.

```mermaid
flowchart TB
    subgraph Client
        U["Caller (browser, mobile app,<br/>another service)"]
    end

    subgraph K8s["Kubernetes Deployment (N replicas)"]
        subgraph Pod1["Pod: FastAPI + Uvicorn"]
            F1["FastAPI app<br/>(request auth, rate limiting,<br/>SSE streaming endpoint)"]
            A1["create_deep_agent()<br/>CompiledStateGraph<br/>(recursion_limit=9999)"]
            F1 --> A1
        end
        subgraph Pod2["Pod: FastAPI + Uvicorn"]
            F2["FastAPI app"]
            A2["create_deep_agent()<br/>CompiledStateGraph"]
            F2 --> A2
        end
    end

    subgraph Shared["Shared, network-accessible state"]
        PG["PostgresSaver<br/>(checkpoints: messages, todos, files*)"]
        ST["BaseStore<br/>(StoreBackend, per-tenant NamespaceFactory)"]
    end

    subgraph MCP["MCP servers"]
        M1["MCP server 1<br/>(own auth)"]
        M2["MCP server 2<br/>(own auth)"]
    end

    subgraph Obs["Observability"]
        LS["LangSmith tracing"]
        LOG["Structured app logs<br/>(thread_id, tool calls, interrupts)"]
    end

    U -->|"HTTPS, load-balanced"| F1
    U -->|"HTTPS, load-balanced"| F2
    A1 <--> PG
    A2 <--> PG
    A1 <--> ST
    A2 <--> ST
    A1 <--> MCP
    A2 <--> MCP
    A1 -.->|"traces"| LS
    A2 -.->|"traces"| LS
    F1 -.->|"structured logs"| LOG
    F2 -.->|"structured logs"| LOG
```

Any replica can pick up any `thread_id` because `PostgresSaver` and the `BaseStore` are network-accessible and shared, not process-local — that single fact is what makes the "N replicas" box in this diagram actually work, and it's the thread running through most of this chapter.

## FastAPI Integration: A Streaming Endpoint

The learner this course assumes already knows how to wire a `StreamingResponse` around an SSE-shaped generator; the only genuinely new piece is what to put inside that generator. `agent.astream(..., stream_mode="messages")` is the natural fit for a chat-style endpoint — it yields `(message_chunk, metadata)` tuples as tokens/messages are produced, which maps directly onto an SSE `data:` stream.

### Wiring the checkpointer at startup (lifespan + dependency injection)

The checkpointer must be constructed once and shared across every request in the process — not per-request, which would be wasteful, and not module-level at import time, which fights FastAPI's own startup/shutdown hooks. The idiomatic place for it is the app's `lifespan`:

```python
# app/deps.py
from contextlib import asynccontextmanager

from fastapi import FastAPI, Depends, Request
from langgraph.checkpoint.postgres import PostgresSaver

from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StoreBackend, FilesystemBackend
from langgraph.store.postgres import PostgresStore  # your chosen BaseStore implementation

from app.model import build_chat_model  # your existing model construction, Ch. 13's prompt-caching middleware included
from app.tools import support_tools
from app.tenancy import tenant_namespace  # NamespaceFactory, see "Authentication" below

DATABASE_URL = "postgresql://deepagents:***@postgres:5432/deepagents"


@asynccontextmanager
async def lifespan(app: FastAPI):
    with PostgresSaver.from_conn_string(DATABASE_URL) as checkpointer, \
         PostgresStore.from_conn_string(DATABASE_URL) as store:
        # Idempotent — creates the checkpoint tables the first time a deployment
        # runs against a fresh database. In a mature deployment this is usually
        # promoted into an explicit migration step rather than run at every boot.
        checkpointer.setup()
        store.setup()

        backend = CompositeBackend(
            default=FilesystemBackend(root_dir="/data/scratch"),
            routes={"/memories/": StoreBackend(store=store, namespace=tenant_namespace)},
        )

        app.state.agent = create_deep_agent(
            model=build_chat_model(),
            tools=support_tools,
            backend=backend,
            checkpointer=checkpointer,
            store=store,
        )
        yield
    # checkpointer/store connections are closed cleanly on shutdown


def get_agent(request: Request):
    return request.app.state.agent
```

```python
# app/main.py
from fastapi import FastAPI
from app.deps import lifespan

app = FastAPI(lifespan=lifespan)
```

One `create_deep_agent()` call, one `PostgresSaver`, one `BaseStore`, constructed exactly once per process at startup — every request handler injects the same compiled graph via `Depends(get_agent)` rather than rebuilding it. This is the same pattern you already use for a database connection pool or an HTTP client session in any other FastAPI service; nothing about `create_deep_agent()` requires special-casing it.

### The streaming route

```python
# app/routes/chat.py
import json

from fastapi import APIRouter, Depends
from fastapi.responses import StreamingResponse
from pydantic import BaseModel

from app.deps import get_agent
from app.tenancy import get_current_tenant, Tenant

router = APIRouter()


class ChatRequest(BaseModel):
    session_id: str
    message: str


def _sse(event: dict) -> str:
    return f"data: {json.dumps(event)}\n\n"


@router.post("/chat/stream")
async def chat_stream(
    req: ChatRequest,
    tenant: Tenant = Depends(get_current_tenant),
    agent=Depends(get_agent),
):
    # thread_id derived from your own request/session model, namespaced per
    # tenant so two tenants can never collide on the same session_id string.
    thread_id = f"{tenant.id}:{req.session_id}"
    config = {"configurable": {"thread_id": thread_id}}

    async def event_stream():
        async for chunk, metadata in agent.astream(
            {"messages": [{"role": "user", "content": req.message}]},
            config=config,
            context={"tenant_id": tenant.id},
            stream_mode="messages",
        ):
            yield _sse({
                "node": metadata.get("langgraph_node"),
                "content": chunk.content,
            })
        yield "event: done\ndata: {}\n\n"

    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

Nothing here is deepagents-specific beyond the shape of `config`/`context` — `stream_mode="messages"` is a standard LangGraph `stream_mode`, exactly as documented for any `CompiledStateGraph` (the same values — `"values"`, `"updates"`, `"messages"`, `"custom"`, `"debug"` — apply unchanged). If your endpoint needs to surface tool-call boundaries rather than token-level content, `stream_mode="updates"` gives you per-node state deltas instead; which one you pick is a UI decision, not a deepagents one.

### Handling errors without leaking internals to the client

A production streaming endpoint needs an explicit answer to "what happens when the agent call fails partway through a stream" — a provider timeout, a tool exception, a `GraphRecursionError` if you've tightened `recursion_limit` for a cost-bounded path (Ch. 2) and a task genuinely exceeds it. None of this is deepagents-specific; it's the same error-boundary discipline you already apply around any other LLM call, just applied inside the SSE generator instead of a plain request handler:

```python
from langgraph.errors import GraphRecursionError

async def event_stream():
    try:
        async for chunk, metadata in agent.astream(
            {"messages": [{"role": "user", "content": req.message}]},
            config=config,
            stream_mode="messages",
        ):
            yield _sse({"node": metadata.get("langgraph_node"), "content": chunk.content})
    except GraphRecursionError:
        log.error("agent_recursion_limit_hit", thread_id=thread_id)
        yield _sse({"error": "task_too_long", "detail": "exceeded configured step budget"})
    except Exception:
        log.exception("agent_stream_failed", thread_id=thread_id)
        yield _sse({"error": "internal_error"})
    finally:
        yield "event: done\ndata: {}\n\n"
```

The specific thing worth internalizing from Chapter 2 here: at the default `recursion_limit=9999`, `GraphRecursionError` is not a realistic thing to plan around — a task would need to accumulate an implausible number of tool-call turns to hit it. If you've deliberately tightened the limit for a cost-bounded production path (`agent.with_config({"recursion_limit": N})`, per Ch. 2), this is exactly where that tighter ceiling turns into a real, catchable exception your endpoint needs to translate into a sane client-facing error rather than a raw 500.

## Async Agents: Same Graph, No Separate Constructor

Chapter 3 already established this, and it is worth restating here precisely because a FastAPI service is exactly the context where the temptation to look for "the async version" shows up: **there is no separate async `create_deep_agent()`.** `.ainvoke`, `.astream`, `.invoke`, and `.stream` are all methods on the exact same `CompiledStateGraph` object that `create_deep_agent()` returned. You call the async methods because your FastAPI route is `async def`, not because deepagents required a different construction path to get there.

A plain, non-streaming async route, for comparison:

```python
@router.post("/chat")
async def chat(
    req: ChatRequest,
    tenant: Tenant = Depends(get_current_tenant),
    agent=Depends(get_agent),
):
    thread_id = f"{tenant.id}:{req.session_id}"
    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": req.message}]},
        config={"configurable": {"thread_id": thread_id}},
        context={"tenant_id": tenant.id},
    )
    return {"reply": result["messages"][-1].content}
```

Same `app.state.agent`, same `create_deep_agent()` call at startup, same checkpointer — the only difference between this route and the streaming one above is `ainvoke` vs. `astream` and whether the response is buffered or chunked. If you already know how to choose between a buffered JSON response and an SSE stream for any other async FastAPI endpoint, you already know how to choose here.

## Retry Policies and Rate Limiting

`deepagents` does not reimplement LLM-call resilience — no retry loop, no backoff, no circuit breaker sits inside `create_deep_agent()`. The `model=` you pass in is exactly the chat model you'd use anywhere else, so wrap it with exactly the retry discipline you already apply outside deepagents, before it ever reaches `create_deep_agent()`:

```python
from langchain_aws import ChatBedrock

model = ChatBedrock(
    model_id="anthropic.claude-sonnet-4-5-...",
    max_retries=3,          # provider SDK's own retry config
).with_retry(
    stop_after_attempt=5,   # LangChain Runnable.with_retry — a second, coarser layer
    wait_exponential_jitter=True,
)

agent = create_deep_agent(model=model, tools=support_tools, checkpointer=checkpointer)
```

`create_deep_agent()` never sees a retry policy directly — it just receives a `model` `Runnable` that already knows how to retry itself. Whether you reach for the chat model constructor's own retry/backoff parameters, `Runnable.with_retry(...)`, or a `tenacity`-decorated wrapper around your own model-construction function is exactly the same decision you'd make for any other LangChain-based service; nothing about the deep-agent context changes the calculus.

This is a genuinely separate concern from **application-level rate limiting** on the FastAPI endpoint itself — a per-user or per-tenant request quota that exists to protect your service and your cost budget, independent of whether any individual model call needs a retry. That belongs at the route or middleware layer, not inside the agent:

```python
from fastapi import HTTPException

@router.post("/chat")
async def chat(req: ChatRequest, tenant: Tenant = Depends(get_current_tenant), ...):
    if not await rate_limiter.allow(key=tenant.id):  # your existing token-bucket/Redis-backed limiter
        raise HTTPException(status_code=429, detail="rate limit exceeded")
    ...
```

Keep the two mental models distinct: model-call retries are about a single LLM request transiently failing and being worth retrying; endpoint rate limiting is about how many *requests* a given caller is allowed to make in a window, regardless of whether any of them fail. A deep agent needs both, exactly as any other production LLM service does — deepagents supplies neither, by design.

## Logging & Observability

**LangSmith tracing** is the natural fit here given the LangChain ecosystem this whole stack is built on — enabling it (via the standard `LANGSMITH_TRACING`/`LANGSMITH_API_KEY` environment variables your LangChain/LangGraph deployments already use) captures every model call, tool call, and subagent invocation inside a deep agent's run as a structured trace tree, with no additional instrumentation inside `create_deep_agent()` itself required. This is the same tracing mechanism you already rely on for any other LangGraph or LangChain service — a deep agent's richer middleware stack just means a richer trace tree to look at, not a different tracing mechanism.

What LangSmith traces don't automatically give you is correlation with *your own* application-level events — an incoming HTTP request ID, a tenant ID, the moment a `PII` request-timeout kicked in. That's where structured application logging fills the gap, bound around exactly the identifier this whole chapter has been threading through everything: `thread_id`.

```python
import structlog

logger = structlog.get_logger()

@router.post("/chat")
async def chat(req: ChatRequest, tenant: Tenant = Depends(get_current_tenant), agent=Depends(get_agent)):
    thread_id = f"{tenant.id}:{req.session_id}"
    log = logger.bind(thread_id=thread_id, tenant_id=tenant.id)

    log.info("agent_invoke_start")
    config = {"configurable": {"thread_id": thread_id}}

    async for chunk, metadata in agent.astream(
        {"messages": [{"role": "user", "content": req.message}]},
        config=config,
        stream_mode="updates",
    ):
        for node_name, node_update in chunk.items():
            if node_name == "__interrupt__":
                log.warning("agent_interrupt_raised", update=node_update)
            elif "messages" in node_update:
                for m in node_update["messages"]:
                    if getattr(m, "tool_calls", None):
                        log.info("agent_tool_call", tool_calls=[tc["name"] for tc in m.tool_calls])
    log.info("agent_invoke_end")
```

The point of this block is not the exact log schema — it's that `thread_id` is the join key between your structured application logs and the corresponding LangSmith trace: given an incident report keyed on a `thread_id`, you should be able to pull up both the application-log timeline (when did the interrupt fire, which tenant, which pod) and the LangSmith trace (which tool calls, which model calls, what the model actually saw) for the same run, without guessing at correlation. If your existing FastAPI observability stack already propagates a request ID via middleware or `contextvars`, bind that alongside `thread_id` in the same structured-log context — the two identifiers answer different questions (which HTTP request vs. which conversation/task) and you'll want both when an incident spans several requests against the same long-running `thread_id`.

### Correlating an interrupt across both layers

Chapter 9 established that `interrupt_on` pauses a run into the checkpointer and needs an application-level design for how that pause reaches a human reviewer and how the resume call gets triggered. In production, "reaches a human reviewer" and "gets triggered later" both depend on the same `thread_id`-keyed correlation this section just built — the log line raised when `__interrupt__` appears in the `stream_mode="updates"` output is what a review-queue worker or webhook handler keys off of to notify a reviewer, and the eventual approval endpoint's `Command(resume=...)` call is what closes the loop, reusing the exact `config` from the original call:

```mermaid
sequenceDiagram
    participant C as Caller
    participant F as FastAPI (/chat)
    participant A as Deep Agent (thread_id=T)
    participant PG as PostgresSaver
    participant Log as Structured Logs
    participant R as Reviewer (via /reviews/{T}/approve)

    C->>F: POST /chat  (message triggers a gated tool)
    F->>A: astream(..., config={thread_id: T})
    A->>PG: checkpoint: paused at interrupt
    A-->>Log: agent_interrupt_raised (thread_id=T)
    A-->>F: run paused, no further chunks
    F-->>C: stream ends; caller told "pending review"
    Log-->>R: (via alert/queue) review needed for T
    R->>F: POST /reviews/T/approve
    F->>A: invoke(Command(resume={"decisions":[{"type":"approve"}]}), config={thread_id: T})
    A->>PG: reads paused checkpoint for T, resumes
    A-->>F: run continues to completion
```

The only two things that make this diagram work are facts this course already established: a durable checkpointer for the pause to survive into (Ch. 9, Ch. 10), and the same `thread_id`/`config` used on both the original call and the resume call. Everything else — the review-queue mechanism, the alerting, the `/reviews/{thread_id}/approve` endpoint itself — is application architecture this chapter's structured-logging section exists to make buildable, not something `deepagents` provides.

## Cost Tracking

Three distinct levers apply here, and conflating them leads to fixing the wrong one when a bill is high.

**1. Per-request token/cost accounting.** If the custom `AgentMiddleware` from Chapter 13 wraps model calls to count tokens (via a `wrap_model_call` hook), the production version of that same middleware is what turns "the model reported N input/output tokens" into "this `thread_id`, for this tenant, cost $X on this request" — the accounting logic doesn't change, only where its output goes (a metrics backend, a per-tenant billing table) instead of a test assertion:

```python
from langchain.agents.middleware import AgentMiddleware

class TokenCostMiddleware(AgentMiddleware):
    def wrap_model_call(self, request, handler):
        response = handler(request)
        usage = response.result[-1].usage_metadata  # provider-reported token counts
        record_token_usage(  # your own metrics/billing sink
            thread_id=request.state.get("thread_id"),
            input_tokens=usage.get("input_tokens"),
            output_tokens=usage.get("output_tokens"),
        )
        return response
```

The exact `request`/`handler` shapes are Chapter 13's territory; the point for this chapter is where the counter's output goes once you're running for real — per-tenant cost dashboards, budget alerts, or a hard per-tenant spend ceiling enforced at the rate-limiter layer described above.

**2. Prompt-caching levers.** `AnthropicPromptCachingMiddleware` (Ch. 13) is always present in a deep agent's middleware stack, and the Bedrock/Fireworks equivalents are available via the `[aws]` extra — these directly reduce token cost on repeated system-prompt and memory content by turning stable, repeated prefix content into cached reads instead of full-price input tokens on every call. Chapter 7's `add_cache_control=True` on `MemoryMiddleware`-injected content is the same lever applied specifically to per-user/per-tenant memory blocks that get re-injected on every turn — turn it on for exactly the same reason: the same content re-sent on every request, for every user sharing it, is precisely the shape prompt caching exists to make cheap.

**3. Eviction and summarization tuning.** `FilesystemMiddleware`'s `tool_token_limit_before_evict` (Ch. 5) and `SummarizationMiddleware`'s trigger/keep settings (Ch. 14) are the main context-cost control knobs beyond caching — they bound how much of a large tool result or a long conversation history actually rides along in every subsequent model call. Tuning either one is a genuine production tradeoff, not a free win: a lower eviction threshold cuts token cost per call but risks more re-fetching if the model needs that content again later; a more aggressive summarization trigger keeps the context window (and therefore cost) smaller but risks losing detail the model actually needed. Treat both as levers to monitor against real production traffic (via the token-accounting middleware above) rather than values set once at design time and never revisited.

### Cost-lever summary

| Lever | What it reduces | Where it's configured | Chapter |
|---|---|---|---|
| Per-request token accounting | Nothing directly — makes cost *visible* per `thread_id`/tenant so the other levers can be tuned against real data | Custom `AgentMiddleware.wrap_model_call` | Ch. 13 |
| `AnthropicPromptCachingMiddleware` / Bedrock, Fireworks variants | Repeated input-token cost on stable system-prompt content | Always present (Anthropic); `[aws]` extra for Bedrock/Fireworks | Ch. 13 |
| `add_cache_control=True` | Repeated input-token cost on injected `memory=[...]` content | `MemoryMiddleware(..., add_cache_control=True)` | Ch. 7 |
| `tool_token_limit_before_evict` | Context size (and cost) from large tool results carried forward | `FilesystemMiddleware` construction | Ch. 5 |
| Summarization trigger/keep settings | Context size (and cost) from long conversation history | `SummarizationMiddleware` construction | Ch. 14 |

Reach for the accounting middleware first, in production, before tuning any of the other four — without real per-`thread_id`/per-tenant token data, tuning an eviction threshold or a summarization trigger is guessing at a tradeoff you haven't actually measured yet.

## Authentication

**Request-level authentication is your FastAPI app's job, exactly as it already is for every other endpoint you operate.** Nothing about `create_deep_agent()` changes how you authenticate an incoming HTTP request — an `Authorization` header, a JWT dependency, an API-gateway-terminated auth layer, whatever your existing stack already does, applies unchanged here (`get_current_tenant` in the examples above is exactly that: your existing auth dependency, not something new).

Two deepagents-specific concerns are worth calling out explicitly, because they're easy to assume are covered by request auth when they aren't:

- **MCP server auth (Ch. 11).** `MultiServerMCPClient` connections to remote MCP servers have their own auth requirements — API keys, OAuth tokens, mTLS certs, whatever the MCP server demands — entirely separate from how the caller authenticated to your FastAPI endpoint. If your deep agent's tools include MCP-sourced tools, that connection's credentials and lifecycle (ideally established once at app startup, alongside the checkpointer, not reconnected per request) are your application's responsibility to manage, same as any other MCP client code you've written outside deepagents.
- **Per-tenant `StoreBackend` namespacing (Ch. 6, Ch. 7).** If any part of your backend routes through a `StoreBackend` — persistent memory, cross-thread data — the `NamespaceFactory` is what stops tenant A's data from being readable or writable by tenant B's requests. This has to be derived from your *authenticated* identity, not from anything the caller supplies directly:

```python
# app/tenancy.py
def tenant_namespace(runtime) -> tuple[str, ...]:
    tenant_id = runtime.context["tenant_id"]  # set from authenticated identity, never from user input
    return ("memories", tenant_id)
```

The `context={"tenant_id": tenant.id}` passed into `agent.ainvoke(...)`/`agent.astream(...)` in the earlier route examples is what makes this namespace resolvable — `tenant.id` there comes from `get_current_tenant`, your auth dependency, not from any field the request body itself controls. A `NamespaceFactory` that derives its namespace from a client-supplied `session_id` or request body field instead of the authenticated identity is a cross-tenant data leak waiting to happen, not a namespacing bug you'll catch in a demo.

### MCP client lifecycle in the same lifespan

The connection-lifecycle concern above is easiest to get right by treating it exactly like the checkpointer: established once, at startup, alongside everything else in `lifespan`, rather than reconstructed per request:

```python
# app/deps.py (extended)
from langchain_mcp_adapters.client import MultiServerMCPClient

@asynccontextmanager
async def lifespan(app: FastAPI):
    with PostgresSaver.from_conn_string(DATABASE_URL) as checkpointer, \
         PostgresStore.from_conn_string(DATABASE_URL) as store:
        checkpointer.setup()
        store.setup()

        mcp_client = MultiServerMCPClient({
            "support_kb": {
                "transport": "streamable_http",
                "url": "https://mcp.internal.example.com/support-kb",
                "headers": {"Authorization": f"Bearer {MCP_SERVICE_TOKEN}"},
            },
        })
        mcp_tools = await mcp_client.get_tools()

        backend = CompositeBackend(
            default=FilesystemBackend(root_dir="/data/scratch"),
            routes={"/memories/": StoreBackend(store=store, namespace=tenant_namespace)},
        )

        app.state.agent = create_deep_agent(
            model=build_chat_model(),
            tools=support_tools + mcp_tools,
            backend=backend,
            checkpointer=checkpointer,
            store=store,
        )
        app.state.mcp_client = mcp_client
        yield
```

`MCP_SERVICE_TOKEN` here is your application's own credential for the MCP server — entirely separate from, and unrelated to, however the *caller* authenticated to `/chat`. If different tenants need to reach different MCP servers, or the same server under different per-tenant credentials, that's an application-level routing decision your `tools=` construction has to make explicitly (e.g., building a per-tenant `MultiServerMCPClient` lazily and caching it, rather than assuming one client config serves every tenant) — `deepagents` has no first-class notion of "the MCP tools for this tenant," so this stays entirely your code's responsibility, same as any other MCP client you've built outside this SDK.

## Docker, Kubernetes, and Horizontal Scaling

### A Dockerfile sketch

Nothing deepagents-specific changes the shape of this beyond making sure the right extras are installed (e.g., `deepagents[aws]` if you're using the Bedrock prompt-caching middleware variant):

```dockerfile
FROM python:3.12-slim AS base

WORKDIR /app

COPY pyproject.toml poetry.lock* ./
RUN pip install --no-cache-dir "deepagents[aws]" fastapi "uvicorn[standard]" \
        langgraph-checkpoint-postgres langgraph-store-postgres

COPY app/ ./app/

ENV PYTHONUNBUFFERED=1

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Ordinary multi-stage/dependency-caching refinements apply exactly as they would to any other FastAPI image — nothing about deepagents changes base-image choice, layer caching strategy, or non-root user hardening (Chapter 19 covers the security side of the image in more depth).

### Why `PostgresSaver`, not `SqliteSaver`, once you scale horizontally

This is Chapter 10's decision, re-stated with the actual failure mode a Kubernetes `Deployment` with `replicas: N > 1` produces if you get it wrong: `SqliteSaver` writes to a single file on a single instance's local disk. The moment a second pod exists, the load balancer can route the next request for an existing `thread_id` to a *different* pod than the one that handled the previous turn — and that pod has no access to the first pod's local SQLite file. From that pod's point of view, the `thread_id` has never been seen before: `todos` reset to nothing, files written under `StateBackend` vanish, an in-flight `interrupt_on` approval has nothing to resume into. `MemorySaver` fails the same way even *within* a single pod, the instant that pod restarts. `PostgresSaver` (or another real, network-accessible, LangGraph-compatible checkpointer) is the only one of the three that gives every pod the same view of every `thread_id`'s state, which is the entire point of the shared boxes in this chapter's architecture diagram.

The same reasoning applies identically to a `BaseStore` backing a `StoreBackend` (Ch. 6/7) — an in-process store has exactly the same single-instance failure mode as `MemorySaver`, for the same reason, and needs the same fix: a real, network-accessible store implementation shared across every pod.

### Kubernetes deployment considerations

- **Readiness/liveness probes should be lightweight, not agent-invoking.** A `/healthz` endpoint that checks the process is up and the checkpointer/store connections are alive is the correct shape; a probe that actually invokes the deep agent would tie pod health to LLM-provider latency and cost, which is not what a liveness probe is for.
- **Design request timeouts deliberately, given `recursion_limit=9999` (Ch. 2).** A raised recursion limit means a runaway or unusually deep task can legitimately consume dozens of tool-call turns and correspondingly many LLM round-trips before the graph itself would ever stop it — the graph's own limit is not a usable request-latency safety net. Set an explicit timeout at the layer that actually matters for your SLA: an ingress/load-balancer timeout, a client-side abort, or your own `asyncio.wait_for(...)`/background-task-with-cancellation pattern around the agent call — sized to your actual cost/latency budget, not left to default to "however long 9999 recursion steps happens to take."
- **Size worker/concurrency settings around I/O-bound waiting, not CPU-bound work.** A deep agent request spends nearly all of its wall-clock time waiting on LLM/tool-call network round-trips, so a single async Uvicorn worker can legitimately handle many concurrent in-flight agent runs — but "many concurrent runs" also means "many concurrent open connections to your checkpointer/store/MCP servers," which is a connection-pool sizing question for those dependencies, not a deepagents-specific one.
- **Autoscale on request concurrency/latency, not on any deepagents-specific metric** — there isn't one. Standard HPA configuration against request count, latency, or CPU (for the FastAPI process itself, which does real work marshalling SSE chunks) applies unchanged.

A sketch of what those two probe considerations look like in an actual manifest — deliberately conservative on the liveness side, since restarting a pod mid-agent-call is disruptive to whichever `thread_id`s it happened to be serving:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deepagent-support-triage
spec:
  replicas: 4
  selector:
    matchLabels: { app: deepagent-support-triage }
  template:
    metadata:
      labels: { app: deepagent-support-triage }
    spec:
      containers:
        - name: api
          image: registry.example.com/deepagent-support-triage:latest
          ports: [{ containerPort: 8000 }]
          env:
            - name: DATABASE_URL
              valueFrom: { secretKeyRef: { name: deepagent-db, key: url } }
          readinessProbe:
            httpGet: { path: /healthz, port: 8000 }
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /healthz, port: 8000 }
            initialDelaySeconds: 15
            periodSeconds: 20
            failureThreshold: 3
          resources:
            requests: { cpu: "250m", memory: "512Mi" }
            limits: { cpu: "1", memory: "1Gi" }
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: deepagent-support-triage-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: deepagent-support-triage
  minReplicas: 4
  maxReplicas: 12
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

`/healthz` here is a plain FastAPI route that pings the checkpointer/store connections and returns `200`/`503` — it never touches `app.state.agent`. The ingress or load-balancer in front of this `Deployment` (an ALB, an nginx ingress, whatever you already operate) is where the *request*-level timeout from the bullet above actually gets enforced — a value picked to match your SLA, independent of both the probes above and the graph's own `recursion_limit`.

### Decision Checklist Before Going to Production

A quick pass through the decisions this chapter assumes you've already made correctly, each one owned by an earlier chapter:

| Decision | Wrong answer for >1 replica | Right answer | Chapter |
|---|---|---|---|
| Checkpointer | `MemorySaver`, `SqliteSaver` | `PostgresSaver` (networked, shared) | Ch. 10 |
| Cross-thread store | in-process `BaseStore` | networked `BaseStore`, namespaced per tenant | Ch. 6, Ch. 7 |
| HITL resume path | none — `interrupt_on` with no application-level resume endpoint | a real approval endpoint calling `Command(resume=...)` against the *same* `config` | Ch. 9 |
| Recursion ceiling vs. request timeout | relying on `recursion_limit=9999` as the safety net | an explicit, independently-sized request timeout | Ch. 2 |
| Prompt caching | left off on repeated system-prompt/memory content | `AnthropicPromptCachingMiddleware`/Bedrock variant + `add_cache_control=True` | Ch. 7, Ch. 13 |
| Context cost | default `tool_token_limit_before_evict`/summarization settings, never revisited | tuned against real production token usage | Ch. 5, Ch. 14 |
| MCP connections | reconnect per request | established once at startup, alongside the checkpointer | Ch. 11 |

If any row in this table is still an open question for a service you're about to ship, that's the chapter to revisit before deployment, not after an incident.

## Alternative: LangSmith Managed Deep Agents

Everything above is the self-hosting path: your own FastAPI app, your own Docker image, your own Kubernetes cluster. A separate path exists via `deepagents-cli`, which provides `init`/`dev`/`deploy`-style commands for shipping a deep agent to **LangSmith Managed Deep Agents** — a distinct, LangSmith-hosted deployment target rather than infrastructure you operate yourself. This is worth knowing exists, particularly if your team is already LangSmith-centric, but given this learner's own production FastAPI/Bedrock background, self-hosting a `create_deep_agent()` graph inside infrastructure you already run and already know how to operate is almost certainly the more directly applicable path — which is why this chapter covers it in depth and treats the managed path as an alternative worth being aware of, not a replacement for the material above. (The exact CLI flags and managed-deployment configuration format weren't independently verified for this course — treat "a managed deployment path exists" as the confirmed fact, and check current `deepagents-cli` documentation directly before depending on specific command syntax.)

## Real-World Scenario

A team ships an internal support-triage deep agent — the same one built up across Chapters 4–9: a `todos`-driven triage checklist, filesystem tools for drafting responses, `interrupt_on` gating anything before it's sent to a customer. It started as a single-instance prototype behind FastAPI, using `SqliteSaver` because it was simple to run locally and the demo never needed more than one process. As ticket volume grew, the team scaled the Kubernetes `Deployment` to four replicas behind an ALB — and immediately started seeing tickets where a customer's follow-up message produced a triage agent with no memory of the ticket's prior `todos`, because the ALB had routed the follow-up to a different pod than the one holding the original SQLite file. The fix was exactly Chapter 10's decision, now enforced by real infrastructure instead of reasoned about hypothetically: migrate to `PostgresSaver`, shared across all four pods, so any replica handling any request for a given `thread_id` sees identical `todos`/`files` state.

Two more gaps surfaced once Postgres-backed checkpointing was in place. First, an `interrupt_on` gate in front of "send this response to the customer" — correct in isolation — had no application-level design for how a paused run actually reached a human reviewer; the team built a separate `/reviews/{thread_id}/approve` endpoint that looks up the pending review, constructs the same `config` the original call used, and calls `agent.invoke(Command(resume={"decisions": [{"type": "approve"}]}), config=config)` — the resume half of Chapter 9's contract, now wired to a real reviewer-facing UI rather than left as a notebook cell. Second, once the agent's memory (Ch. 7) began storing per-customer context in a `StoreBackend`, a `NamespaceFactory` keyed correctly on the authenticated support-agent's tenant, not on any client-supplied field, was what kept two different support organizations' customer data from ever landing in the same namespace — precisely the "Authentication" section's warning above, discovered in production rather than caught in review.

## Best Practices

- **Construct the checkpointer, store, and compiled agent once, at process startup, via `lifespan`** — never per-request. Rebuilding `create_deep_agent()` on every call wastes setup cost and risks subtly different middleware configuration drifting across requests.
- **Never let `MemorySaver` or `SqliteSaver` reach a horizontally-scaled deployment.** `PostgresSaver` (or an equivalent real, network-accessible, LangGraph-compatible checkpointer) is required the moment more than one pod/process can serve the same `thread_id`.
- **Design an explicit request timeout independent of `recursion_limit=9999`.** The raised limit is a deliberate "don't artificially throttle legitimate deep tasks" choice (Ch. 2), not a production safety net — your own timeout, sized to your actual SLA, is the safety net.
- **Keep model-call retries and endpoint rate limiting conceptually and structurally separate.** Wrap `model=` with your own retry/backoff discipline before it reaches `create_deep_agent()`; enforce request quotas at the FastAPI route/middleware layer. Neither substitutes for the other.
- **Bind `thread_id` into every structured log line for a request**, and treat it as the join key to the corresponding LangSmith trace during incident investigation.
- **Derive `NamespaceFactory` namespaces from authenticated identity, never from client-supplied fields.** This is the single highest-leverage multi-tenant correctness check in this chapter.
- **Treat prompt-caching flags (`AnthropicPromptCachingMiddleware`, Bedrock/Fireworks variants, `add_cache_control=True`) as a checklist item for any deployment with repeated system-prompt or memory content**, not an optional micro-optimization — the cost delta on real production traffic is usually not small.
- **Manage MCP client connections at the same lifecycle layer as the checkpointer** — established once at startup, reused across requests — rather than reconnecting per call.
- **Load-test with an actually long-running task before shipping**, specifically to observe what your chosen request timeout, worker concurrency, and readiness-probe configuration do when a task legitimately runs for dozens of tool-call turns.

## Common Mistakes

- **Shipping `MemorySaver` — or `SqliteSaver` — behind more than one replica.** This is the single most common production failure mode with deep agents specifically: it doesn't produce an error, it produces *silent* state loss — a `thread_id` that "has never been seen" from the pod that happens to receive the next request, with `todos`, files, and pending interrupts all quietly gone. Nothing in normal load-balanced traffic surfaces this until a customer notices their conversation forgot everything.
- **No request-level timeout around an agent call, relying on `recursion_limit=9999` as if it were a production safety net.** It isn't one — it's a deliberately raised ceiling so legitimate deep tasks aren't artificially throttled (Ch. 2). A runaway or unusually deep task really can consume dozens of LLM round-trips before the graph decides to stop, and without your own timeout, that shows up as a stuck HTTP request, an exhausted worker slot, or a bill nobody expected — not a `GraphRecursionError` you can catch.
- **Forgetting per-tenant `StoreBackend` namespacing**, or deriving the `NamespaceFactory`'s namespace from something a caller controls rather than from authenticated identity — leading directly to cross-tenant memory leakage: one tenant's persisted preferences, findings, or conversation memory becoming readable (or worse, writable) by another tenant's requests.
- **Reconnecting to MCP servers on every request instead of managing the connection lifecycle at app startup**, adding avoidable per-request latency and, depending on the MCP server, needlessly exercising its auth/connection-setup path far more than necessary.
- **Conflating model-call retry configuration with endpoint rate limiting**, leading to a service that retries a failing Bedrock call five times per request while having no actual cap on how many requests a single caller can issue per minute — two different problems, solved by two different layers, neither one covering for the other.
- **Treating LangSmith tracing as sufficient observability on its own**, with no structured application-level logging keyed on `thread_id` — leaving no way to correlate "what the model saw" (the trace) with "what your application did" (auth decisions, rate-limit rejections, which pod handled the request) for the same incident.

## Summary

Deploying a `create_deep_agent()` graph to production is not a new discipline — it's your existing FastAPI/Docker/Kubernetes/LangGraph discipline, applied to a `CompiledStateGraph` whose specific middleware stack you now understand in depth. The FastAPI integration is standard LangGraph streaming (`.astream(stream_mode="messages")` into an SSE `StreamingResponse`) with `thread_id` and a shared `PostgresSaver` wired through the framework's own lifespan/dependency-injection pattern; `.ainvoke`/`.astream` are the identical `CompiledStateGraph` methods you'd use anywhere else, with no separate async construction path. Retry policies belong on the `model=` you pass in, wrapped exactly as you'd wrap any other chat model, kept conceptually distinct from endpoint-level rate limiting. LangSmith tracing and `thread_id`-keyed structured application logs are complementary, not redundant, observability layers. Cost control has three concrete levers — per-request token accounting via a custom `wrap_model_call` middleware, prompt-caching middleware/`add_cache_control`, and eviction/summarization tuning — each addressing a different part of the token bill. Authentication is your app's job as always, except for two deepagents-specific concerns worth designing deliberately: MCP server connection auth, and per-tenant `StoreBackend` namespacing derived from authenticated identity. And the whole thing only actually scales horizontally once `PostgresSaver` (never `MemorySaver`, and not `SqliteSaver` past a single instance) and a real shared `BaseStore` replace their single-process equivalents — the same decision Chapter 10 introduced, now enforced by an actual multi-replica Kubernetes deployment rather than reasoned about in the abstract. A managed alternative exists via `deepagents-cli` and LangSmith Managed Deep Agents, worth knowing about even if self-hosting remains the more directly applicable path given this course's assumed background.

## Knowledge Check

1. Why is `.ainvoke`/`.astream` not "the async version of the deep agent" — what, precisely, is identical between the sync and async call paths?
2. A team runs their deep agent behind a Kubernetes `Deployment` with `replicas: 4`, using `SqliteSaver`. Describe the exact failure a customer would observe, and explain why it happens even though `SqliteSaver` genuinely does persist state to disk.
3. Why is `recursion_limit=9999` not a substitute for an application-level request timeout? What concretely could go wrong in production if a team relied on it as one?
4. Name the three distinct cost-control levers this chapter discussed, and give one sentence on what each one actually reduces (token count per call vs. cost per cached token vs. something else).
5. A `NamespaceFactory` derives its namespace from a `session_id` field the client supplies directly in the request body, rather than from an authenticated `tenant_id`. What's the concrete production risk, and how would you fix it?
6. Distinguish, with an example of each, a model-call retry concern from an application-level rate-limiting concern in a deep agent's FastAPI service.

## Hands-On Exercise

Containerize a minimal deep agent FastAPI service with a `PostgresSaver`-backed checkpointer, end to end:

1. Write a small FastAPI app (`app/main.py`, `app/deps.py`) with a `lifespan` that opens a `PostgresSaver.from_conn_string(...)` connection to a local Postgres instance (a `docker run postgres:16` container is sufficient), calls `checkpointer.setup()`, and constructs one `create_deep_agent()` graph stored on `app.state`.
2. Add a `POST /chat` route that derives `thread_id` from a request body field, builds `config`, and calls `agent.ainvoke(...)`, returning the final assistant message.
3. Add a `POST /chat/stream` route using `agent.astream(..., stream_mode="messages")` piped into an SSE `StreamingResponse`, per this chapter's example.
4. Write a Dockerfile for the service per this chapter's sketch, and a `docker-compose.yml` (or two `docker run` invocations) running the FastAPI container alongside a Postgres container on the same network, with `DATABASE_URL` pointing at the Postgres container's service name.
5. Prove the horizontal-scaling property directly: run **two instances** of your FastAPI container (two separate `docker run`/`docker compose up --scale` processes) pointed at the same Postgres container, send a first message to instance A with a given `thread_id`, then send a follow-up message for the *same* `thread_id` to instance B. Confirm instance B's response reflects state (e.g., prior `todos`, if your agent uses planning) that only instance A had produced — this is the concrete proof that `PostgresSaver` gives you the multi-replica property `SqliteSaver` cannot.
6. As a stretch: repeat step 5 with `checkpointer=SqliteSaver.from_conn_string("local.db")` mounted only into instance A's container, and observe instance B fail to see any prior state for the same `thread_id` — a direct, hands-on demonstration of this chapter's central common mistake.

## Further Reading

- [DeepAgents Overview (LangChain Docs)](https://docs.langchain.com/oss/python/deepagents/overview)
- [`langchain-ai/deepagents` GitHub repository](https://github.com/langchain-ai/deepagents) — read `libs/deepagents/deepagents/graph.py` to confirm the `checkpointer=`/`store=` passthrough and the `recursion_limit=9999` configuration this chapter's timeout guidance depends on
- [LangSmith documentation](https://docs.smith.langchain.com/) — tracing setup and configuration for the observability layer described above
- Chapter 2 of this course (Architecture & Internals) — the `CompiledStateGraph`/`recursion_limit=9999` facts this chapter's timeout and probe guidance builds on
- Chapter 6 of this course (Backends & Storage Architecture) — `StoreBackend`, `CompositeBackend`, `NamespaceFactory`, the mechanism behind this chapter's multi-tenant namespacing section
- Chapter 9 of this course (Human-in-the-Loop) — `interrupt_on`/`Command(resume=...)`, the mechanism behind this chapter's approval-endpoint scenario
- Chapter 10 of this course (Checkpointing & Durable Execution) — the full `MemorySaver`/`SqliteSaver`/`PostgresSaver` decision this chapter assumes and enforces with real infrastructure
- Chapter 11 of this course (MCP Integration) — `MultiServerMCPClient`, the mechanism behind this chapter's MCP-auth/connection-lifecycle note

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./17-testing-and-evaluation.md">← Previous: Testing & Evaluation</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./19-security-and-governance.md">Next: Security & Governance →</a>
</div>
