# MCP Integration

## Learning Objectives

By the end of this chapter, you will be able to:

- State precisely why `create_deep_agent()` has no `mcp_servers=` parameter, and why that absence is a *feature*
  of the design rather than a gap
- Wire tools from one or more MCP servers into a deep agent using `langchain-mcp-adapters`' `MultiServerMCPClient`,
  with nothing more than the standard `tools=` argument you already know from Chapter 3
- Distinguish the SDK-level `tools=` pattern from the `deepagents-cli`'s `mcp-servers add`/`tools.json`
  deployment convenience, and explain why conflating the two leads to real confusion
- Assign MCP-derived tools to specific subagents so a coordinator's own context stays uncluttered (tying
  directly into Chapter 8)
- Gate write-capable MCP tools — a GitHub tool that opens a PR, a Slack tool that posts a message — behind
  `interrupt_on` or `permissions`, exactly as you would any other tool (tying directly into Chapter 9)
- Build an end-to-end "Analytics Agent" that combines MongoDB MCP tools, SQL MCP tools, and the built-in
  filesystem tools to answer questions and produce a Markdown dashboard report

## Prerequisites for This Chapter

- **Chapter 1** (Introduction & Prerequisites) — package layout, and the general "DeepAgents is middleware over
  `create_agent`, not a new runtime" framing
- **Chapter 2** (Architecture & Internals) — how `create_deep_agent()` assembles its middleware stack from the
  keyword arguments you pass it; this chapter leans on the fact that `tools=` is handled identically no matter
  where a tool came from
- **Chapter 3** (Your First Deep Agent) — the `create_deep_agent(model, tools=..., system_prompt=..., ...)` call
  shape this chapter extends
- **Chapter 5** (Filesystem-Backed Context) — the write-intermediate-results-to-a-file pattern the Analytics
  Agent project reuses directly
- **Chapter 7** (Memory & Persistence) — the SDK-vs-CLI distinction established there for memory is mirrored
  here for MCP; if you haven't read that chapter, at least note that this course consistently separates
  "what the `deepagents` Python package's functions do" from "what the `deepagents-cli` product's deployment
  tooling does"
- **Chapter 8** (Subagent Orchestration) — the `SubAgent` dict shape this chapter reuses to route MCP tools to
  a specific subagent
- **Chapter 9** (Human-in-the-Loop) — `interrupt_on` and `permissions`, which this chapter applies to MCP tools
  with zero new mechanism
- **Your own MCP server/client experience** — this course assumes you have built MCP servers and wired MCP
  clients into agents before. Nothing about MCP itself — transports, sessions, tool discovery, resources,
  prompts — is re-explained here. This chapter is scoped entirely to *where the resulting tools plug into
  `deepagents`*.

---

## The One Correction This Chapter Exists to Make

If you searched the `create_deep_agent()` signature from Chapter 2 for an `mcp_servers=` parameter, a
`with_mcp(...)` builder method, or an `McpMiddleware` class, you won't find one. Look again at the full
signature:

```python
def create_deep_agent(
    model: str | BaseChatModel | None = None,
    tools: Sequence[BaseTool | Callable | dict[str, Any]] | None = None,
    *,
    system_prompt: str | SystemMessage | None = None,
    middleware: Sequence[AgentMiddleware[StateT_co, ContextT]] = (),
    subagents: Sequence[SubAgent | CompiledSubAgent | AsyncSubAgent] | None = None,
    skills: list[str] | None = None,
    memory: list[str] | None = None,
    permissions: list[FilesystemPermission] | None = None,
    backend: BackendProtocol | None = None,
    interrupt_on: dict[str, bool | InterruptOnConfig] | None = None,
    response_format: ResponseFormat[ResponseT] | type[ResponseT] | dict[str, Any] | None = None,
    state_schema: type[DeepAgentState] | None = None,
    context_schema: type[ContextT] | None = None,
    checkpointer: Checkpointer | None = None,
    store: BaseStore | None = None,
    debug: bool = False,
    name: str | None = None,
    cache: BaseCache | None = None,
) -> CompiledStateGraph[...]
```

That's the complete parameter list. There is no `mcp_servers=`, no `mcp_config=`, nothing MCP-shaped at all.
This is a common wrong assumption — enough tutorials frame DeepAgents as a batteries-included agent product that
it's reasonable to expect a first-class MCP feature to exist somewhere in the constructor. It doesn't, and it
isn't supposed to.

Here's the good news buried in that correction, and it's worth stating plainly because it's easy to read "no
MCP feature" as a limitation: **for someone who has already built MCP servers and wired MCP clients into other
LangChain/LangGraph agents, there is nothing new to learn about MCP here.** The `deepagents` package's own
`README.md` states its tool story as a single line — "Tools — bring your own functions or any MCP server" —
and that line is the entire MCP integration surface. MCP tools are, from `create_deep_agent()`'s point of view,
indistinguishable from a hand-written `@tool` function. Both satisfy `BaseTool`. Both go in the same `tools=`
list. `create_deep_agent()` never asks how a tool was produced.

That means the actual work in this chapter is not "how do I teach DeepAgents about MCP" — there is no such
teaching to do — but "where do I put the tool list I already know how to produce." That's a one-line answer:
**produce the list with `langchain-mcp-adapters`, same as you would for any other LangChain/LangGraph agent, and
pass it into `tools=`.**

## The Standard Wiring Pattern

`langchain-mcp-adapters` gives you `MultiServerMCPClient`, configured with one entry per MCP server, and a
`get_tools()` (or async `aget_tools()`) method that returns a flat list of `BaseTool` instances — one per tool
exposed by any of the configured servers, already bound to a session/transport so calling the tool round-trips
to the right server. This is the same client shape you'd use to wire MCP tools into a plain `create_agent()`
call or a hand-built LangGraph node; nothing about it is deepagents-specific.

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from deepagents import create_deep_agent

mcp_client = MultiServerMCPClient(
    {
        "mongo": {
            "transport": "stdio",
            "command": "npx",
            "args": ["-y", "mongodb-mcp-server", "--connectionString", MONGO_URI],
        },
        "sql": {
            "transport": "stdio",
            "command": "uvx",
            "args": ["mcp-server-sql", "--dsn", ANALYTICS_DB_DSN],
        },
        "slack": {
            "transport": "streamable_http",
            "url": "https://mcp.internal.example.com/slack/mcp",
            "headers": {"Authorization": f"Bearer {SLACK_MCP_TOKEN}"},
        },
        "github": {
            "transport": "streamable_http",
            "url": "https://mcp.internal.example.com/github/mcp",
            "headers": {"Authorization": f"Bearer {GITHUB_MCP_TOKEN}"},
        },
    }
)

mcp_tools = await mcp_client.get_tools()

agent = create_deep_agent(
    model=model,
    tools=mcp_tools + [render_chart],  # ordinary Python tools mixed in freely
    system_prompt="You are an enterprise analytics assistant...",
)
```

Nothing here is a deepagents API — everything from `MultiServerMCPClient(...)` down through `await
mcp_client.get_tools()` is straight `langchain-mcp-adapters` usage exactly as you'd already write it. The
*only* deepagents-specific line is the last one, and it's the same `tools=` argument from Chapter 3. Per the
ground truth for this course, the exact shape of `MultiServerMCPClient`'s config dict and the `get_tools()` /
`aget_tools()` split is standard `langchain-mcp-adapters` API surface — this chapter relies on your own MCP
client experience for that detail rather than re-deriving it from the deepagents fact sheet, which only confirms
the destination (`tools=`), not the adapter's internals.

A subtlety worth naming: because `get_tools()` is async, and `create_deep_agent()` itself is synchronous
(it just builds a middleware list and compiles a graph — Chapter 2), you fetch the tool list *before* calling
`create_deep_agent()`, typically once at process startup inside your FastAPI app's lifespan/startup hook, not
per-request. The agent object itself doesn't hold a live MCP session; the individual `BaseTool` instances do,
and how long those sessions stay valid is governed by `langchain-mcp-adapters` and your transport choice, not by
anything in `deepagents`.

### Wiring the fetch into a FastAPI lifespan

Since you're building this behind FastAPI, the natural place for `mcp_client.get_tools()` to run is the same
lifespan hook you'd already use to open a database pool or an HTTP client session — MCP sessions are just
another resource with a startup/shutdown lifecycle, and `deepagents` has no opinion about how you manage that:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    mcp_tools = await mcp_client.get_tools()
    app.state.analytics_agent = create_deep_agent(
        model=model,
        tools=[],
        subagents=[{**database_subagent, "tools": mcp_tools}],
        system_prompt=coordinator_system_prompt,
    )
    yield
    # langchain-mcp-adapters session teardown, if your transport requires it,
    # belongs here — again, ordinary resource-lifecycle code, not a deepagents concern.

app = FastAPI(lifespan=lifespan)
```

Building the agent once at startup and reusing it across requests is exactly the pattern you already follow for
a plain `create_agent()`/`StateGraph` service — a `CompiledStateGraph` is safe to share across concurrent
`.ainvoke()`/`.astream()` calls as long as each call carries its own `thread_id` (Chapter 10). The MCP tools
attached to it don't change that.

### Tool Name Collisions Across Multiple Servers

With four servers in one `MultiServerMCPClient` config, it's worth checking for a failure mode that has nothing
to do with deepagents but will absolutely surface once you wire real servers together: two servers exposing a
tool with the same name (a Mongo server and a generic "database" server both naming a tool `query`, for
instance). `MultiServerMCPClient` namespaces tools by server internally, but the flat list `get_tools()` returns
uses each tool's own declared name — if two servers declare the identical tool name, you can end up with two
`BaseTool` objects with the same `.name` in one `tools=` list, and only one of them will be addressable by name
once it reaches a model's tool-calling surface (and, more concretely for this chapter, once it's the key you use
for `interrupt_on={"...": True}` or a subagent's `tools=` filter). This is standard `langchain-mcp-adapters`/
tool-calling behavior, not a deepagents wrinkle, but it bites hardest exactly in the kind of multi-server setup
this chapter builds — check `[t.name for t in mcp_tools]` for duplicates before you rely on name-based filtering
or gating anywhere downstream.

## Diagram: From MCP Servers to a Deep Agent's Tool List

```mermaid
flowchart TD
    S1["Mongo MCP server<br/>(stdio)"] --> C["MultiServerMCPClient(...)"]
    S2["SQL MCP server<br/>(stdio)"] --> C
    S3["Slack MCP server<br/>(streamable_http)"] --> C
    S4["GitHub MCP server<br/>(streamable_http)"] --> C

    C --> G["await mcp_client.get_tools()<br/>returns flat list[BaseTool]"]
    G --> M["mcp_tools = [mongo_find, mongo_aggregate,<br/>sql_query, slack_post_message,<br/>github_open_pr, ...]"]

    M --> COMBINE["combined tools list =<br/>mcp_tools + [your own @tool functions]"]
    COMBINE --> CDA["create_deep_agent(model, tools=combined)"]

    CDA --> COORD["Coordinator agent<br/>(sees every tool by default)"]
    CDA -.->|"subagents=[{'tools': [mongo_find, sql_query], ...}]"| SUB["'database' subagent<br/>(Mongo + SQL tools only)"]

    style C fill:#e1f5ff
    style CDA fill:#fff3e0
```

There is no box in this diagram for "deepagents MCP layer" — because there isn't one. Every arrow into
`create_deep_agent(...)` is a plain `list[BaseTool]`, assembled entirely by `langchain-mcp-adapters` and your own
code, before deepagents ever sees it.

## The CLI's `mcp-servers add` / `tools.json` — A Different Layer Entirely

Separately from anything above, the `deepagents-cli`/`deepagents init` **deployment tooling** — the CLI product
built around the `deepagents-code` conventions, not the `create_deep_agent()` SDK function — has its own MCP
convenience: a `deepagents mcp-servers add` command and a `tools.json` configuration file that registers MCP
servers by name so a deployed CLI agent can discover and use them without a developer hand-writing a
`MultiServerMCPClient` config.

This is the exact same shape of distinction Chapter 7 drew for memory (`MemoryMiddleware` in the SDK vs. the
`deepagents-code` CLI's `AGENTS.md` convention), and it's worth being just as explicit about it here:

| | SDK (`create_deep_agent()`) | CLI (`deepagents-cli` / `deepagents init`) |
|---|---|---|
| What it configures | The `tools=` argument to a Python function call | A deployed CLI agent's runtime tool discovery |
| Where MCP servers are declared | Wherever you construct `MultiServerMCPClient({...})` — your own code | `tools.json`, edited via `deepagents mcp-servers add` |
| What you get back | A `list[BaseTool]` you control the shape of | Tools auto-discovered by the CLI product at agent runtime |
| Who this is for | You, writing a `create_deep_agent(tools=...)` call in a Python service | Teams deploying agents through the `deepagents-cli` product without writing an adapter client by hand |

If you're building a Python service that calls `create_deep_agent()` directly — which is the entire premise of
this course and almost certainly your actual production path given your FastAPI/Bedrock background — `tools.json`
and `mcp-servers add` are not something you need to touch. They solve a different problem (zero-code MCP
registration for a CLI-deployed agent), not the one this chapter is about (getting MCP tools into a `tools=`
list you're constructing in your own code). Don't go looking for a `tools.json` your `create_deep_agent()` call
is supposed to read — it doesn't read one; there's no wiring between the two layers.

## Routing MCP Tools to Specific Subagents

Because MCP tools are just entries in a `list[BaseTool]`, everything Chapter 8 taught you about subagent tool
scoping applies to them with zero new mechanism. A `SubAgent` dict's own `tools=` key can be a subset of the
combined list, which means you can hand a `"database"` subagent exactly the Mongo and SQL tools, and nothing
else — keeping the coordinator's own context free of tool definitions it never needs to reason about directly.

```python
mongo_tools = [t for t in mcp_tools if t.name.startswith("mongo_")]
sql_tools = [t for t in mcp_tools if t.name.startswith("sql_")]
collab_tools = [t for t in mcp_tools if t.name.startswith(("slack_", "github_"))]

database_subagent = {
    "name": "database",
    "description": "Runs MongoDB and SQL queries against the analytics warehouse. "
                    "Delegate any question requiring raw data retrieval to this subagent.",
    "system_prompt": "You are a data-retrieval specialist. Given a question, decide "
                      "whether Mongo or SQL is the right source, run the minimal query "
                      "needed, and write results to /data/ as JSON for the coordinator "
                      "to read back.",
    "tools": mongo_tools + sql_tools,
}

collab_subagent = {
    "name": "collaboration",
    "description": "Posts updates to Slack and opens GitHub PRs on request.",
    "system_prompt": "You post finished, approved content to Slack or GitHub. "
                      "You do not draft content yourself.",
    "tools": collab_tools,
}

coordinator = create_deep_agent(
    model=model,
    tools=[],  # coordinator itself holds no MCP tools directly
    subagents=[database_subagent, collab_subagent],
    system_prompt="Delegate data questions to the 'database' subagent and "
                  "publishing actions to the 'collaboration' subagent.",
)
```

The payoff is the same one Chapter 8 argued for generally, just concretely instantiated with MCP tools: the
coordinator's own system prompt and tool-calling surface stay small and legible, while the `"database"` subagent
gets a system prompt written *for* Mongo/SQL usage specifically, without every other tool's schema competing for
the model's attention on every turn.

## Gating Write-Capable MCP Tools with `interrupt_on` / `permissions`

MCP tools are named tools like any other — `mongo_aggregate`, `sql_query`, `slack_post_message`,
`github_open_pr`, whatever the server's tool schema calls them. That means Chapter 9's `interrupt_on` dict and
`permissions=[...]` list gate them exactly like a filesystem tool, keyed by the same tool name
`langchain-mcp-adapters` assigned when it built the `BaseTool`.

This matters most for MCP tools that perform a side effect outside your own system — a Slack tool that posts a
message a human hasn't reviewed, a GitHub tool that opens a pull request against a real repository. Read-only
tools (`mongo_find`, `sql_query` against a reporting replica) are reasonable to leave ungated; write-capable
ones generally are not.

```python
agent = create_deep_agent(
    model=model,
    tools=mcp_tools,
    subagents=[database_subagent, collab_subagent],
    interrupt_on={
        "slack_post_message": True,
        "github_open_pr": True,
        # everything else (mongo_find, sql_query, ...) is not listed -> no gate
    },
)
```

A useful discipline when a multi-server MCP setup grows past a handful of tools: classify every MCP tool
name into "read" or "write/side-effecting" up front, and let that classification drive the `interrupt_on` dict
mechanically rather than deciding gate-by-gate as an afterthought.

| MCP tool (example name) | Server | Effect | Gate with `interrupt_on`? |
|---|---|---|---|
| `mongo_find` | Mongo | Read-only query | No — safe to leave ungated |
| `sql_query` | SQL | Read-only query against a reporting replica | No |
| `slack_post_message` | Slack | Posts a message a human hasn't reviewed | Yes — `True` |
| `slack_list_channels` | Slack | Read-only lookup | No |
| `github_open_pr` | GitHub | Opens a real PR against a tracked repo | Yes — `True` |
| `github_get_file` | GitHub | Read-only file fetch | No |

This is exactly the same read/write judgment call Chapter 9 asked you to make for filesystem tools via
`FilesystemPermission` — `delete`/`write_file`/`edit_file` gated, `read_file`/`ls`/`glob`/`grep` left alone. MCP
tools don't introduce a new axis of risk assessment; they're just more tool names on the same table.

Because this is the identical `interrupt_on` mechanism from Chapter 9, the resume flow is identical too: the
graph pauses via `interrupt()` before executing `slack_post_message` or `github_open_pr`, your application
surfaces the pending tool call (arguments included) to a human, and you resume with `Command(resume=...)` to
approve, edit the arguments, reject, or inject a substitute response — exactly as you would for any
`FilesystemPermission`-gated `write_file` call. There is no MCP-specific approval UI or payload shape to learn;
the tool call that reaches the interrupt is a standard LangChain `ToolCall`, whether it originated from an
MCP server or a function you wrote yourself.

## Project: The Analytics Agent

This section builds the multi-system agent this chapter has been assembling pieces for: a deep agent that
answers questions against a MongoDB event store and a SQL analytics warehouse, writes intermediate query
results to files (the Chapter 5 pattern — don't let raw JSON dumps sit in message history), and synthesizes a
final Markdown dashboard report.

### Requirements, translated into wiring decisions

1. **Data access** — MongoDB MCP tools for event-level lookups, SQL MCP tools for aggregate/warehouse queries.
   Both come from `MultiServerMCPClient`, not from anything deepagents-specific.
2. **Context discipline** — query results can be large. Per Chapter 5, the agent should `write_file` raw results
   to `/data/` and read them back in slices rather than keeping full JSON blobs in the conversation.
3. **Isolation** — the `"database"` subagent (Section 5 above) owns the Mongo/SQL tools so the coordinator's
   context isn't cluttered with query-tool schemas it never calls directly.
4. **Safety** — this particular agent doesn't post anywhere or open PRs, so no `interrupt_on` gating is required
   for its own tools; the moment you add the Slack/GitHub tools from the Hands-On Exercise below, that changes.
5. **Deliverable** — a Markdown report written to `/reports/dashboard.md`, built from the filesystem-backed
   working notes, not hallucinated from memory of the query results.

### Wiring code

```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from deepagents import create_deep_agent

MONGO_URI = "mongodb://analytics-readonly:***@mongo.internal:27017/events"
ANALYTICS_DB_DSN = "postgresql://analytics_ro:***@warehouse.internal:5432/analytics"

mcp_client = MultiServerMCPClient(
    {
        "mongo": {
            "transport": "stdio",
            "command": "npx",
            "args": ["-y", "mongodb-mcp-server", "--connectionString", MONGO_URI],
        },
        "sql": {
            "transport": "stdio",
            "command": "uvx",
            "args": ["mcp-server-sql", "--dsn", ANALYTICS_DB_DSN],
        },
    }
)


async def build_analytics_agent(model):
    mcp_tools = await mcp_client.get_tools()
    mongo_tools = [t for t in mcp_tools if t.name.startswith("mongo_")]
    sql_tools = [t for t in mcp_tools if t.name.startswith("sql_")]

    database_subagent = {
        "name": "database",
        "description": (
            "Runs MongoDB and SQL queries against the events store and analytics "
            "warehouse. Delegate any question requiring raw data retrieval here. "
            "Always writes raw results to /data/ before summarizing them."
        ),
        "system_prompt": (
            "You are a data-retrieval specialist for an airport/enterprise analytics "
            "platform. Given a request:\n"
            "1. Decide whether MongoDB (per-event/per-flight granularity) or SQL "
            "(pre-aggregated warehouse tables) is the right source.\n"
            "2. Run the minimal query needed to answer the question.\n"
            "3. write_file the raw JSON/rows to /data/<short-name>.json — never "
            "reason over large raw output directly in your response.\n"
            "4. read_file back only the slice you need, then return a short "
            "structured summary (not the raw payload) to the coordinator."
        ),
        "tools": mongo_tools + sql_tools,
    }

    coordinator_system_prompt = """You are an enterprise analytics coordinator.

    For any question requiring data, delegate to the 'database' subagent — do not
    call Mongo/SQL tools yourself; you don't have them.

    Workflow for a dashboard report request:
    1. Break the request into concrete data questions (e.g. "flight delays by
       terminal, last 7 days", "passenger throughput by hour").
    2. Delegate each to the 'database' subagent; it will write findings and
       return a summary.
    3. write_file each summary into /reports/sections/<topic>.md as you receive it.
    4. Once all sections are gathered, read them back and write_file the final
       synthesis to /reports/dashboard.md: an executive summary followed by one
       section per topic, each with a short narrative and a markdown table.
    5. Reply to the user with a short pointer to /reports/dashboard.md, not a
       copy of its full contents."""

    return create_deep_agent(
        model=model,
        tools=[],  # coordinator delegates; the database subagent holds the MCP tools
        subagents=[database_subagent],
        system_prompt=coordinator_system_prompt,
    )


async def main():
    from your_app.models import model  # your configured Bedrock/Anthropic chat model

    agent = await build_analytics_agent(model)
    result = await agent.ainvoke(
        {
            "messages": [
                {
                    "role": "user",
                    "content": (
                        "Build today's operations dashboard: flight delays by "
                        "terminal for the last 7 days, and passenger throughput "
                        "by hour for yesterday."
                    ),
                }
            ]
        }
    )
    print(result["messages"][-1].content)


if __name__ == "__main__":
    asyncio.run(main())
```

### Resulting filesystem layout (via whichever backend is configured — Chapter 6)

```
/data/
├── flight_delays_by_terminal.json     # raw SQL/Mongo output, written by the database subagent
└── passenger_throughput_by_hour.json
/reports/
├── sections/
│   ├── flight-delays.md               # curated per-topic summary
│   └── passenger-throughput.md
└── dashboard.md                       # final synthesized report — the deliverable
```

This is the same three-layer pattern from Chapter 5 (raw dump, working notes, final artifact), just now fed by
MCP tool output instead of a hand-written `paper_search` tool. Nothing about the pattern changed because the
data source changed — that's the entire point this chapter has been making: MCP tools are just tools.

## Real-World Scenario

You're extending the Analytics Agent for an internal ops team that also wants it to post a daily summary to a
Slack channel and, when it detects a recurring issue (say, the same delay cause three days running), open a
GitHub issue against the ops-tracking repo. Both are genuinely useful automations and both are exactly the kind
of MCP tool this chapter warned you to gate:

1. Add a `slack` and a `github` server to the same `MultiServerMCPClient` config used above.
2. Filter their tools into a `collab_subagent["tools"]` list, following the same pattern as
   `database_subagent`.
3. Because `slack_post_message` and `github_open_issue` perform real side effects a human hasn't reviewed yet,
   gate both with `interrupt_on={"slack_post_message": True, "github_open_issue": True}` on the top-level
   `create_deep_agent()` call — `interrupt_on` gates by tool name regardless of which subagent calls it, so this
   one dict covers the tool no matter where in the hierarchy it's invoked.
4. Wire a checkpointer (Chapter 10) so the interrupted run survives your API process restarting between "agent
   proposed a Slack message" and "a human on the ops team approved it the next morning."

Nothing in this extension required a new deepagents concept — it's the same `MultiServerMCPClient` pattern, the
same `SubAgent["tools"]` scoping, and the same `interrupt_on` dict this chapter already covered, applied to two
more MCP servers.

## Best Practices

- **Fetch MCP tools once, at startup, not per-request.** `get_tools()`/`aget_tools()` establishes sessions;
  treat the resulting `list[BaseTool]` as part of your application's startup wiring (a FastAPI lifespan hook),
  not something you reconstruct on every incoming request.
- **Filter by tool name into subagent-specific lists, not by hand-maintaining parallel configs.** `[t for t in
  mcp_tools if t.name.startswith("mongo_")]` keeps the subagent tool lists in sync with whatever the MCP server
  actually exposes, rather than a manually copy-pasted list that drifts.
- **Gate by tool name, not by server.** `interrupt_on` and `permissions` don't know or care which MCP server a
  tool came from — decide gating per tool based on whether it performs a write/side-effect, the same judgment
  you'd apply to any tool regardless of origin.
- **Keep the coordinator's own `tools=` list free of MCP tools it never calls directly.** If only a subagent
  needs the Mongo/SQL/Slack/GitHub tools, give them to the subagent's own `tools=`, not the top-level agent's —
  this is the Chapter 8 context-isolation argument, and MCP tool schemas are exactly the kind of bulk that
  benefits from it.
- **Never assume `tools.json`/`mcp-servers add` affects a `create_deep_agent()` call in your own code.** If
  you're writing Python that calls `create_deep_agent(tools=...)` directly, that CLI config file is not read by
  it — the two layers don't share wiring.

## Common Mistakes

- **Looking for an `mcp_servers=` parameter on `create_deep_agent()`.** It does not exist, on this version or
  any other documented in this course's ground truth. If you find yourself searching for one, the fix is not a
  different parameter name — it's recognizing that MCP tools belong in the ordinary `tools=` list, built with
  `MultiServerMCPClient(...).get_tools()`.
- **Conflating the CLI's `tools.json` with the SDK's `tools=`.** They solve different problems at different
  layers (`deepagents-cli` deployment-time tool discovery vs. a Python function argument you populate yourself).
  Editing `tools.json` will not add tools to a `create_deep_agent()` call anywhere in your service code, and
  vice versa.
- **Forgetting to gate write-capable MCP tools with HITL.** An MCP tool that opens a PR or posts to Slack is
  exactly as consequential as a `write_file` call to a shared filesystem — arguably more so, since the side
  effect lands outside anything your own checkpointer or backend controls. Not adding `interrupt_on` for these
  is the same class of mistake as shipping an ungated `execute` tool (Chapter 6, Chapter 19).
- **Constructing a new `MultiServerMCPClient` (and re-establishing sessions) on every request** instead of once
  at startup — a performance and connection-churn mistake, not a deepagents-specific one, but one this chapter's
  pattern makes easy to fall into if you inline the client construction inside a request handler.
- **Assuming subagent tool scoping requires special MCP-aware code.** It doesn't — a `SubAgent["tools"]` list is
  just a Python list; filtering it by `tool.name` prefix (or any other criterion) is ordinary list comprehension,
  not a deepagents or MCP feature.

## Summary

There is no first-class MCP feature in the `deepagents` SDK, and that absence is the whole story worth
internalizing: MCP tools are ordinary `BaseTool` objects once `MultiServerMCPClient(...).get_tools()` produces
them, and `create_deep_agent(tools=mcp_tools + [...])` is the entire integration point. The `deepagents-cli`'s
`mcp-servers add`/`tools.json` is a separate, CLI-deployment-level convenience that doesn't touch the SDK's
`tools=` argument at all. Because MCP tools are just named tools, everything you already know from Chapter 8
(scoping tools to a specific `SubAgent`) and Chapter 9 (`interrupt_on`/`permissions` gating) applies to them
without modification — the practical payoff for a multi-system integration like the Analytics Agent, where
Mongo/SQL tools go to a `"database"` subagent and any Slack/GitHub write-tools get an HITL gate before they can
act.

## Knowledge Check

1. Why is there no `mcp_servers=` parameter on `create_deep_agent()`, and what is the actual mechanism for
   giving a deep agent access to MCP tools?
2. What is the difference between the `deepagents-cli`'s `tools.json`/`mcp-servers add` and the pattern this
   chapter teaches with `MultiServerMCPClient(...).get_tools()`? Which one applies if you're calling
   `create_deep_agent()` directly in a FastAPI service?
3. You want a `"database"` subagent to have only the Mongo and SQL tools, not the Slack/GitHub ones, while the
   coordinator has none of them directly. How do you express that with `SubAgent["tools"]`?
4. A GitHub MCP tool named `github_open_pr` should require human approval before running. What single
   `create_deep_agent()` argument accomplishes this, and does it matter which subagent calls the tool?
5. Why is fetching MCP tools once at application startup preferable to calling `get_tools()` inside a request
   handler?
6. A colleague claims the Analytics Agent needs a custom deepagents MCP adapter to talk to two different MCP
   servers at once. Is that true? What actually handles "more than one MCP server"?

## Hands-On Exercise

Extend the Analytics Agent built in this chapter with a gated Slack tool:

1. Add a `"slack"` entry to the `MultiServerMCPClient` config (a real Slack MCP server, or a stub server exposing
   a single `slack_post_message` tool for practice purposes).
2. After `await mcp_client.get_tools()`, filter out the Slack tool(s) and add them to a `collab_subagent["tools"]`
   list (reuse the `collab_subagent` shape sketched in the Real-World Scenario section), separate from
   `database_subagent`.
3. Add `interrupt_on={"slack_post_message": True}` to the top-level `create_deep_agent()` call.
4. Invoke the agent with a request that requires posting a Slack summary. Confirm the run pauses via
   `interrupt()` before `slack_post_message` executes, exactly as Chapter 9 described for a `write_file`-style
   gated tool — inspect the pending tool call's arguments from the interrupt payload.
5. Resume the run with `Command(resume=...)` using an **approve** decision, and confirm the Slack tool actually
   executes afterward. Then repeat the exercise with a **reject** decision and confirm the tool never runs.
6. As a stretch, verify the gate applies regardless of *which* subagent invokes the tool — if your MCP server
   setup allows it, temporarily also expose the Slack tool to the coordinator directly and confirm the same
   `interrupt_on` entry still fires, since gating is keyed by tool name, not by which node in the graph called it.

## Further Reading

- [DeepAgents Overview (LangChain Docs)](https://docs.langchain.com/oss/python/deepagents/overview)
- [`langchain-ai/deepagents` GitHub repository](https://github.com/langchain-ai/deepagents) — read
  `libs/deepagents/README.md`'s feature list ("Tools — bring your own functions or any MCP server") and
  `libs/deepagents/deepagents/graph.py`'s `create_deep_agent()` signature directly; there is no MCP-specific
  module to find because there is no MCP-specific code in the SDK
- [`langchain-ai/langchain-mcp-adapters` GitHub repository](https://github.com/langchain-ai/langchain-mcp-adapters) —
  the actual source of `MultiServerMCPClient`, its config shape, and `get_tools()`/`aget_tools()`

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./10-checkpointing-and-durable-execution.md">← Previous: Checkpointing & Durable Execution</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./12-multi-agent-systems.md">Next: Multi-Agent Systems →</a>
</div>
