# MCP + Databases

## Learning Objectives

By the end of this chapter, you will be able to:

- Name and compare the two dominant patterns for database-backed MCP tools — the generic `execute_sql` tool and purpose-built domain-specific tools
- Explain, with concrete reasoning tied to the injection surface, auditability, and rate-limiting problems, why production AI systems should default to domain-specific tools
- Write parameterized queries for MCP tool implementations and explain why string-formatted SQL is never acceptable, even "just for an internal tool"
- Apply the tool → service → data-access separation from Chapter 7 to a real, async, connection-pooled PostgreSQL-backed MCP server
- Generalize the domain-specific pattern to MongoDB (aggregation-pipeline-backed tools) and describe, at a conceptual level, how it extends to Elasticsearch
- Explain why a long-lived MCP server process should own a single connection pool rather than opening a connection per tool call, and what that costs you if you get it wrong

## Prerequisites

This chapter builds directly on:

- **Chapter 4** (MCP Tools) — tool definitions, schemas, and results
- **Chapter 7** (Building MCP Servers) — the tool-function → service-function → business-logic separation this chapter applies to a database backend specifically
- **Chapter 10** (Tool Schema Design) — narrow, well-typed schemas that an LLM can call correctly; this chapter is the concrete database case study for that chapter's principles
- **Chapter 11** (Error Handling) — protocol errors vs. tool execution errors, needed when a query fails or times out
- **Chapter 14** (MCP Security) — least privilege, sandboxing, and injection-style attack patterns; this chapter is the database-specific application of that reasoning

You should also be comfortable with, independent of MCP:

- Writing async Python against a database driver (`asyncpg`, `psycopg`, `pymongo`/`motor`) — this chapter does not teach those drivers from scratch
- Basic SQL and the concept of a parameterized/prepared statement
- What a connection pool is and why opening a raw TCP connection per request is expensive

If any of the database-driver material is unfamiliar, treat this chapter as "how MCP tools sit on top of database code you already know how to write," not as a database tutorial.

---

## 1. Two Dominant Patterns for Database-Backed Tools

Almost every non-trivial MCP server eventually needs to answer questions backed by a database. There are, in practice, exactly two shapes this takes:

| | Generic (`execute_sql`) | Domain-specific |
|---|---|---|
| Tool surface | One tool, arbitrary query string in, rows out | Several tools, each named and typed for one business question |
| Example | `execute_sql(query: str) -> list[dict]` | `get_daily_sales(start_date, end_date, region=None)` |
| What the model must get right | An entire SQL dialect, correctly, every time | A handful of scalar/date arguments |
| Injection surface | The entire SQL grammar | None — no user- or model-supplied string ever becomes part of the query structure |
| Auditability | "Ran some query" — you have to parse SQL after the fact to know what happened | "Called `get_daily_sales` for region=APAC, 2026-07-01..2026-07-30" — self-describing |
| Rate limiting / quotas | Meaningless at the query level (a `SELECT 1` and a five-way join both look like "one call") | Meaningful (N calls/minute to a known, bounded-cost operation) |
| Blast radius of a mistake | Anything the DB credential can reach | Whatever that one tool's query touches, by design |

Both patterns are real and both show up in production systems. The rest of this chapter argues, in detail, that **domain-specific is the default** and generic `execute_sql` is, at most, a tightly-gated administrative capability — not something you hand to a general-purpose agent.

```mermaid
flowchart TD
    subgraph Generic["Pattern A: Generic execute_sql"]
        M1[Model decides *what SQL to write*] --> T1["execute_sql(query: str)"]
        T1 --> DB1[(Database)]
    end
    subgraph Domain["Pattern B: Domain-specific tools"]
        M2[Model decides *which tool, with what typed args*] --> T2["get_daily_sales(start_date, end_date, region)"]
        T2 --> S2[Service function]
        S2 --> Q2[Fixed, parameterized query]
        Q2 --> DB2[(Database)]
    end
```

Notice the difference in *where the risk lives*. In Pattern A, the model is effectively a SQL author — every call is a fresh opportunity to generate something subtly or catastrophically wrong. In Pattern B, the model is a caller of a fixed, reviewed, tested function; the query itself was written once, by a human, and never changes shape based on model output.

## 2. Pattern A: The Generic `execute_sql` Tool

It's worth being honest about why this pattern is so tempting, before dismantling it.

```python
# mcp v1.x (classic) — the generic pattern, shown so you can recognize and critique it
from mcp.server.fastmcp import FastMCP
import asyncpg

mcp = FastMCP("Generic DB Access")

@mcp.tool()
async def execute_sql(query: str) -> list[dict]:
    """Run an arbitrary read-only SQL query against the analytics database
    and return the resulting rows."""
    conn = await asyncpg.connect(DATABASE_URL)
    try:
        rows = await conn.fetch(query)
        return [dict(r) for r in rows]
    finally:
        await conn.close()
```

This looks appealing for a reason: it's roughly ten lines, it answers *any* question the schema can express, and you never have to anticipate what the user will ask. One tool, unbounded coverage. For a prototype, or for a tool you personally invoke through the MCP Inspector while debugging (Chapter 12), that tradeoff can be fine.

It stops being fine the moment a model — not you — is the one composing the query string, for these concrete reasons:

1. **The entire SQL grammar is the attack surface.** Chapter 14 covers prompt injection and tool poisoning in general; a generic SQL tool is the database-specific worst case of the same problem. Any text the model has seen — a user's message, a poisoned document it retrieved, a malicious tool description from another server — can influence the query string it generates. There is no schema constraint (Chapter 10) narrowing what "valid input" looks like; `query: str` accepts literally anything, including `DROP TABLE`, `UNION`-based exfiltration of unrelated tables, or a `WHERE 1=1` that returns every row in a customer table instead of one.
2. **You cannot express least privilege at the tool boundary.** Chapter 14's least-privilege principle wants each capability scoped as tightly as possible. `execute_sql` can't be scoped tighter than "whatever the underlying DB credential can do" — the tool has no concept of "this operation only ever touches sales aggregates for a bounded date range." The credential becomes the *only* enforcement point, and DB credentials are usually provisioned per-application, not per-tool-call.
3. **Read-only intent is a promise, not a guarantee.** A docstring that says "read-only" does not stop the model from emitting `DELETE FROM orders`. Enforcing actual read-only behavior needs a read-only DB role or a query-parsing allowlist — real controls, not comments — and even then, an expensive `SELECT` with a ten-way cross join is a resource-exhaustion vector the docstring says nothing about.
4. **Meaningful rate limiting and auditing require knowing what happened.** "37 calls to `execute_sql` in the last minute" tells you nothing about blast radius — one call might be `SELECT NOW()` and another might be a full-table export. Domain-specific tools give you a call count *per business operation*, which is the granularity every downstream system (billing, anomaly detection, incident response) actually wants.
5. **The model has to get SQL right, from scratch, every single call.** This is a schema-design failure in Chapter 10's terms: you're asking the model to correctly generate an entire query language instead of populating a handful of typed fields. Even a well-behaved model will occasionally misquote a date, forget a `WHERE` clause, or misjudge a join — and because the "schema" here is just `query: str`, nothing catches the mistake before it reaches the database.

> **2026-07-28 spec note:** none of this changes with the stateless redesign — `tools/call` still carries whatever arguments the tool's `inputSchema` accepts, and a schema of `{"query": {"type": "string"}}` is exactly as unconstrained under the new spec as under the classic one. The fix here is a tool-design decision, not a protocol-version decision.

## 3. Pattern B: Domain-Specific Tools

The domain-specific alternative names the *business question*, not the query mechanism, and constrains the model to a small, typed set of arguments:

```python
@mcp.tool()
async def get_daily_sales(
    start_date: str,
    end_date: str,
    region: str | None = None,
) -> list[dict]:
    """Get total daily sales figures between two dates (inclusive), optionally
    filtered to one region. Dates are ISO-8601 (YYYY-MM-DD)."""
    ...

@mcp.tool()
async def get_vehicle_entries(
    start_time: str,
    end_time: str,
    gate_id: str,
    vehicle_type: str | None = None,
) -> list[dict]:
    """Get vehicle entry events at a specific gate within a time window,
    optionally filtered by vehicle type (e.g. 'car', 'truck', 'two_wheeler')."""
    ...

@mcp.tool()
async def get_passenger_footfall(
    start_time: str,
    end_time: str,
    site_id: str,
    zone_id: str | None = None,
) -> list[dict]:
    """Get aggregated passenger/visitor footfall counts for a site within a
    time window, optionally scoped to one zone within the site."""
    ...
```

Each of these is deliberately narrow, per Chapter 10's schema-design guidance: a bounded date/time range, an optional categorical filter, and nothing else. The model cannot ask this server to do anything the tool author didn't already anticipate — which is exactly the point. The query behind `get_vehicle_entries` is fixed at development time, reviewed once, tested once, and never changes shape based on what the model passes in; only the *parameter values* change.

This is a direct instance of Chapter 10's core argument: schemas aren't documentation for a human, they're the interface the model reasons against. A `start_date`/`end_date`/`region` triple is something a model fills in correctly essentially every time. An open-ended `query: str` inviting a full SQL query is something a model gets *approximately* right — which, for a database, is a much scarier failure mode than getting a REST parameter approximately right.

## 4. Why Production Systems Should Default to Domain-Specific Tools

Pulling the previous two sections together into an explicit default:

- **The injection surface shrinks to zero.** There is no string the model supplies that ever becomes part of the query's structure — only typed leaf values (a date, an ID, an enum-like string) that flow into a parameterized query as *data*, never as *code*. This is the database-specific instance of Chapter 14's injection-defense guidance, and it's a stronger guarantee than any amount of input sanitization on a free-text query string can offer.
- **Least privilege becomes expressible at the tool level, not just the credential level.** `get_daily_sales` can be backed by a DB role that can only `SELECT` from a `sales_daily_agg` view — and even if it couldn't, the tool itself structurally cannot run anything but that one shape of query. You get defense in depth instead of a single point of failure.
- **Auditing and rate limiting become meaningful.** A log line reading `get_vehicle_entries(gate_id=G3, vehicle_type=truck, 2026-07-29T00:00→2026-07-30T00:00)` is directly actionable — you know exactly what was asked, of what scope, and can rate-limit or alert on it per business operation instead of per opaque SQL string.
- **The model succeeds more often, with less prompt engineering.** Chapter 10's central lesson applies at full force here: a handful of typed, well-named parameters is something the model fills in reliably; an entire query language is not. Fewer malformed calls means fewer wasted round-trips and less need to coach the model with elaborate SQL-writing instructions in the tool description.
- **Cost is bounded and predictable.** Each domain-specific tool maps to a known query shape with a known worst-case cost (an indexed range scan over a bounded window, say). A generic `execute_sql` tool has no such ceiling — a poorly-formed join can be an accidental denial-of-service against your own database, which ties directly into Chapter 14's "unbounded resource reads" pattern.

The generalizable rule: **design MCP database tools around the questions your agents actually need answered, not around the shape of your query language.** If you find yourself building `execute_sql` because you can't predict what questions will come up, that's a signal to talk to the people building the agent about what they actually need — not a reason to hand the model a live SQL console.

## 5. If You Must Keep `execute_sql`: Treat It as Break-Glass

Some organizations have a real, narrow need for ad-hoc querying — an internal data-analyst tool, a support-engineering "debug this customer's data" workflow. That's a legitimate use case, but it is an **administrative capability**, not a general-purpose agent tool, and should be treated with the same caution Chapter 14 applies to any high-privilege operation:

- **Gate it behind a separate, explicitly-scoped MCP server** (or a distinct tool namespace) that ordinary agent deployments never connect to. Don't mix it into the same server that exposes `get_daily_sales` to a customer-facing assistant.
- **Back it with a read-only database role**, at minimum, and ideally a role scoped to specific schemas/views rather than the full database.
- **Require a human in the loop** — either a human directly issuing the query through the MCP Inspector (Chapter 12), or an approval step before an agent-issued query executes, never a fully autonomous agent with unattended access to `execute_sql`.
- **Log every query verbatim**, tied to the identity that invoked it, and alert on write-shaped statements (`INSERT`/`UPDATE`/`DELETE`/`DROP`/`ALTER`) even against a read-only role, since a rejected write attempt is itself a signal worth seeing.
- **Never expose it to the same agent context that also processes untrusted input** (retrieved documents, third-party tool output, arbitrary user text) — that's precisely the condition under which prompt injection turns "a debugging convenience" into "an attacker-controlled query."

Put simply: `execute_sql` is a maintenance/on-call capability, gated like `kubectl exec` into production, not a tool you hand to a general-purpose customer-facing or even internal-facing autonomous agent.

## 6. Parameterized Queries: The Non-Negotiable Baseline

Whether you build domain-specific tools (which should never need this discussion to go wrong) or an admin-gated `execute_sql`, the underlying rule is the same one Chapter 14 gives for any injection-prone boundary: **never build a query by string-formatting untrusted input into it.**

```python
# NEVER DO THIS — string-formatted SQL, full injection surface
async def get_daily_sales_BAD(start_date: str, end_date: str, region: str | None, conn):
    region_clause = f"AND region = '{region}'" if region else ""
    query = f"""
        SELECT sale_date, SUM(amount) AS total
        FROM sales
        WHERE sale_date BETWEEN '{start_date}' AND '{end_date}'
        {region_clause}
        GROUP BY sale_date
        ORDER BY sale_date
    """
    return await conn.fetch(query)   # region = "x' OR '1'='1" breaks this instantly
```

```python
# ALWAYS DO THIS — parameterized query, no string ever becomes query structure
async def get_daily_sales_GOOD(start_date: str, end_date: str, region: str | None, conn) -> list[dict]:
    query = """
        SELECT sale_date, SUM(amount) AS total
        FROM sales
        WHERE sale_date BETWEEN $1 AND $2
          AND ($3::text IS NULL OR region = $3)
        GROUP BY sale_date
        ORDER BY sale_date
    """
    rows = await conn.fetch(query, start_date, end_date, region)
    return [dict(r) for r in rows]
```

`asyncpg` (and every other serious driver — `psycopg`'s `%s` placeholders, `pymongo`'s dict-shaped filters) sends parameters to the database **separately from the query text**; the database's own query planner treats them strictly as data, never as SQL to parse. This is true regardless of whether the value came from a trusted config file or an LLM's tool call — parameterization is what makes the *source* of the value irrelevant to injection risk. This is the one piece of database hygiene that has nothing to do with MCP specifically and everything to do with Chapter 14's broader point: the defense against injection lives in how you build the query, not in how much you trust whoever supplied the input.

## 7. Worked Example: An Analytics MCP Server over PostgreSQL

This section builds a small but complete analytics MCP server, applying Chapter 7's tool-function → service-function → data-access separation to a real async PostgreSQL backend.

### 7.1 Project Layout

```
analytics-mcp-server/
├── server.py            # FastMCP instance + @mcp.tool() functions (thin)
├── db.py                 # connection pool lifecycle
├── services/
│   ├── sales.py          # get_daily_sales business logic + query
│   └── vehicles.py        # get_vehicle_entries business logic + query
└── models.py             # optional: typed row shapes (TypedDict/dataclass)
```

This mirrors Chapter 7's guidance directly: **tool functions stay thin** (schema, docstring, calling one service function) and **all query logic, connection handling, and business rules live in the services layer**, independently testable without an MCP client in the loop at all.

### 7.2 The Connection Pool (`db.py`)

```python
# db.py — owns the single pool for the server's lifetime
import asyncpg

_pool: asyncpg.Pool | None = None

async def init_pool(dsn: str) -> asyncpg.Pool:
    global _pool
    _pool = await asyncpg.create_pool(
        dsn=dsn,
        min_size=2,
        max_size=10,
        command_timeout=10,   # seconds — bound worst-case query latency
    )
    return _pool

async def close_pool() -> None:
    if _pool is not None:
        await _pool.close()

def get_pool() -> asyncpg.Pool:
    if _pool is None:
        raise RuntimeError("Connection pool not initialized — call init_pool() at server startup")
    return _pool
```

### 7.3 Service Functions (`services/sales.py`, `services/vehicles.py`)

```python
# services/sales.py — business logic + parameterized query, no MCP imports at all
from db import get_pool

async def get_daily_sales(start_date: str, end_date: str, region: str | None = None) -> list[dict]:
    query = """
        SELECT sale_date, region, SUM(amount) AS total_amount, COUNT(*) AS order_count
        FROM sales
        WHERE sale_date BETWEEN $1 AND $2
          AND ($3::text IS NULL OR region = $3)
        GROUP BY sale_date, region
        ORDER BY sale_date
    """
    pool = get_pool()
    async with pool.acquire() as conn:
        rows = await conn.fetch(query, start_date, end_date, region)
        return [dict(r) for r in rows]
```

```python
# services/vehicles.py
from db import get_pool

async def get_vehicle_entries(
    start_time: str, end_time: str, gate_id: str, vehicle_type: str | None = None
) -> list[dict]:
    query = """
        SELECT entry_time, gate_id, vehicle_type, plate_hash
        FROM vehicle_entries
        WHERE entry_time BETWEEN $1 AND $2
          AND gate_id = $3
          AND ($4::text IS NULL OR vehicle_type = $4)
        ORDER BY entry_time
    """
    pool = get_pool()
    async with pool.acquire() as conn:
        rows = await conn.fetch(query, start_time, end_time, gate_id, vehicle_type)
        return [dict(r) for r in rows]
```

Note the same shape as the "GOOD" example in Section 6, and note that these functions are ordinary `async def`s with no MCP or FastMCP imports — you can unit-test `get_daily_sales` and `get_vehicle_entries` directly against a test database, with no MCP client, no protocol layer, and no model involved at all. That testability is the entire reward for keeping the services layer separate, per Chapter 7.

### 7.4 Tool Functions (`server.py`)

```python
# server.py — mcp v1.x (classic); thin tool layer, calls services, owns startup/shutdown
from contextlib import asynccontextmanager
from mcp.server.fastmcp import FastMCP

import db
from services import sales, vehicles

DATABASE_URL = "postgresql://analytics_ro:***@db.internal:5432/analytics"

@asynccontextmanager
async def lifespan(_mcp: FastMCP):
    await db.init_pool(DATABASE_URL)   # one pool, created once, at process startup
    try:
        yield
    finally:
        await db.close_pool()          # drained cleanly on server shutdown

mcp = FastMCP("Analytics", lifespan=lifespan)

@mcp.tool()
async def get_daily_sales(start_date: str, end_date: str, region: str | None = None) -> list[dict]:
    """Get total daily sales (amount and order count) between two dates
    (inclusive, ISO-8601 YYYY-MM-DD), optionally filtered to one region."""
    return await sales.get_daily_sales(start_date, end_date, region)

@mcp.tool()
async def get_vehicle_entries(
    start_time: str, end_time: str, gate_id: str, vehicle_type: str | None = None
) -> list[dict]:
    """Get vehicle entry events at a gate within a UTC time window (ISO-8601),
    optionally filtered by vehicle_type ('car', 'truck', 'two_wheeler')."""
    return await vehicles.get_vehicle_entries(start_time, end_time, gate_id, vehicle_type)

if __name__ == "__main__":
    mcp.run()
```

Every tool function here does exactly two things: describe itself to the model (name, docstring, typed parameters — the schema FastMCP derives from the signature) and delegate to a service function. No query text, no connection handling, and no business logic live at this layer — which is exactly Chapter 7's separation of protocol concerns from business logic, applied to a database backend.

```mermaid
sequenceDiagram
    participant Model as LLM (via agent)
    participant Tool as get_daily_sales (tool fn)
    participant Svc as sales.get_daily_sales (service fn)
    participant Pool as asyncpg.Pool
    participant DB as PostgreSQL

    Model->>Tool: tools/call get_daily_sales(start_date, end_date, region)
    Tool->>Svc: await sales.get_daily_sales(...)
    Svc->>Pool: pool.acquire()
    Pool-->>Svc: pooled connection (reused, not opened fresh)
    Svc->>DB: parameterized query ($1, $2, $3)
    DB-->>Svc: rows
    Svc-->>Tool: list[dict]
    Tool-->>Model: tools/call result (content: text/structuredContent)
```

## 8. Connection Pooling: Why the Server Owns the Pool

An MCP server process, once started, typically lives for the lifetime of a long-running deployment — hours, days, or weeks — serving many `tools/call` invocations across that lifetime, often from multiple concurrent agent sessions. That's a fundamentally different lifecycle from a short-lived script, and it changes the right answer for connection management.

**Wrong for this lifecycle:**

```python
# Anti-pattern: open a fresh connection on every tool call
@mcp.tool()
async def get_daily_sales_slow(start_date: str, end_date: str) -> list[dict]:
    conn = await asyncpg.connect(DATABASE_URL)   # TCP handshake + auth, every single call
    try:
        return [dict(r) for r in await conn.fetch(QUERY, start_date, end_date)]
    finally:
        await conn.close()
```

This works, but pays the full cost of a TCP handshake, TLS negotiation (if applicable), and database authentication on *every* tool call — latency that has nothing to do with the query itself and everything to do with connection setup. Under concurrent load (several agent sessions calling tools at once), it also has no ceiling: nothing stops the server from opening dozens of simultaneous raw connections and exhausting the database's `max_connections` limit, which fails every *other* client of that database too, not just this MCP server.

**Right for this lifecycle:** create one pool at process startup (Section 7.4's `lifespan` hook), size it deliberately (`min_size`/`max_size`), and have every tool call `acquire()` a connection from that pool and release it back when done. The pool amortizes connection setup across the server's entire uptime, and `max_size` gives you an explicit, tunable ceiling on how much concurrent load this one server can put on the database — a form of the same resource-bounding discipline Chapter 14 asks for at the security layer, applied here for capacity planning.

Concretely, this is why `db.init_pool()` runs once in the `lifespan` context manager rather than inside a tool function: the pool's lifetime should match the *server process's* lifetime, not any individual tool call's. Chapter 20 goes further into sizing pools under real production load, health-checking pooled connections, and handling pool exhaustion gracefully (backpressure vs. queuing vs. failing fast) — treat this section as the foundational reason that discussion exists, not the full production treatment.

| | Per-call connection | Server-owned pool |
|---|---|---|
| Latency per tool call | Pays handshake + auth every time | Pays it once, amortized over server lifetime |
| Concurrency ceiling | None — unbounded simultaneous connections possible | Explicit (`max_size`), tunable, protects the DB from this server |
| Failure mode under load | Database refuses new connections; every client of the DB is affected | This server's calls queue or fail fast at a known ceiling; blast radius is contained |
| Fits MCP server lifecycle | No — designed for short-lived scripts | Yes — matches a long-lived process serving many calls |

## 9. Generalizing to MongoDB: Aggregation-Pipeline-Backed Tools

The same generic-vs-domain-specific argument, and the same services-layer separation, applies unchanged to MongoDB — only the query mechanism differs. Instead of parameterized SQL, the "fixed query shape, variable leaf values" pattern becomes a **fixed aggregation pipeline** whose stage arguments are populated from typed tool parameters:

```python
# services/footfall.py — MongoDB, same tool→service separation, aggregation instead of SQL
from datetime import datetime
from pymongo import AsyncMongoClient  # or motor.motor_asyncio.AsyncIOMotorClient

_client: AsyncMongoClient | None = None

def get_db():
    return _client["analytics"]

async def get_passenger_footfall(start_time: str, end_time: str, site_id: str, zone_id: str | None = None) -> list[dict]:
    match_stage: dict = {
        "site_id": site_id,
        "timestamp": {"$gte": datetime.fromisoformat(start_time), "$lt": datetime.fromisoformat(end_time)},
    }
    if zone_id is not None:
        match_stage["zone_id"] = zone_id

    pipeline = [
        {"$match": match_stage},
        {"$group": {"_id": "$zone_id", "total_footfall": {"$sum": "$count"}}},
        {"$sort": {"_id": 1}},
    ]
    db = get_db()
    cursor = db["footfall_events"].aggregate(pipeline)
    return [doc async for doc in cursor]
```

The security and design argument carries over exactly: `match_stage` is built from typed leaf values (a `site_id` string, parsed datetimes, an optional `zone_id`) that MongoDB's driver serializes as BSON data — never as JavaScript or query operators the caller controls. That's the Mongo-world equivalent of a parameterized SQL query: the pipeline's *shape* (`$match` → `$group` → `$sort`) is fixed by the service function, and only leaf values vary. A generic `execute_mongo_query(pipeline: list[dict])` tool would recreate exactly the same problem as `execute_sql` — the model could supply a `$where` stage with arbitrary JavaScript, an unbounded `$lookup`, or a pipeline that scans an entire collection — so the same "domain-specific by default, generic gated as break-glass" rule applies.

A Mongo client, like an `asyncpg.Pool`, is itself connection-pooled internally and should be created once at server startup and reused across tool calls — the same "server owns the pool" reasoning from Section 8 applies without modification.

## 10. Generalizing to Elasticsearch: Search-Backed Tools (Conceptual)

The pattern extends the same way to Elasticsearch, without needing a full worked example to see the shape: instead of exposing a generic `execute_es_query(query_dsl: dict)` tool that hands the model Elasticsearch's full Query DSL, expose domain-specific search tools — `search_incident_logs(keyword, start_time, end_time, severity=None)`, `find_similar_support_tickets(ticket_text, max_results=10)` — where the service function builds a fixed query template (a `bool` query with `must`/`filter` clauses, or a `match`/`knn` query for semantic search) and only the leaf values (the keyword, the date range, the severity enum) come from the tool's typed parameters. The Elasticsearch (or OpenSearch) client, like the Postgres pool and the Mongo client, is created once at server startup and reused, and the same case against a generic query-DSL-passthrough tool applies: an open `query_dsl: dict` parameter lets the model construct arbitrarily expensive aggregations or reach indices the tool author never intended to expose.

The throughline across all three backends: **the query mechanism changes (SQL, aggregation pipeline, Query DSL), but the tool-design principle does not.** Fix the query shape at development time behind a reviewed service function; let only scalar, typed leaf values vary at call time; never let the model supply anything that becomes query *structure* rather than query *data*.

## Examples

### Example 1: Refactoring a generic tool into domain-specific ones

Given an existing `execute_sql` tool that an internal dashboard team has been using to answer ad-hoc questions, the refactor is mechanical once you look at query logs for the last month: group the queries by shape (ignoring literal values), and each recurring shape becomes one domain-specific tool.

```python
# Before: every one of these came through the same execute_sql tool
"SELECT SUM(amount) FROM sales WHERE sale_date BETWEEN '2026-07-01' AND '2026-07-31'"
"SELECT SUM(amount) FROM sales WHERE sale_date BETWEEN '2026-06-01' AND '2026-06-30' AND region = 'APAC'"
"SELECT COUNT(*) FROM vehicle_entries WHERE gate_id = 'G3' AND entry_time > '2026-07-29'"
```

```python
# After: two domain-specific tools cover the actual shapes seen in production
@mcp.tool()
async def get_daily_sales(start_date: str, end_date: str, region: str | None = None) -> list[dict]: ...

@mcp.tool()
async def get_vehicle_entries(start_time: str, end_time: str, gate_id: str, vehicle_type: str | None = None) -> list[dict]: ...
```

The refactor doesn't lose capability the dashboard team was actually using — it removes the capability nobody asked for (arbitrary joins, arbitrary tables, arbitrary write statements) while keeping every query shape that appeared in real traffic.

### Example 2: Testing the services layer without MCP at all

Because `services/sales.py` has no MCP imports, it's testable with an ordinary async test, independent of any MCP client, tool schema, or model:

```python
import pytest
from services import sales

@pytest.mark.asyncio
async def test_get_daily_sales_filters_by_region(db_pool):
    rows = await sales.get_daily_sales("2026-07-01", "2026-07-31", region="APAC")
    assert all(r["region"] == "APAC" for r in rows)
```

This is the direct payoff of Chapter 7's services-layer separation: the entire correctness of the business logic can be verified before a single MCP tool call ever happens.

## Real-World Scenario

A smart-facilities analytics vendor runs an MCP server exposing site operations data — vehicle entries at parking gates, passenger footfall by zone, daily sales at on-site retail — to an internal ops-assistant agent used by facility managers ("How many trucks entered Gate 3 yesterday?", "What was footfall in the north wing this week?"). Early on, an engineer prototyped this with a single `execute_sql` tool against a read replica, reasoning "we don't know every question facility managers will ask, so let the model write the query."

Within weeks, three problems surfaced: (1) a facility manager asked a vague question, the model generated a query with an unindexed full-table scan, and it degraded the replica for every other read consumer during business hours; (2) a support engineer debugging a complaint noticed the agent had, in one session, retrieved rows spanning a different site's data because the model's `WHERE` clause omitted a `site_id` filter the human would never have forgotten; (3) when leadership asked "what has this agent actually been querying," the answer was a pile of ad-hoc SQL strings that took a data engineer a day to categorize, because there was no stable vocabulary of "operations" to report on — only raw query text.

The team's fix mirrors this chapter's argument exactly: they inventoried the query *shapes* that had actually been asked for (grouped by structure, not literal values) and replaced `execute_sql` with `get_vehicle_entries`, `get_passenger_footfall`, and `get_daily_sales` — each backed by a parameterized query, a service function with its own unit tests, and a connection pool owned by the server process. `execute_sql` was kept, but moved to a separate, human-gated internal tool used only by the data engineering team through the MCP Inspector, never connected to the facility-manager-facing agent. The site-isolation bug in problem (2) became structurally impossible, because every domain-specific tool required a `gate_id`/`site_id` argument the model could not omit; and the "what has this agent been doing" question in problem (3) became a one-line log query grouped by tool name.

## Best Practices

- **Default to domain-specific tools.** Design them around the business questions your agents actually need answered (Chapter 10's schema-design lens), not around exposing your query language.
- **Always parameterize.** Every database driver worth using supports it (`asyncpg`'s `$1`/`$2`, `psycopg`'s `%s`, MongoDB's dict-shaped filters/pipelines) — there is never a legitimate reason to string-format a value into a query, regardless of how "internal" or "trusted" the caller seems.
- **Keep the services layer free of MCP imports**, per Chapter 7. If a service function can't be unit-tested without spinning up an MCP client, the separation has leaked.
- **Create exactly one connection pool per server process, at startup**, and reuse it across every tool call — never open a fresh connection inside a tool function.
- **Size your pool deliberately** (`min_size`/`max_size`, a sane `command_timeout`) rather than accepting driver defaults blindly; this is capacity planning, not an afterthought (Chapter 20 goes deeper).
- **Use a least-privilege database role for every tool**, ideally scoped to specific views/collections/indices rather than the full schema — defense in depth alongside, not instead of, domain-specific tool design (Chapter 14).
- **If you keep a generic query tool at all, gate it separately** — its own server or namespace, human-in-the-loop, read-only role, verbatim logging — and never expose it to the same context that processes untrusted input.
- **Log at the domain-tool granularity** (tool name + typed arguments), not just raw query text, so auditing and rate-limiting are meaningful without post-hoc SQL parsing.

## Common Mistakes

- **Reaching for `execute_sql` "because we can't predict every question."** That uncertainty is a reason to talk to the people using the agent about what they actually need, not a reason to hand the model a live query console — see Section 4.
- **String-formatting values into a query "just this once."** There is no safe version of this; use parameterized queries and mocked/test data to verify behavior instead of skipping parameterization for convenience.
- **Opening a new database connection inside a tool function.** This pays handshake/auth cost on every call and has no concurrency ceiling — always use a pool owned by the server process (Section 8).
- **Putting query logic directly inside `@mcp.tool()` functions**, skipping the services layer. This makes the business logic untestable without an MCP client and re-couples protocol concerns to database concerns, which Chapter 7 specifically argues against.
- **Assuming a "read-only" docstring is an actual control.** It's a hint to the model, not an enforcement mechanism — pair domain-specific tools with a genuinely read-only database role wherever an execute-style capability exists at all.
- **Treating MongoDB's `$where`/JavaScript-execution stages, or Elasticsearch's raw Query DSL passthrough, as somehow safer than raw SQL** because they "aren't SQL." The injection reasoning is backend-agnostic: any tool that lets the model supply query *structure* rather than query *data* has the same problem, regardless of database technology.
- **Sizing a connection pool without thinking about concurrent agent sessions.** A pool tuned for one interactive user starves under ten concurrent agent sessions each issuing tool calls; size for your actual expected concurrency, and revisit under load (Chapter 20).

## Summary

- Database-backed MCP tools come in two dominant shapes: a **generic `execute_sql`-style tool** (flexible, but the entire SQL grammar becomes the injection surface, with no meaningful way to enforce least privilege, rate limits, or audit granularity) and **domain-specific tools** (`get_daily_sales`, `get_vehicle_entries`, `get_passenger_footfall`) that expose a small, typed set of parameters for one business question at a time.
- Production systems should default to domain-specific tools, tying directly back to Chapter 10's schema-design guidance and Chapter 14's least-privilege and injection-defense principles; a generic query tool, if kept at all, belongs behind a separate, human-gated, read-only-role, break-glass boundary — never wired into the same agent context that handles untrusted input.
- Parameterized queries (`$1`/`$2` placeholders, dict-shaped Mongo filters) are non-negotiable regardless of pattern — never build a query by string-formatting a value, whether that value came from a config file or a model's tool call.
- A real analytics MCP server applies Chapter 7's tool → service → data-access separation: thin `@mcp.tool()` functions delegate to plain `async def` service functions that hold the query logic, independently testable without any MCP client involved.
- A long-lived MCP server process should own exactly one connection pool, created once at startup and reused across every tool call, rather than opening a fresh connection per call — this avoids both unnecessary handshake latency and an unbounded concurrency ceiling against the database (foreshadowing Chapter 20's production-sizing discussion).
- The same pattern generalizes cleanly: MongoDB tools fix an aggregation pipeline's *shape* and vary only leaf values; Elasticsearch tools fix a Query DSL template the same way. The technology changes; the "fixed structure, typed data in" principle does not.

## Knowledge Check

1. List three concrete reasons a generic `execute_sql` tool is riskier in production than a small set of domain-specific tools.
2. Why does a "read-only" docstring on an `execute_sql` tool not actually guarantee read-only behavior?
3. What specifically makes a parameterized query safe against injection, compared to a string-formatted one, even when the string-formatted value looks harmless?
4. In the worked PostgreSQL example, why do the functions in `services/sales.py` and `services/vehicles.py` have no MCP imports? What does that buy you?
5. Why should a long-lived MCP server create its connection pool once at startup instead of opening a new connection inside each tool function?
6. How does the domain-specific pattern extend to MongoDB, given that MongoDB doesn't use SQL at all?
7. If your team genuinely needs an ad-hoc `execute_sql`-style capability, what concrete controls should surround it before it's connected to any agent?

<details>
<summary>Answers</summary>

1. Any three of: the entire SQL grammar becomes the injection surface since the model composes free-form query text; least privilege can't be expressed at the tool boundary (the DB credential becomes the only control); "read-only" is unenforceable by docstring alone; rate limiting and auditing are meaningless at the level of "ran a query" versus "ran a specific, bounded business operation"; the model has to correctly generate an entire query language from scratch on every call instead of populating a few typed fields.
2. A docstring is a hint to the model, not an enforcement mechanism — nothing stops the model from emitting a `DELETE`/`UPDATE`/`DROP` statement through the same `execute_sql(query: str)` parameter. Actual read-only behavior requires a real control: a database role restricted to `SELECT`, or a query-parsing allowlist, not a comment in the tool description.
3. A parameterized query sends the query text and the parameter values to the database separately; the database's query planner treats parameters strictly as data values, never as SQL syntax to parse, regardless of what characters they contain. A string-formatted value is spliced directly into the query text before the database ever sees it, so a value like `x' OR '1'='1` (or a MongoDB `$where` clause with embedded JavaScript) can change the query's actual structure — "looks harmless" is irrelevant, since the risk is in how the value is incorporated, not in its apparent content.
4. Because they contain only business logic and database queries — no `FastMCP`, no tool decorators, no protocol concerns. That buys independent unit-testability (you can verify `get_daily_sales`'s filtering logic with a normal async test and a test database, with no MCP client or model involved) and reusability (the same service function could back a REST endpoint or a scheduled job, not just an MCP tool), per Chapter 7's separation of protocol from business logic.
5. A long-lived server serves many tool calls, often from multiple concurrent agent sessions, over its entire uptime. Opening a fresh connection per call pays TCP/TLS/auth handshake cost every single time and has no concurrency ceiling, which can exhaust the database's connection limit under load and affect every other client of that database. A pool created once at startup amortizes that setup cost and gives you an explicit, tunable ceiling (`max_size`) on how much concurrent load this server can place on the database.
6. The query mechanism changes from parameterized SQL to a fixed aggregation pipeline (`$match` → `$group` → `$sort`, etc.), but the principle is identical: the pipeline's stage *structure* is fixed by a reviewed service function, and only typed leaf values (an ID, a parsed datetime, an optional filter) vary based on the tool's arguments. A generic tool that accepted an arbitrary pipeline or a `$where`-style JavaScript stage from the model would reintroduce the exact same injection and cost-control problems as generic SQL.
7. Isolate it to a separate, explicitly-scoped server or tool namespace never connected to agents that also process untrusted input; back it with a read-only database role scoped as narrowly as possible; require a human in the loop (either issuing the query directly via the MCP Inspector, or approving an agent-proposed query before execution); log every query verbatim tied to the invoking identity; and alert on any attempted write-shaped statement even against a supposedly read-only role.

</details>

## Hands-On Exercise

Using the worked PostgreSQL example in Section 7 as a starting point (you don't need a real production database — a local Postgres instance with a small `sales` and `vehicle_entries` table, or even an in-memory SQLite stand-in for the exercise, is enough to practice the pattern):

1. **Inventory query shapes.** Write down three to five example "questions" an internal ops agent might ask of a database you're familiar with (real or hypothetical). For each, identify the *shape* of the query it implies (which columns, which filters, which aggregation) separate from the literal values.
2. **Design domain-specific tools.** For each distinct shape, design one tool signature — name, typed parameters, one-sentence docstring — following Chapter 10's guidance on narrow, well-typed schemas. Do not design a tool per literal value; design one tool per *shape*.
3. **Implement the services layer.** Write the corresponding `async def` service functions with parameterized queries (no MCP imports), against your test database. Verify each with a plain `pytest`/`pytest-asyncio` test, independent of any MCP client.
4. **Wire up the tool layer.** Add a `lifespan` hook that creates one connection pool at server startup and closes it at shutdown, and thin `@mcp.tool()` functions that each delegate to exactly one service function.
5. **Break it on purpose, then fix it.** Temporarily rewrite one service function to string-format a parameter into the query instead of using a placeholder. Pass it a value like `' OR '1'='1` and observe the difference in returned rows compared to the parameterized version. Revert the change and confirm the parameterized version is unaffected by the same input.
6. **Try the MCP Inspector** (Chapter 12) against your finished server: call each domain-specific tool with valid and invalid arguments, and confirm the schema constraints (typed dates, optional filters) actually stop malformed calls before they reach your service layer.

## Further Reading

- `asyncpg` documentation — connection pools (`asyncpg.create_pool`), parameterized queries, and `command_timeout`: `magicstack.github.io/asyncpg`
- `psycopg` (v3) documentation — parameterized queries and async connection pooling: `psycopg.org/psycopg3`
- PostgreSQL documentation on roles and privileges — for scoping a genuinely read-only database role: `postgresql.org/docs/current/user-manag.html`
- MongoDB aggregation pipeline reference — for building fixed-shape, parameterized-by-leaf-value pipelines: `www.mongodb.com/docs/manual/aggregation`
- OWASP SQL Injection Prevention Cheat Sheet — the general-purpose version of Section 6's argument, independent of MCP: `cheatsheetseries.owasp.org`
- Chapter 7 of this course (Building MCP Servers), for the full treatment of the tool/service/data-access separation applied here to a database backend
- Chapter 10 of this course (Tool Schema Design), for the general principles behind why narrow, typed schemas beat open-ended ones
- Chapter 14 of this course (MCP Security), for the least-privilege and injection-defense reasoning this chapter applies specifically to databases
- Chapter 20 of this course (Production MCP Architecture), for connection-pool sizing, retries, and observability under real production load

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./14-mcp-security.md">← Previous: MCP Security</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./16-mcp-and-rest-apis.md">Next: MCP + REST APIs →</a>
</div>
