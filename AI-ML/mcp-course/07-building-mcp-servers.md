# Building MCP Servers

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain, in concrete terms, why cramming database queries and API calls directly inside `@mcp.tool()`-decorated
  functions is an anti-pattern, and what specifically breaks as a server grows past a handful of tools
- Lay out a real MCP server project — `server.py`, `tools/`, `resources/`, `prompts/`, `services/`, `config/` —
  and explain the responsibility each directory carries
- Write MCP decorator functions as **thin adapters**: input validation and shape conversion only, with all
  business logic delegated to a `services/` layer that has no knowledge that MCP exists
- Register tools, resources, and prompts that live in separate files against one shared `FastMCP` instance,
  and navigate the import-order subtlety that makes this work
- Centralize configuration (database URLs, API keys, tunables) in a single Pydantic `Settings` class instead of
  scattering `os.environ[...]` lookups across tool files
- Unit test a service function — and the thin tool function that wraps it — with plain `pytest`, with no MCP
  client, no transport, and no running server process involved
- Build a small, complete analytics MCP server end to end using this layout, and recognize the same shape when
  you meet it in a production codebase

---

## Prerequisites for This Chapter

This chapter assumes Chapters 4–6: you already know what a tool, resource, and prompt object looks like on the
wire, how `@mcp.tool()`/`@mcp.resource()`/`@mcp.prompt()` register a single function against a `FastMCP`
instance, and what a minimal one-file `FastMCP("Demo")` server looks like. This chapter does not re-explain any
of that — it answers the next question every one of those chapters left open: *what happens once you have
fifteen tools, four resources, and three prompts instead of one of each?*

You should also be comfortable with:

- Ordinary Python package structure (`__init__.py`, relative vs. absolute imports, how `import` executes module
  top-level code exactly once and caches the result in `sys.modules`)
- `async`/`await` — the worked example uses async tool and service functions throughout, consistent with the
  fact sheet's note that the SDK is natively async
- Basic `pytest` and `pytest-asyncio` usage
- Pydantic v2 basics; `pydantic-settings`'s `BaseSettings` if you haven't used it before, this chapter introduces
  it at the level you need

This chapter does not cover transport selection (stdio vs. Streamable HTTP — Chapter 8), schema design quality
(Chapter 10), or debugging with MCP Inspector (Chapter 12). It is scoped narrowly to **project structure and the
protocol/business-logic boundary**.

---

## 1. The Anti-Pattern: Business Logic Inside Tool Functions

Here is how almost every MCP server starts, and how a worrying number stay:

```python
# server.py — everything in one file, business logic inline
from mcp.server.fastmcp import FastMCP
import asyncpg

mcp = FastMCP("analytics-server")

@mcp.tool()
async def get_daily_active_users(day: str) -> int:
    """Return the count of distinct users active on the given day."""
    pool = await asyncpg.create_pool("postgresql://user:pass@localhost/analytics")
    async with pool.acquire() as conn:
        row = await conn.fetchrow(
            "SELECT count(DISTINCT user_id) AS n FROM events WHERE event_date = $1",
            day,
        )
        return row["n"] if row else 0

if __name__ == "__main__":
    mcp.run()
```

For a one-tool demo this is fine. It stops being fine the moment a second tool needs the same connection pool,
a third tool needs the same query logic with a different filter, or anyone tries to write a test for
`get_daily_active_users` without a live Postgres instance and a running MCP process. Named concretely, the
problems that compound as this pattern scales:

1. **The pool gets recreated per call, or gets duplicated per tool.** Nothing in the function above shares
   connection state with any other tool — the natural next step is either a global mutable pool (fragile,
   initialization-order-dependent) or copy-pasting the `asyncpg.create_pool(...)` line into every tool function.
2. **Business logic becomes untestable without MCP in the loop.** To verify the SQL is correct, you need a real
   database *and* a way to invoke the decorated function — which, depending on how the decorator is written,
   may or may not still be a plain callable. Either way, the query logic and the protocol plumbing are welded
   together, so you can't test one without dragging in the other.
3. **The same logic can't be reused outside MCP.** The moment someone wants a REST endpoint or a CLI command
   that reports "daily active users," the only reusable artifact is a function wrapped in an MCP-specific
   decorator, in a file that imports `mcp.server.fastmcp`. Copy-pasting the SQL into a second place is how the
   two implementations quietly drift out of sync.
4. **Credentials and config end up hardcoded or scattered.** The connection string above is inline; in a
   slightly-less-naive version it's an `os.environ["DATABASE_URL"]` call repeated in every tool file, with no
   single place to see or validate what configuration the whole server depends on.

None of these are protocol problems — they're ordinary software-architecture problems that MCP servers are not
exempt from just because the entry point is a decorator instead of a route or a CLI command. The fix is the
same one you'd apply to a Flask app or a Typer CLI: **separate the thin adapter layer from the logic it calls.**

---

## 2. The Recommended Project Layout

```
mcp-server/
├── server.py            # Composition root: creates the FastMCP instance, imports
│                         # every tools/resources/prompts module, starts the server
├── tools/
│   ├── __init__.py
│   ├── database.py       # @mcp.tool() adapters for DB-backed tools
│   ├── analytics.py       # @mcp.tool() adapters for analytics tools
│   └── search.py          # @mcp.tool() adapters for search tools
├── resources/
│   ├── __init__.py
│   └── datasets.py         # @mcp.resource() adapters
├── prompts/
│   ├── __init__.py
│   └── report_templates.py # @mcp.prompt() adapters
├── services/
│   ├── __init__.py
│   ├── analytics_service.py  # actual business logic: SQL, aggregation, formatting
│   ├── search_service.py
│   └── database_service.py
├── config/
│   ├── __init__.py
│   └── settings.py         # one Pydantic Settings class, one source of truth
└── tests/
    ├── test_analytics_service.py   # tests services/ directly — no MCP involved
    └── test_tools_analytics.py     # tests tools/ adapters — still no MCP protocol involved
```

Each directory has exactly one job:

| Directory | Owns | Does **not** own |
|---|---|---|
| `server.py` | Constructing the `FastMCP` instance, importing every registration module, starting the server | Any business logic |
| `tools/`, `resources/`, `prompts/` | Translating between MCP's shape (typed function signatures, docstrings, return values the SDK can serialize) and a call into `services/` | Database queries, HTTP calls, aggregation logic, anything with a retry policy |
| `services/` | The actual work: querying a database, calling an external API, computing an aggregate, formatting a domain object | Anything that imports `mcp.server.fastmcp` — a service module should not know MCP exists |
| `config/` | One typed, validated view of every environment-derived setting the server needs | Ad hoc `os.environ[...]` calls anywhere else in the codebase |

The rule that keeps this honest: **if you can imagine writing the same functionality as a REST endpoint or a
CLI command, the code inside `services/` should be exactly what that REST endpoint or CLI command would also
call.** The `tools/` layer's only job is adapting `services/` output into whatever a specific protocol expects.

---

## 3. `server.py` — The Composition Root

`server.py` is deliberately thin. It creates the shared `FastMCP` instance, pulls in every module that registers
something against it, and starts the process:

```python
# server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("analytics-server")

# Each import below runs that module's top-level code exactly once. Since every
# tool/resource/prompt module below does `from server import mcp` and then applies
# @mcp.tool()/@mcp.resource()/@mcp.prompt() at import time, importing a module here
# IS how it registers itself — there is no separate "register everything" call.
import tools.analytics    # noqa: E402,F401
import tools.database     # noqa: E402,F401
import tools.search       # noqa: E402,F401
import resources.datasets           # noqa: E402,F401
import prompts.report_templates     # noqa: E402,F401

if __name__ == "__main__":
    mcp.run()
```

Two things worth being deliberate about here:

**Import order matters, and it's the whole trick.** `mcp = FastMCP("analytics-server")` must execute *before*
`import tools.analytics` runs, because `tools/analytics.py` does `from server import mcp` at its own top level.
This looks circular — `server.py` imports `tools.analytics`, which imports `server` — and it *is* circular, but
it works: by the time Python starts executing `tools/analytics.py`, `server` is already present in
`sys.modules` (partially initialized, mid-execution), and its `mcp` attribute already exists because the
assignment on the line above ran first. `from server import mcp` inside `tools/analytics.py` finds that
attribute on the cached, in-progress module object. Reorder the two lines in `server.py` — `import
tools.analytics` before `mcp = FastMCP(...)` — and it breaks immediately with an `ImportError`, because at that
point `server.mcp` doesn't exist yet.

This pattern is common enough in real FastMCP projects that it's worth understanding rather than just copying,
but if the circularity ever makes you uneasy (e.g., because a linter or a stricter import-order tool complains),
the safe alternative is to move `mcp = FastMCP(...)` into its own tiny module — say, `config/mcp_instance.py` —
that both `server.py` and every `tools/`/`resources/`/`prompts/` module import from. Nothing then imports
`server` itself except whatever actually runs the process, and the circularity disappears entirely. Either
approach is fine; know which one you're using and why.

**`mcp.run()` starts the server.** The fact sheet's worked examples stop at defining tools and don't show the
run call in detail — what to pass to `mcp.run()` to select stdio vs. Streamable HTTP, and any additional
constructor-time options `FastMCP(...)` accepts for shaping HTTP responses, is genuinely transport-configuration
territory. Chapter 8 covers that surface properly; treat `mcp.run()` here as "the thing that starts the process
once every tool/resource/prompt has been registered," and confirm the exact transport-selection arguments
against your installed SDK version before relying on a specific keyword signature from memory.

---

## 4. `config/` — Centralized Settings with Pydantic

Every credential, connection string, and tunable the server needs should come from exactly one place: a
`pydantic-settings` `BaseSettings` subclass. This has nothing to do with MCP specifically — it's the same
practice you'd apply to any FastAPI service — but it matters more here than usual, because tool/service files
are easy to write quickly and `os.environ["SOME_KEY"]` calls are the path of least resistance if you don't have
a settings object already sitting there to import.

```python
# config/settings.py
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="ANALYTICS_")

    database_url: str
    api_key: str | None = None
    max_query_rows: int = 10_000
    query_timeout_seconds: float = 5.0


settings = Settings()
```

`services/analytics_service.py` imports `settings` and never touches `os.environ` directly:

```python
from config.settings import settings

class AnalyticsService:
    def __init__(self, dsn: str | None = None) -> None:
        self._dsn = dsn or settings.database_url
```

The `dsn: str | None = None` parameter on the constructor is deliberate — it lets tests construct an
`AnalyticsService` against a test database or a mock without touching the real `Settings` object at all
(Section 8 uses exactly this).

One validated `Settings` instance also means a missing or malformed environment variable fails loudly at import
time — before the server ever accepts a connection — rather than surfacing as a `KeyError` three call-stack
frames deep inside whichever tool happened to touch that variable first.

---

## 5. `services/` — Where the Real Logic Lives

A service module is ordinary Python: no `mcp` import, no decorators, no awareness that a protocol exists on the
other side of it. It should be indistinguishable from a service module you'd write for a FastAPI app.

```python
# services/analytics_service.py
import asyncpg

from config.settings import settings


class AnalyticsService:
    """All analytics business logic. No MCP import anywhere in this file —
    this class would work identically behind a REST endpoint or a CLI command."""

    def __init__(self, dsn: str | None = None) -> None:
        self._dsn = dsn or settings.database_url
        self._pool: asyncpg.Pool | None = None

    async def _get_pool(self) -> asyncpg.Pool:
        if self._pool is None:
            self._pool = await asyncpg.create_pool(self._dsn)
        return self._pool

    async def daily_active_users(self, day: str) -> int:
        pool = await self._get_pool()
        async with pool.acquire() as conn:
            row = await conn.fetchrow(
                "SELECT count(DISTINCT user_id) AS n FROM events WHERE event_date = $1",
                day,
            )
            return row["n"] if row else 0

    async def top_events(self, day: str, limit: int = 10) -> list[dict]:
        limit = min(limit, settings.max_query_rows)
        pool = await self._get_pool()
        async with pool.acquire() as conn:
            rows = await conn.fetch(
                "SELECT event_name, count(*) AS n FROM events "
                "WHERE event_date = $1 GROUP BY event_name ORDER BY n DESC LIMIT $2",
                day,
                limit,
            )
            return [dict(r) for r in rows]


# One shared instance the tools/ layer imports — the pool is lazily created on
# first use and reused across every tool call that goes through this instance.
analytics_service = AnalyticsService()
```

Notice `limit = min(limit, settings.max_query_rows)` — a business rule (don't let a caller request an
unbounded number of rows) lives here, in the service, not duplicated across every tool that happens to expose a
`limit` parameter. That's the entire point of the layer: rules like this get written once.

---

## 6. `tools/` — Thin Protocol Adapters

A tool module's job is narrow: define the function signature the LLM will see (parameter names, types,
docstring), and delegate immediately to the service layer.

```python
# tools/analytics.py
from server import mcp
from services.analytics_service import analytics_service


@mcp.tool()
async def get_daily_active_users(day: str) -> int:
    """Return the count of distinct users active on the given day (YYYY-MM-DD)."""
    return await analytics_service.daily_active_users(day)


@mcp.tool()
async def get_top_events(day: str, limit: int = 10) -> list[dict]:
    """Return the top `limit` event names by count for the given day (YYYY-MM-DD)."""
    return await analytics_service.top_events(day, limit=limit)
```

Nothing in `tools/analytics.py` opens a connection, builds SQL, or applies a business rule. If a tool needs to
reshape a service's return value into something a different shape than what the service naturally produces
(say, the service returns a domain object and the tool needs a plain dict for JSON serialization), that
reshaping is the *only* logic that belongs at this layer — genuine adaptation, not computation.

`resources/` and `prompts/` follow the identical shape:

```python
# resources/datasets.py
from server import mcp


@mcp.resource("analytics://schema")
def get_schema() -> str:
    """Expose the events table schema so the LLM can write reasonable questions
    against get_top_events / get_daily_active_users."""
    return (
        "Table: events(user_id TEXT, event_name TEXT, "
        "event_date DATE, properties JSONB)"
    )
```

```python
# prompts/report_templates.py
from server import mcp


@mcp.prompt()
def daily_report_prompt(day: str) -> str:
    """Prompt template for generating a daily analytics report."""
    return (
        f"Using the get_daily_active_users and get_top_events tools for {day}, "
        "write a two-paragraph executive summary of the day's activity."
    )
```

A resource or a prompt that needs real data (say, a resource listing "all dataset names currently available,"
which requires a database round trip) should call into `services/` exactly the same way `tools/analytics.py`
does — the thin-adapter rule doesn't only apply to tools.

---

## 7. Registering Primitives Across Multiple Files

Stepping back from the specific files above, the general pattern for any multi-file FastMCP project is:

1. Exactly one `FastMCP(...)` instance exists for the whole server (created in `server.py`, or in a small
   dedicated module every other file imports from).
2. Every module under `tools/`, `resources/`, `prompts/` imports that instance and applies
   `@mcp.tool()`/`@mcp.resource()`/`@mcp.prompt()` to functions defined in that module, at import time.
3. `server.py` (or a package `__init__.py`, if you prefer that convention) imports every one of those modules
   for their **registration side effect** — the import itself is what causes the decorator to run and the
   function to be added to the server's tool/resource/prompt list. Nothing calls a separate "register" function;
   `import tools.analytics` *is* the registration step.

```mermaid
flowchart TD
    subgraph Entry["server.py — composition root"]
        M["mcp = FastMCP('analytics-server')"]
    end

    subgraph Adapters["Thin adapter layer"]
        TA["tools/analytics.py<br/>@mcp.tool() functions"]
        TD["tools/database.py<br/>@mcp.tool() functions"]
        TS["tools/search.py<br/>@mcp.tool() functions"]
        RD["resources/datasets.py<br/>@mcp.resource() functions"]
        PR["prompts/report_templates.py<br/>@mcp.prompt() functions"]
    end

    subgraph Services["services/ — business logic, no MCP import"]
        SA["AnalyticsService"]
        SD["DatabaseService"]
        SS["SearchService"]
    end

    Config["config/settings.py<br/>Settings (Pydantic)"]
    DB[(Database / External APIs)]

    M -->|"import (registers via decorator)"| TA
    M -->|"import (registers via decorator)"| TD
    M -->|"import (registers via decorator)"| TS
    M -->|"import (registers via decorator)"| RD
    M -->|"import (registers via decorator)"| PR

    TA --> SA
    TD --> SD
    TS --> SS
    RD --> SA

    SA --> Config
    SD --> Config
    SS --> Config
    SA --> DB
    SD --> DB
    SS --> DB
```

If you'd rather avoid the "import for side effect" pattern entirely — some teams find `# noqa: F401` comments
on unused imports uncomfortable — a valid alternative is to have each `tools/*.py` module export a plain
undecorated function, and apply `mcp.tool()(fn)` programmatically from a single loop in `server.py`. This is
exactly equivalent (a decorator is just syntax sugar for calling the decorator function with the decorated
function as its argument), but makes the registration step visually explicit at the cost of one extra layer of
indirection. Either approach is fine — pick one and apply it consistently across the whole project so a new
contributor only has to learn the pattern once.

> **2026-07-28 spec note:** none of the registration mechanics above change under the new stateless spec —
> `@mcp.tool()`/`@mcp.resource()`/`@mcp.prompt()` are the same decorator names in `v2.0.0`'s `MCPServer` per the
> fact sheet. What changes is what happens *underneath* those decorators at request-handling time (no
> handshake, every request self-describing) — Chapter 21 covers that. The project-layout advice in this chapter
> is SDK-generation-agnostic: it's about where business logic lives, not about wire mechanics.

---

## 8. Testing Without the Protocol Layer

This is the single biggest practical payoff of the service-layer split. A `services/` object is a plain Python
class with no MCP import — it is testable with ordinary `pytest`, no MCP client, no transport, no running
server process:

```python
# tests/test_analytics_service.py
import pytest

from services.analytics_service import AnalyticsService


class FakeConnection:
    def __init__(self, dau: int) -> None:
        self._dau = dau

    async def fetchrow(self, query: str, day: str):
        return {"n": self._dau}


class FakePool:
    def __init__(self, dau: int) -> None:
        self._conn = FakeConnection(dau)

    def acquire(self):
        return self  # supports `async with pool.acquire() as conn:`

    async def __aenter__(self):
        return self._conn

    async def __aexit__(self, *exc):
        return False


@pytest.mark.asyncio
async def test_daily_active_users_returns_count(monkeypatch):
    service = AnalyticsService(dsn="postgresql://unused-in-test")
    monkeypatch.setattr(service, "_get_pool", lambda: FakePool(dau=42))

    result = await service.daily_active_users("2026-07-29")

    assert result == 42
```

Because `tools/analytics.py`'s `@mcp.tool()`-decorated functions are still ordinary Python functions after
decoration — the decorator's job is to *register* the function against `mcp`, and idiomatic FastMCP-style
decorators leave the function itself directly callable — the thin adapter layer is testable too, with the
service mocked out instead of the database:

```python
# tests/test_tools_analytics.py
import pytest

import tools.analytics as analytics_tools


@pytest.mark.asyncio
async def test_get_daily_active_users_delegates_to_service(monkeypatch):
    async def fake_dau(day: str) -> int:
        assert day == "2026-07-29"
        return 42

    monkeypatch.setattr(
        analytics_tools.analytics_service, "daily_active_users", fake_dau
    )

    result = await analytics_tools.get_daily_active_users("2026-07-29")

    assert result == 42
```

Neither test spins up a `FastMCP` server, opens a transport, or performs an `initialize` handshake. There is no
JSON-RPC anywhere in this file. That's deliberate: the protocol layer's job is serialization and routing, and
those concerns are exactly what MCP Inspector (Chapter 12) and integration tests against a real running server
are for. Unit tests belong at the layer where the actual logic lives — which, with this project structure, is
almost never the protocol layer itself.

---

## 9. FastMCP Instantiation Options

`FastMCP(name)` — the single positional string argument shown throughout the fact sheet and this chapter — is
the part of the constructor you can rely on across the v1.x line: it sets the server's identity, surfaced to
clients in the `initialize` response's `serverInfo.name` field (Chapter 3).

Beyond the name, some FastMCP releases and community tutorials show additional constructor-time keyword
arguments that shape how the HTTP transport behaves once a server is running over Streamable HTTP — for
example, whether responses come back as plain JSON versus an SSE-framed stream. Exactly which keyword options
your installed SDK version exposes, and what their defaults are, is transport-configuration detail this chapter
deliberately doesn't pin down — Chapter 8 is the right place to verify the current surface against whatever
`mcp` version you have installed, rather than hard-coding a specific keyword argument from memory here. The
takeaway for this chapter is narrower and doesn't depend on that detail: **one `FastMCP(name)` instance, created
once, shared by every `tools/`/`resources/`/`prompts/` module that registers against it.**

---

## Examples: A Complete Analytics MCP Server

Putting Sections 3–7 together, here is the full project, file by file, exactly as it would sit on disk (some
files trimmed to their essential shape where earlier sections already showed the complete version):

```
analytics-mcp-server/
├── server.py
├── tools/
│   ├── __init__.py
│   └── analytics.py
├── resources/
│   ├── __init__.py
│   └── datasets.py
├── prompts/
│   ├── __init__.py
│   └── report_templates.py
├── services/
│   ├── __init__.py
│   └── analytics_service.py
├── config/
│   ├── __init__.py
│   └── settings.py
├── tests/
│   ├── test_analytics_service.py
│   └── test_tools_analytics.py
├── .env
└── pyproject.toml
```

`config/settings.py`, `services/analytics_service.py`, `tools/analytics.py`, `resources/datasets.py`,
`prompts/report_templates.py`, and `server.py` are exactly as given in Sections 4–6 above — nothing new to add
there. A `.env` alongside them supplies the settings the `Settings` class expects:

```bash
# .env
ANALYTICS_DATABASE_URL=postgresql://analytics:analytics@localhost:5432/analytics
ANALYTICS_MAX_QUERY_ROWS=5000
```

Running it (stdio, the default a bare `mcp.run()` call typically selects — confirm against your SDK version):

```bash
uv run server.py
# or, to exercise it interactively without writing a client at all:
uv run mcp dev server.py
```

`uv run mcp dev server.py` launches MCP Inspector (Chapter 12) against this exact server — a fast way to confirm
`get_daily_active_users`, `get_top_events`, the `analytics://schema` resource, and the `daily_report_prompt`
prompt all show up and behave as expected, before wiring in a real LangChain/LangGraph client (Chapter 17–18).

A second consumer reusing the same `services/` layer — a small FastAPI endpoint that has nothing to do with
MCP — demonstrates why the split paid off:

```python
# A hypothetical REST endpoint, reusing the exact same service the MCP tool uses
from fastapi import FastAPI
from services.analytics_service import analytics_service

app = FastAPI()

@app.get("/analytics/daily-active-users/{day}")
async def daily_active_users_endpoint(day: str) -> dict:
    return {"day": day, "count": await analytics_service.daily_active_users(day)}
```

Nothing about `AnalyticsService` needed to change to make this work. The query logic, the connection pooling,
the `max_query_rows` business rule — all of it is shared, unmodified, between the MCP server and this REST
endpoint. This is the payoff the anti-pattern in Section 1 makes structurally impossible.

```mermaid
flowchart LR
    LLM["LLM Host<br/>(via MCP client)"] -->|"tools/call<br/>get_daily_active_users"| MCPTool["tools/analytics.py<br/>@mcp.tool()"]
    RESTClient["REST client"] -->|"GET /analytics/daily-active-users/2026-07-29"| RESTEndpoint["FastAPI endpoint"]
    CLIUser["CLI user"] -->|"analytics-cli dau 2026-07-29"| CLICommand["Typer/argparse command"]

    MCPTool --> Service["services/analytics_service.py<br/>AnalyticsService.daily_active_users()"]
    RESTEndpoint --> Service
    CLICommand --> Service

    Service --> DB[(Postgres)]
```

Three entry points, one implementation of the actual logic — exactly the reuse argument from Section 1,
made concrete.

---

## Real-World Scenario

A team's first MCP server started as a single `server.py` with eight `@mcp.tool()` functions, each opening its
own `asyncpg` connection inline, each with its SQL string built by concatenating a base query with a filter
clause assembled from the tool's arguments. It worked, and it shipped. Three things then happened in sequence:

**A security review flagged the query construction.** One tool built a `WHERE` clause by string-formatting a
user-supplied `event_name` argument directly into SQL, because it was "just internal" and nobody had reviewed
it against the rest of the codebase's (correctly) parameterized queries elsewhere. Because the query logic
lived inline inside the tool function, there was no single place to check "does every query in this server use
parameterized SQL" — the answer required reading eight separate functions in eight separate files, several of
which had drifted stylistically from each other since being written by different people at different times.

**Product asked for the same daily-active-users number in an internal dashboard.** The fastest path available
was copying the query out of the MCP tool function into a new FastAPI route, which is exactly the "quietly
drift out of sync" failure mode from Section 1 — three weeks later, someone tightened the MCP tool's date-range
handling to exclude a known-bad ingestion window, and nobody remembered to make the same change in the
dashboard's copy-pasted version. The two numbers disagreed in a way that took an afternoon to root-cause.

**Testing the tools required a live database, every time.** With no service layer, "does `get_top_events`
correctly cap `limit`" could only be verified by running the whole server against a real (or heavily mocked at
the connection level) Postgres instance, because the query construction and the business rule were both baked
into the same function the MCP decorator wrapped.

The fix was exactly Sections 2–6: extract each tool's logic into a `services/` class, replace every ad hoc
`asyncpg.create_pool(...)` with the shared `AnalyticsService`/`DatabaseService` pattern, and rewrite the
`@mcp.tool()` functions as thin delegators. The security review became a one-file audit of `services/`. The
dashboard became a second consumer of `AnalyticsService` instead of a second copy of the query. And the test
suite for `limit` capping became the kind of five-line `pytest` test shown in Section 8 — no database, no MCP
process, running in milliseconds as part of CI.

---

## Best Practices

- **Keep `@mcp.tool()`/`@mcp.resource()`/`@mcp.prompt()` functions thin.** If a tool function's body is longer
  than "validate/reshape arguments, call a service method, maybe reshape the result," the logic belongs in
  `services/` instead.
- **Never let a service module import `mcp.server.fastmcp`.** That single rule is the fastest way to check
  whether the boundary is holding — if a `services/` file needs anything from the MCP SDK, logic has leaked
  across the line.
- **One `Settings` class, one source of truth.** Route every credential and tunable through it; resist adding a
  second `os.environ[...]` call anywhere else in the codebase once `config/settings.py` exists.
- **Accept dependencies through constructor parameters with sensible defaults** (`AnalyticsService(dsn: str |
  None = None)`), so tests can substitute a fake without needing to monkeypatch global state.
- **Import registration modules for their side effect, and be deliberate about the import order** if you keep
  the `FastMCP` instance in `server.py` itself — or sidestep the subtlety entirely with a dedicated instance
  module if that's uncomfortable for your team's linting setup.
- **Test the service layer first, the thin adapters second.** The service layer carries the actual risk
  (correctness of queries, business rules); the adapter layer mostly just needs confirming it calls the right
  service method with the right arguments.
- **Design `services/` as if a REST endpoint or CLI command will consume it eventually** — even if none exists
  yet. That discipline is what makes reuse actually available later instead of requiring a rewrite.

---

## Common Mistakes

- **Writing SQL/API-call logic directly inside a tool function.** This is the anti-pattern this whole chapter
  exists to name — it's fine for a five-minute demo and actively harmful past that point (Section 1, Real-World
  Scenario).
- **Reordering `server.py`'s `mcp = FastMCP(...)` line after the `import tools....` lines.** This breaks the
  circular-import trick from Section 3 with an `ImportError`, because `tools/analytics.py`'s `from server
  import mcp` needs `server.mcp` to already exist when it runs.
- **Scattering `os.environ[...]` calls across `tools/`, `services/`, and anywhere else**, instead of routing
  everything through one `Settings` object. This makes it impossible to answer "what does this server need
  configured" without grepping the whole codebase.
- **Duplicating a service's logic into a second file** (a REST endpoint, a CLI command) instead of importing
  the existing service class — this is exactly how the dashboard/MCP-tool numbers disagreed in the Real-World
  Scenario.
- **Trying to unit test a tool function by spinning up a full `FastMCP` server and driving it through a client
  session.** That's an integration test, and it's the wrong tool for verifying "does this function call the
  right service method" — Section 8's plain `pytest` approach is faster and more precise for that question.
  Reserve full-server tests (Chapter 12's MCP Inspector, or a real client session) for verifying the protocol
  surface itself.
- **Letting `services/` accumulate a hidden dependency on `mcp`** — e.g., a service method that returns an MCP
  content-block dict instead of a plain domain value, because "that's the shape the tool needs anyway." This
  quietly re-couples the two layers; keep the service's return values protocol-agnostic and let the thin
  adapter do any final reshaping.

---

## Summary

- A one-file MCP server with business logic inline in `@mcp.tool()` functions works for a demo and breaks down
  as tools multiply: duplicated connection setup, untestable logic, and no way to reuse the same behavior
  outside MCP.
- The fix is the same layered architecture you'd apply to any service: `tools/`/`resources/`/`prompts/` as thin
  protocol adapters, `services/` holding the actual business logic with zero knowledge that MCP exists,
  `config/` centralizing settings in one Pydantic `Settings` class.
- `server.py` is the composition root: it creates the one shared `FastMCP` instance and imports every
  registration module for its side effect — importing a module *is* how its `@mcp.tool()`/`@mcp.resource()`/
  `@mcp.prompt()` functions get registered.
- The circular `from server import mcp` pattern inside `tools/`/`resources/`/`prompts/` modules works because of
  Python's module-caching semantics, provided `mcp = FastMCP(...)` executes before those modules are imported;
  a dedicated instance module sidesteps the subtlety if you'd rather avoid it.
- Because a service is a plain Python class and a thin tool function is still an ordinary callable after
  decoration, both layers are testable with plain `pytest` — no MCP client, no transport, no running server
  process required for unit-level testing.
- The same `services/` layer that backs an MCP tool can back a REST endpoint or a CLI command unmodified — that
  reuse is the structural payoff of keeping business logic out of the protocol-adapter layer.

---

## Knowledge Check

1. A tool function's body directly builds a SQL string and executes it against a freshly created connection
   pool. Name at least three concrete problems this causes as the server grows past one or two tools.
2. In `server.py`, why must `mcp = FastMCP(...)` execute before `import tools.analytics` if `tools/analytics.py`
   contains `from server import mcp`? What error do you get if the order is reversed, and why?
3. What is the one rule that tells you whether business logic has leaked from `services/` into `tools/` (or vice
   versa)? Give a concrete example of a violation.
4. Explain why a service-layer class like `AnalyticsService` should accept its dependencies (e.g., a `dsn`)
   through constructor parameters rather than reading `config.settings.settings` directly inside every method.
5. Why can `tools/analytics.py`'s `get_daily_active_users` function be unit tested with plain `pytest`, with no
   MCP client or server process involved? What specifically about the decorator pattern makes this possible?
6. A teammate proposes returning an MCP content-block dict (`{"type": "text", "text": ...}`) directly from a
   `services/` method, "since that's what the tool needs anyway." What's wrong with this, and what should the
   method return instead?

---

## Hands-On Exercise

Extend the analytics MCP server from the Examples section with a second capability, `search`, following the
exact same layered pattern — and prove the layering actually holds by testing it without touching MCP at all.

1. **Add `services/search_service.py`** with a `SearchService` class exposing one async method,
   `search_events(query: str, limit: int = 10) -> list[dict]`, that (for this exercise) searches an in-memory
   list of sample event dicts by substring match on `event_name` rather than hitting a real database — keep the
   exercise focused on structure, not on wiring up search infrastructure.

2. **Add `tools/search.py`** with a single `@mcp.tool()`-decorated `search_events` function that does nothing
   but validate arguments and delegate to `SearchService.search_events`.

3. **Register it** by adding `import tools.search` to `server.py`, respecting the import-order rule from
   Section 3.

4. **Write `tests/test_search_service.py`** that constructs a `SearchService` directly (no MCP import anywhere
   in the test file) and asserts `search_events("checkout")` returns the expected matching events, with no
   database and no running server.

5. **Write `tests/test_tools_search.py`** that imports `tools.search`, monkeypatches
   `tools.search.search_service.search_events` with a fake, and asserts the tool function delegates correctly —
   mirroring Section 8's `test_tools_analytics.py`.

6. **Bonus:** run `uv run mcp dev server.py` and confirm `search_events` shows up alongside
   `get_daily_active_users` and `get_top_events` in MCP Inspector, with no changes needed to `config/` or any
   other existing file — that absence of ripple effect is the layout doing its job.

---

## Further Reading

- `github.com/modelcontextprotocol/python-sdk` — official Python SDK; read `mcp/server/fastmcp/` directly for
  the current `FastMCP` constructor signature and decorator implementations
- `github.com/PrefectHQ/fastmcp` — the actively maintained standalone FastMCP project; its documentation and
  example repositories show multi-file project layouts in more depth than the bundled SDK's own docs
- [Pydantic Settings documentation](https://docs.pydantic.dev/latest/concepts/pydantic_settings/) — the
  `BaseSettings`/`SettingsConfigDict` pattern used throughout `config/settings.py` in this chapter
- Related chapter in this course: [Chapter 4 — MCP Tools](./04-mcp-tools.md) — the single-tool, single-file
  foundation this chapter builds a multi-file project on top of
- Related chapter in this course: [Chapter 8 — Transport Mechanisms](./08-transport-mechanisms.md) — what
  `mcp.run()` and `FastMCP(...)`'s transport-shaping options actually do, deliberately deferred from this
  chapter
- Related chapter in this course: [Chapter 12 — MCP Inspector & Debugging](./12-mcp-inspector-and-debugging.md)
  — testing the protocol surface itself, once unit tests at the service/tool layer are already in place
- Related chapter in this course: [Chapter 20 — Production MCP Architecture](./20-production-mcp-architecture.md)
  — what this same layered project structure needs on top of it (retries, rate limiting, observability) to run
  in production

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./06-mcp-prompts.md">← Previous: MCP Prompts</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./08-transport-mechanisms.md">Next: Transport Mechanisms →</a>
</div>
