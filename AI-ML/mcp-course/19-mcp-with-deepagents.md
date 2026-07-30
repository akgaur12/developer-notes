# MCP + DeepAgents

Every prior chapter in this course answered "how does an agent talk to an MCP server." This chapter answers a narrower, more consequential question for anyone who has already built with `deepagents`: **where, exactly, does that MCP tool list plug into `create_deep_agent()`?** The honest answer is almost anticlimactic — there is no dedicated plug. MCP tools enter a deep agent through the exact same `tools=` parameter as a hand-written `@tool` function, and everything else you already know from the DeepAgents course — subagent scoping, `interrupt_on` approval gates, the built-in filesystem middleware — applies to them with zero new mechanism. This chapter is about making that "zero new mechanism" claim precise, and then using it to build something real: a repo-maintenance agent that triages GitHub issues by combining MCP-sourced GitHub tools with DeepAgents' planning and subagent features.

## Learning Objectives

By the end of this chapter, you will be able to:

- State precisely, and defend from the source signature, why `create_deep_agent()` has no `mcp_servers=`, `mcp=`, or `mcp_client=` parameter — and explain why that absence is a design choice, not a gap
- Build a `MultiServerMCPClient` spanning a GitHub MCP server and a database MCP server, `await` its `get_tools()`, and wire the combined list into `create_deep_agent(tools=...)` alongside ordinary Python tools
- Explain the async/sync boundary this creates — why `get_tools()` must resolve *before* the synchronous `create_deep_agent()` call, and where that fetch belongs in a FastAPI service (the lifespan hook, not per-request)
- Assign different MCP tool subsets to different subagents via the `SubAgent` `tools=` key, so a coordinator's own context isn't cluttered with tool schemas it never calls directly
- Gate destructive MCP-derived tools (opening a PR, running a migration) behind `interrupt_on`, and explain precisely why `permissions`/`FilesystemPermission` does **not** extend to arbitrary MCP tool names the way `interrupt_on` does
- Articulate the division of labor between DeepAgents' own built-in filesystem tools (`ls`/`read_file`/`write_file`/`edit_file`/`glob`/`grep`) and an MCP-provided filesystem or GitHub server — when each is the right tool for a given piece of state
- Build an end-to-end "Repo Maintenance DeepAgent" that triages GitHub issues using a GitHub MCP server's tools, a scoped triage subagent, and a human-approval gate before any write action

---

## Prerequisites

This chapter assumes you have completed the bulk of this course and, per this course's stated audience, **the companion [DeepAgents course](../deepagents-course/00-index.md) in this repository**. Specifically, you should already be comfortable with:

- **From this MCP course** — Chapter 4 (Tools: schemas, `tools/call`, results), Chapter 9 (Building MCP Clients), Chapter 13 (Authentication & Authorization, for the GitHub server's token handling), Chapter 14 (MCP Security, for the destructive-action gating this chapter leans on), Chapter 17 (MCP + LangChain: `langchain-mcp-adapters`, `MultiServerMCPClient`, `get_tools()`), and Chapter 18 (MCP + LangGraph: MCP tools as ordinary graph tool nodes)
- **From the DeepAgents course** — `create_deep_agent()`'s full parameter shape ([Chapter 2](../deepagents-course/02-architecture-and-internals.md)), the built-in filesystem tool surface ([Chapter 5](../deepagents-course/05-filesystem-backed-context.md)), the `SubAgent`/`CompiledSubAgent`/`AsyncSubAgent` shapes and context isolation via the `task` tool ([Chapter 8](../deepagents-course/08-subagent-orchestration.md)), and `interrupt_on`/`permissions`/`FilesystemPermission` as the human-in-the-loop mechanism ([Chapter 9](../deepagents-course/09-human-in-the-loop.md))

Nothing about MCP itself is re-taught here — no protocol mechanics, no transport details, no `ClientSession` internals. This chapter is scoped entirely to the seam between a tool list you already know how to produce (Chapter 17) and a `deepagents` constructor you already know how to call.

---

## 1. Where This Chapter Sits: DeepAgents on Top of MCP

Zoom out from any single tool call and the shape of a production deep agent looks like this:

```mermaid
flowchart TB
    subgraph DA["DeepAgent (create_deep_agent)"]
        direction TB
        PLAN["Planning<br/>(todo/plan middleware)"]
        MEM["Memory<br/>(cross-session recall)"]
        FS["Filesystem<br/>(ls / read_file / write_file / edit_file /<br/>glob / grep — scratch-space context)"]
        SUB["Subagents<br/>(task tool, context isolation)"]
        PLAN --- MEM --- FS --- SUB
    end

    DA -->|"tools= (ordinary list[BaseTool])"| MCP["langchain-mcp-adapters<br/>MultiServerMCPClient(...).get_tools()"]

    MCP --> S1["Database MCP server<br/>(SQL/NoSQL query tools)"]
    MCP --> S2["GitHub MCP server<br/>(issues, PRs, repo tools)"]
    MCP --> S3["Internal API MCP server<br/>(org-specific business logic)"]

    style DA fill:#e1f5ff
    style MCP fill:#fff3e0
```

Read this as a division of labor, not a layering of one framework on another:

- **DeepAgents supplies the reasoning and orchestration substrate** — a planning loop that decomposes a task into steps, a memory layer that survives across sessions, a scratch-space filesystem for keeping large intermediate state out of the message history, and subagents for delegating focused chunks of work with their own isolated context.
- **MCP supplies the standardized external capabilities** — the actual database queries, the actual GitHub API calls, the actual internal-API business logic, each behind a protocol that doesn't care whether the caller is `deepagents`, a bare LangGraph node, or an entirely different agent framework next year.

Nothing in the left-hand box (DeepAgents) knows or cares that the right-hand box (MCP) exists. That's the whole point of this chapter's next section.

---

## 2. The Corrected Misconception: There Is No `mcp_servers=` Parameter

> **Common wrong assumption, stated plainly so you can rule it out once:** because `deepagents` markets itself as a batteries-included agent framework — planning, memory, filesystem, subagents, human-in-the-loop all pre-wired — it's reasonable to *expect* MCP to be a sixth first-class feature, configured through something like `create_deep_agent(mcp_servers={...})` or an `McpMiddleware` class. **It is not.** Confirmed directly against the current source (`langchain-ai/deepagents`, `graph.py`), the full signature is:

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

Scan every keyword in that list. There is no `mcp_servers`, no `mcp_config`, no `mcp` of any spelling. This is not an oversight in an otherwise-complete API — it's the same design decision that keeps `create_deep_agent()` from needing a `rest_apis=` parameter or a `databases=` parameter. **`tools=` accepts `Sequence[BaseTool | Callable | dict[str, Any]]`, and an MCP-derived tool satisfies that type exactly as well as a function you decorated with `@tool` yourself.** `create_deep_agent()` never asks how a tool came to exist; it only asks whether it's a callable/`BaseTool`/dict shape it can bind to the model.

The practical consequence is the one sentence this entire chapter exists to establish: **you produce an MCP tool list with `langchain-mcp-adapters`, exactly as you would for a plain LangGraph agent (Chapter 18), and you hand it to `create_deep_agent()` through the ordinary `tools=` argument.** There is no DeepAgents-specific MCP API surface to learn beyond that sentence.

---

## 3. The Standard Wiring Pattern

`langchain-mcp-adapters`' `MultiServerMCPClient` (Chapter 17) takes a dict keyed by server name and returns a flat `list[BaseTool]` from its async `get_tools()` method — one `BaseTool` per tool exposed by any configured server, already bound to a transport so invoking it round-trips to the correct server.

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from deepagents import create_deep_agent

mcp_client = MultiServerMCPClient(
    {
        "github": {
            "transport": "stdio",
            "command": "github-mcp-server",
            "args": [],
            "env": {"GITHUB_PERSONAL_ACCESS_TOKEN": GITHUB_TOKEN},
        },
        "database": {
            "transport": "streamable_http",
            "url": "https://mcp.internal.example.com/postgres/mcp",
            "headers": {"Authorization": f"Bearer {DB_MCP_TOKEN}"},
        },
    }
)

mcp_tools = await mcp_client.get_tools()   # list[BaseTool] — confirmed async

def summarize_for_slack(text: str) -> str:
    """Shrink a long triage report into a Slack-friendly summary."""
    return text[:500]

agent = create_deep_agent(
    model=model,
    tools=mcp_tools + [summarize_for_slack],   # MCP tools and hand-written tools, one flat list
    system_prompt="You are a repo maintenance assistant with access to GitHub and the analytics database...",
)
```

Every line above `create_deep_agent(...)` is plain `langchain-mcp-adapters` usage — nothing about it is `deepagents`-aware, and nothing about it changes if you swap `create_deep_agent` for a bare `create_agent()` call. The **only** `deepagents`-specific line is the last one, and it's the same `tools=` argument you used in your very first deep agent.

### The async/sync boundary

`get_tools()` is async; `create_deep_agent()` itself is synchronous — it only assembles a middleware stack and compiles a graph (DeepAgents course Chapter 2), it does not open any MCP session on your behalf. That means the fetch has to happen **before** the constructor call, not inside it. In a FastAPI service, the natural home for this is the lifespan hook — the same place you'd already open a database connection pool:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    mcp_tools = await mcp_client.get_tools()
    app.state.repo_agent = build_repo_maintenance_agent(model, mcp_tools)
    yield
    # Transport-specific MCP session teardown, if your servers need it, goes here —
    # ordinary resource-lifecycle code, not a deepagents concern.

app = FastAPI(lifespan=lifespan)
```

Build the compiled graph once at startup and reuse it across requests, exactly as you would for a plain `create_agent()` service — a `CompiledStateGraph` is safe to share across concurrent `.ainvoke()` calls as long as each carries its own `thread_id`. The MCP tools bound to it don't change that; they're just more entries in an already-immutable tool list.

> **2026-07-28 spec note:** `create_deep_agent()` and `langchain-mcp-adapters` both sit *above* the wire protocol entirely — a `BaseTool`'s `.invoke()` call round-trips through whatever `ClientSession`/transport the adapter opened, and neither the deep agent nor the adapter's public API changes shape based on which spec revision a given server implements. The classic handshake (through 2025-11-25) is still what essentially every MCP server and every version of `langchain-mcp-adapters` speaks as of this writing (Chapter 21 covers the stateless 2026-07-28 redesign in full) — but from inside `create_deep_agent(tools=mcp_tools)`, that detail is fully absorbed by the adapter layer and never surfaces in your code.

### Tool name collisions, revisited for deep agents

Chapter 17 flagged that two servers can declare a tool with the identical name, and `get_tools()`'s flat list only preserves one addressable `.name` per collision. This matters more inside `deepagents` than in a plain LangGraph node, because two features in this chapter — subagent `tools=` filtering (Section 4) and `interrupt_on` keying (Section 5) — both select tools **by name**. Before relying on either, run:

```python
names = [t.name for t in mcp_tools]
assert len(names) == len(set(names)), "duplicate MCP tool names across servers"
```

A silent collision here doesn't just call the wrong tool — it can mean an `interrupt_on` gate you believe covers a destructive GitHub action is actually keyed to a same-named, unrelated tool from a different server.

---

## 4. Scoping MCP Tools to Specific Subagents

Because an MCP tool is just an entry in a `list[BaseTool]`, the `SubAgent` `tools=` key (DeepAgents course Chapter 8) subsets it exactly like it would subset any other tool list — MCP or not. If a `SubAgent` dict omits `tools`, it inherits the parent's full list; if it provides `tools`, that list *replaces* the inherited one. This is the mechanism for keeping a coordinator's context free of tool schemas it never calls directly, and for MCP integrations specifically, for keeping a subagent from even being *able* to reach a server it has no business touching.

```python
github_tools = [t for t in mcp_tools if t.name.startswith("github_")]
db_tools = [t for t in mcp_tools if t.name.startswith(("query_", "run_migration"))]

triage_subagent = {
    "name": "issue-triage",
    "description": (
        "Reads and labels open GitHub issues; searches for likely duplicates. "
        "Delegate any 'triage the backlog' or 'find duplicate issues' request here."
    ),
    "system_prompt": (
        "You are a GitHub issue triage specialist. For each open issue, determine "
        "severity, apply labels, and flag likely duplicates. You never open PRs or "
        "close issues yourself — write your findings to /triage/ for the coordinator."
    ),
    "tools": [t for t in github_tools if not t.name.endswith(("_merge_pr", "_close_issue"))],
}

db_subagent = {
    "name": "database",
    "description": "Runs read-only queries against the analytics warehouse to correlate issue reports with error rates.",
    "system_prompt": "You answer data questions using query_* tools only. You never run migrations.",
    "tools": [t for t in db_tools if t.name.startswith("query_")],
}

coordinator = create_deep_agent(
    model=model,
    tools=[],   # the coordinator itself holds no MCP tools directly
    subagents=[triage_subagent, db_subagent],
    system_prompt="Delegate issue-triage work to 'issue-triage' and data questions to 'database'.",
)
```

Two things are worth underlining here because they're easy to get backwards:

- **Scoping happens at the Python list-comprehension level, not inside MCP.** The GitHub MCP server itself doesn't know or care that you've split its tools between two subagents — the split is purely a `deepagents`-side filtering decision over an already-materialized `list[BaseTool]`.
- **A subagent that inherits (omits `tools`) still gets everything, including MCP tools from servers it may not need.** If you want the "least privilege" property Chapter 14's security material argues for, you must explicitly pass a filtered `tools=` list to every subagent that shouldn't see the full surface — an omitted `tools` key is an opt-in to *more* access, not less.

---

## 5. Gating Destructive MCP Tools with `interrupt_on`

An MCP tool arrives as a named `BaseTool` — `github_merge_pr`, `github_close_issue`, `query_run_migration`, whatever the server's `tools/list` schema called it (Chapter 4). That name is exactly what `interrupt_on` keys off of, so gating an MCP tool is identical to gating any hand-written tool: no new mechanism, same dict.

```python
from deepagents import create_deep_agent
from langchain.agents.middleware import InterruptOnConfig

agent = create_deep_agent(
    model=model,
    tools=github_tools + db_tools,
    subagents=[triage_subagent, db_subagent],
    interrupt_on={
        "github_merge_pr": True,
        "github_close_issue": True,
        "query_run_migration": InterruptOnConfig(allowed_decisions=["approve", "reject"]),
        # github_list_issues, github_get_issue, query_select — left ungated: read-only
    },
)
```

This is the direct continuation of **Chapter 14's human-approval-gate pattern**: a `destructiveHint`-style tool (something the MCP server's own `annotations` may flag, or that you classify yourself by reading its description) gets an `interrupt_on` entry; the graph pauses via `interrupt()` before the tool executes, your application surfaces the pending call — tool name and arguments — to a human, and you resume with `Command(resume=...)` to approve, edit the arguments, or reject. The tool call that reaches the interrupt is a standard LangChain `ToolCall` object regardless of whether it originated from `@tool` or from an MCP server's `tools/call` schema — there is no MCP-specific approval payload to learn.

### Why `permissions`/`FilesystemPermission` does *not* extend to MCP tools

It's tempting to read "`interrupt_on`/`permissions` are DeepAgents' human-in-the-loop mechanism" as meaning either one can gate anything, including an arbitrary MCP tool name. Look closely at the signature from Section 2: `permissions: list[FilesystemPermission] | None`. `FilesystemPermission` is a **path-and-operation-scoped rule** — it matches a filesystem operation (`write_file`, `edit_file`, `delete`) against a path pattern, and auto-generates `interrupt_on` entries for the built-in filesystem tools it covers. A `github_merge_pr` call has no filesystem path to match against — `FilesystemPermission` has nothing to attach to.

The correct mental model: **`interrupt_on` is the general mechanism, keyed by tool name, that covers every tool including MCP ones. `permissions`/`FilesystemPermission` is a convenience layer that generates `interrupt_on` entries specifically for the built-in filesystem surface.** For MCP tools, you always reach for `interrupt_on` directly — there is no MCP equivalent of `FilesystemPermission` to auto-generate gates for you, so the read/write classification table below is work you do by hand, once, per server:

| MCP tool (example) | Server | Effect | Gate with `interrupt_on`? |
|---|---|---|---|
| `github_list_issues` | GitHub | Read-only | No |
| `github_get_issue` | GitHub | Read-only | No |
| `github_add_labels` | GitHub | Mutates issue metadata, low blast radius | Team's call — often left ungated |
| `github_close_issue` | GitHub | Closes a real issue | Yes |
| `github_merge_pr` | GitHub | Merges code into a tracked branch | Yes |
| `query_select` | Database | Read-only query | No |
| `query_run_migration` | Database | Schema-altering write | Yes |

---

## 6. Built-in Filesystem Tools vs. an MCP Filesystem/GitHub Server

A deep agent's built-in filesystem middleware (`ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep` — DeepAgents course Chapter 5) and an MCP-provided filesystem or GitHub server look superficially similar — both let the agent read and write "files" — but they answer to completely different backends and exist for different reasons:

- **The built-in filesystem tools operate on the agent's own scratch space** — a virtual or backend-mapped filesystem (Chapter 6 of the DeepAgents course covers backend choices) used to keep large intermediate state out of the message history: a triage report draft, a JSON dump of query results, working notes between subagent calls. Nothing here talks to a real external system by default.
- **An MCP filesystem server, or a GitHub server's file-reading tools, reach an actual external resource** — a real directory on a real host, or a real file inside a real GitHub repository at a real commit. Calling `github_get_file` returns the actual contents of `src/app.py` on GitHub; calling the built-in `read_file` returns whatever your own agent previously wrote into its own scratch space.

A well-designed repo-maintenance agent uses **both, deliberately, for different halves of its state**: MCP's `github_get_file`/`github_search_code` to pull real repository content down for inspection, and the built-in `write_file` to persist its own triage notes, draft summaries, and reports — artifacts that belong to the agent's run, not to the repository. Conflating the two is a common design mistake (Section on Common Mistakes below returns to it): writing a triage report with a GitHub "create file" tool turns a scratch note into a permanent repository commit nobody asked for, while trying to `read_file` a GitHub path that was never fetched into scratch space just returns "file not found" against the wrong filesystem entirely.

---

## Examples

### Minimal end-to-end: two servers, one tool list, one agent

```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from deepagents import create_deep_agent

mcp_client = MultiServerMCPClient(
    {
        "github": {
            "transport": "stdio",
            "command": "github-mcp-server",
            "args": [],
            "env": {"GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_***"},
        },
        "database": {
            "transport": "stdio",
            "command": "uvx",
            "args": ["mcp-server-postgres", "--dsn", "postgresql://ro_user:***@warehouse/analytics"],
        },
    }
)

async def main():
    mcp_tools = await mcp_client.get_tools()

    agent = create_deep_agent(
        model="anthropic:claude-sonnet-4-5",
        tools=mcp_tools,
        interrupt_on={"github_merge_pr": True, "github_close_issue": True},
        system_prompt="You maintain a GitHub repository and can query its usage analytics.",
    )

    result = await agent.ainvoke({
        "messages": [{"role": "user", "content": "List every open issue labeled 'bug' with no assignee."}]
    })
    print(result["messages"][-1].content)

asyncio.run(main())
```

Every moving part above is either pure `langchain-mcp-adapters` (the client, the config dict, `get_tools()`) or pure `deepagents` (`create_deep_agent`, `interrupt_on`) — the seam between them is exactly the one line, `tools=mcp_tools`, this chapter has been building toward.

---

## Project: The Repo Maintenance DeepAgent

This project assembles every piece above into a single agent: triage the open-issue backlog of a repository, using a GitHub MCP server for real data, a scoped `issue-triage` subagent for context isolation, the built-in filesystem tools for a durable triage report, and an `interrupt_on` gate before any write action reaches GitHub.

### Requirements, translated into wiring decisions

1. **Read the backlog** — `github_list_issues`/`github_get_issue`/`github_search_issues` tools from a GitHub MCP server, via `MultiServerMCPClient`.
2. **Isolate the triage work** — an `issue-triage` subagent owns the GitHub *read* tools and does the actual per-issue reasoning, so the coordinator's own prompt and context stay small (DeepAgents course Chapter 8's core argument, applied here).
3. **Durable output, not hallucinated memory** — the coordinator writes the finished triage report to `/triage/report.md` using the *built-in* `write_file` tool (Section 6) — this is the agent's own scratch artifact, not a GitHub commit.
4. **Safety** — any tool that mutates the real repository (`github_add_labels`, `github_close_issue`, a hypothetical `github_comment`) is gated behind `interrupt_on` so a human reviews every batch of proposed label/close actions before they hit GitHub.
5. **Delegation boundary** — the coordinator never calls a GitHub tool directly; it only ever reads the subagent's short report and, if the human approves, dispatches the approved write actions itself (kept at the coordinator level specifically so the `interrupt_on` gate has one place to fire, not one per subagent).

### Wiring code

```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from deepagents import create_deep_agent
from langchain.agents.middleware import InterruptOnConfig

GITHUB_TOKEN = "ghp_***"  # loaded from your secret manager in practice

mcp_client = MultiServerMCPClient(
    {
        "github": {
            "transport": "stdio",
            "command": "github-mcp-server",
            "args": [],
            "env": {"GITHUB_PERSONAL_ACCESS_TOKEN": GITHUB_TOKEN},
        },
    }
)


def split_github_tools(mcp_tools):
    read_names = {"github_list_issues", "github_get_issue", "github_search_issues", "github_get_file"}
    write_names = {"github_add_labels", "github_close_issue", "github_comment"}
    read_tools = [t for t in mcp_tools if t.name in read_names]
    write_tools = [t for t in mcp_tools if t.name in write_names]
    return read_tools, write_tools


async def build_repo_maintenance_agent(model):
    mcp_tools = await mcp_client.get_tools()
    read_tools, write_tools = split_github_tools(mcp_tools)

    triage_subagent = {
        "name": "issue-triage",
        "description": (
            "Reads open GitHub issues, assesses severity, and drafts label/close "
            "recommendations. Delegate any 'triage the backlog' request here. "
            "This subagent never mutates GitHub directly."
        ),
        "system_prompt": (
            "You are a GitHub issue triage specialist. For every open issue: "
            "read its title/body/comments, classify severity (critical/normal/minor), "
            "identify likely duplicates by searching titles, and propose labels. "
            "Write your full findings to /triage/notes.json as structured data. "
            "You have NO write access to GitHub — every recommendation is a draft "
            "for the coordinator and a human to review."
        ),
        "tools": read_tools,   # read-only GitHub tools only — no write_names here
    }

    coordinator = create_deep_agent(
        model=model,
        tools=write_tools,   # only the coordinator can ever call a write tool
        subagents=[triage_subagent],
        system_prompt=(
            "You coordinate GitHub repo maintenance. Delegate the actual issue "
            "review to the 'issue-triage' subagent, read its /triage/notes.json "
            "output back with read_file, synthesize a human-readable report at "
            "/triage/report.md via write_file, and only after the human approves "
            "your proposed actions, apply labels/closes using your GitHub write tools."
        ),
        interrupt_on={
            "github_add_labels": True,
            "github_close_issue": True,
            "github_comment": InterruptOnConfig(allowed_decisions=["approve", "edit", "reject"]),
        },
    )
    return coordinator


async def main():
    agent = await build_repo_maintenance_agent(model="anthropic:claude-sonnet-4-5")
    result = await agent.ainvoke({
        "messages": [{
            "role": "user",
            "content": "Triage every open issue in this repo, write a report, and "
                       "propose labels for anything that looks stale or duplicate.",
        }]
    })
    print(result["messages"][-1].content)

asyncio.run(main())
```

### Diagram: request flow through the Repo Maintenance DeepAgent

```mermaid
flowchart TD
    U["User: 'triage the backlog'"] --> COORD["Coordinator<br/>(planning + write-tool gate)"]
    COORD -->|"task('issue-triage', ...)"| SUB["issue-triage subagent<br/>(read-only GitHub tools)"]
    SUB -->|"github_list_issues, github_get_issue,<br/>github_search_issues"| GH1["GitHub MCP server"]
    SUB -->|"write_file('/triage/notes.json', ...)"| FS["Built-in scratch filesystem"]
    SUB -->|"short report only"| COORD
    COORD -->|"read_file('/triage/notes.json')"| FS
    COORD -->|"write_file('/triage/report.md', ...)"| FS
    COORD -->|"github_add_labels / github_close_issue"| GATE{"interrupt_on<br/>fires -> human review"}
    GATE -->|"approve"| GH2["GitHub MCP server<br/>(mutating call executes)"]
    GATE -->|"reject/edit"| COORD

    style GATE fill:#ffe0e0
    style GH1 fill:#fff3e0
    style GH2 fill:#fff3e0
    style FS fill:#e1f5ff
```

Notice what never happens in this diagram: the subagent never touches a write-capable GitHub tool (it was never given one), the coordinator never bypasses the `interrupt_on` gate (there's no code path that would let it), and the triage notes never leave the agent's own scratch filesystem to become a real GitHub artifact until a human explicitly approves a specific action. Every one of those properties comes from ordinary `deepagents` mechanisms — `SubAgent.tools`, `interrupt_on`, the built-in filesystem — applied to tools that happen to be MCP-sourced.

---

## Real-World Scenario

A platform team is building exactly this kind of maintenance agent for a 40-repository open-source organization. Early in the design, an engineer proposes giving the coordinator the *entire* GitHub MCP tool list directly — read and write both — with a single `interrupt_on` entry gating every tool whose name contains `"delete"`, reasoning that this is simpler than splitting tools between a subagent and the coordinator.

The proposal is revised after two problems surface in review, both of which map directly onto this chapter's material:

- **Naming isn't a reliable safety boundary.** The GitHub MCP server in use ships tools named `github_close_issue`, `github_merge_pr`, and `github_add_labels` — none of which contain the substring `"delete"`, so the proposed `interrupt_on` filter silently gates nothing that actually matters. The fix is the same discipline this chapter's read/write classification table demonstrates: enumerate every tool name the server actually exposes and classify each one by hand, rather than pattern-matching on a name convention the server author never promised to follow.
- **A subagent without a `tools=` override inherits everything.** The team had planned to add an `issue-triage` subagent later "as a context-management optimization," assuming it could be bolted on without revisiting safety. Because an omitted `tools` key inherits the *parent's full list*, adding that subagent without an explicit `tools=` override would have quietly handed the triage subagent the same write-capable GitHub tools the coordinator has — including ones no `interrupt_on` entry was written to expect a subagent-originated call for. The team explicitly scoped the subagent's `tools=` to the read-only subset from day one, precisely as Section 4 recommends, closing the gap before it shipped.

The shipped design matches Section's worked project almost exactly: read tools live on a scoped subagent, write tools live only on the coordinator, and every write-capable tool name has an explicit, individually reasoned `interrupt_on` entry — no substring matching, no inherited surprises.

---

## Best Practices

- **Fetch MCP tools once, at process startup, not per-request.** `get_tools()` is async and `create_deep_agent()` is not — resolve the tool list in a lifespan/startup hook and build the compiled graph once; reuse it across requests keyed by `thread_id`.
- **Classify every MCP tool name as read or write by hand before wiring `interrupt_on`.** There is no `FilesystemPermission`-style auto-generation for arbitrary MCP tools — the table has to be built manually, once per server, and revisited whenever a server adds a tool.
- **Give write-capable tools only to the coordinator (or a narrowly-scoped subagent built specifically to hold them), never to a general-purpose or read-focused subagent by omission.** An omitted `tools=` key inherits everything; treat that as a reason to always pass an explicit list to any subagent that shouldn't have full access.
- **Keep the built-in filesystem tools for the agent's own scratch state, and MCP filesystem/GitHub tools for the real external resource.** Don't let a triage report accidentally become a GitHub commit, and don't expect `read_file` to see a repository's real files unless something explicitly fetched them into scratch space first.
- **Check for tool-name collisions across configured MCP servers before relying on name-based `interrupt_on` or subagent `tools=` filtering.** A silent collision breaks both mechanisms at once, and it breaks them quietly — the wrong tool gets gated, or the wrong tool reaches the subagent.
- **Don't go looking for an `mcp_servers=` parameter, a `with_mcp()` builder, or an `McpMiddleware` class.** None exist. If a tutorial or blog post shows one, treat it as either speculative or wrong — the verified signature is Section 2's, and MCP integration is, deliberately, just `tools=`.

---

## Common Mistakes

- **Assuming `create_deep_agent()` has first-class MCP support beyond `tools=`.** It doesn't, and the absence is intentional — MCP tools and hand-written tools are indistinguishable once they satisfy `BaseTool`, and the constructor has no reason to treat them differently.
- **Calling `get_tools()` inside `create_deep_agent()`'s call site, expecting it to await something for you.** `create_deep_agent()` is synchronous; the async fetch must complete first, in your own `async def` startup code.
- **Reaching for `permissions`/`FilesystemPermission` to gate an MCP tool by name.** `FilesystemPermission` is a path-and-operation rule scoped to the built-in filesystem tools; it has no path to match against a `github_merge_pr` call. Use `interrupt_on` directly for any non-filesystem tool, MCP or otherwise.
- **Letting a subagent inherit write-capable MCP tools by omitting its own `tools=` key.** "I'll scope it down later" is exactly the gap the Real-World Scenario's second finding closes — scope it down when the subagent is created, not as a follow-up ticket.
- **Using an MCP GitHub/filesystem tool to persist the agent's own working notes, or the built-in `write_file` to try to change a real repository file.** These are two different backends with two different blast radii; conflating them either pollutes a real external system with scratch artifacts or silently fails to produce the external change you intended.
- **Filtering `interrupt_on`/subagent `tools=` by a name substring convention (`"delete"`, `"write"`) instead of enumerating actual tool names.** MCP server authors are under no obligation to name destructive tools predictably — `github_close_issue` and `github_merge_pr` are real, extremely consequential examples that a `"delete"`-substring filter would miss entirely.
- **Not checking for duplicate tool names across a multi-server `MultiServerMCPClient` config before relying on name-based logic anywhere downstream.** This bites hardest in exactly the multi-server setups this chapter builds — check `[t.name for t in mcp_tools]` for duplicates as a standing habit, not an occasional debugging step.

---

## Summary

- `create_deep_agent()` has **no `mcp_servers=`/`mcp=`/`mcp_client=` parameter** — confirmed against the current source signature. MCP integration is entirely: build a `MultiServerMCPClient`, `await client.get_tools()`, pass the resulting `list[BaseTool]` through the ordinary `tools=` argument, exactly like any other LangChain tool.
- The framing for this whole chapter: **DeepAgents supplies the reasoning/orchestration substrate (planning, memory, filesystem, subagents); MCP supplies the standardized external capabilities (databases, GitHub, internal APIs).** Neither layer is aware of the other's internals.
- `get_tools()` is async; `create_deep_agent()` is not — fetch the tool list once, typically in a FastAPI lifespan hook, and build the compiled graph once at startup.
- A `SubAgent`'s `tools=` key subsets MCP tools exactly like any other tool list — an omitted `tools` key inherits the parent's **full** list, which is the most common way a subagent ends up with write access nobody intended.
- `interrupt_on`, keyed by tool name, is the general human-in-the-loop mechanism that covers MCP tools directly. `permissions`/`FilesystemPermission` is scoped specifically to the built-in filesystem tools and does not auto-generate gates for arbitrary MCP tool names — classify and gate MCP tools by hand.
- The built-in filesystem tools (`ls`/`read_file`/`write_file`/`edit_file`/`glob`/`grep`) operate on the agent's own scratch space; an MCP filesystem/GitHub server reaches a real external resource. Use both, deliberately, for different halves of an agent's state.
- The Repo Maintenance DeepAgent project ties every piece together: a scoped `issue-triage` subagent holding only read-capable GitHub tools, a coordinator holding the write-capable ones, an `interrupt_on` gate before any GitHub mutation, and the built-in filesystem tools for a durable, non-hallucinated triage report.

---

## Knowledge Check

1. Quote, or closely paraphrase, why `create_deep_agent()` has no `mcp_servers=` parameter, referencing the actual type of the `tools=` argument.
2. Why must `MultiServerMCPClient(...).get_tools()` be awaited *before* `create_deep_agent()` is called, rather than inside it?
3. A subagent's `SubAgent` dict omits the `tools` key entirely. What tools does it end up with, and why is that a common source of unintended write access when MCP tools are involved?
4. Explain precisely why `permissions=[FilesystemPermission(...)]` cannot be used to gate a `github_merge_pr` tool call, and what mechanism you'd use instead.
5. Give one concrete example of state that belongs in the built-in filesystem tools' scratch space, and one concrete example of state that only an MCP GitHub/filesystem server can provide — and explain why swapping them would be a mistake.
6. Two MCP servers in one `MultiServerMCPClient` config both expose a tool literally named `search`. What breaks, specifically, if you then try to write `interrupt_on={"search": True}`?
7. In the Repo Maintenance DeepAgent project, why does the coordinator — not the `issue-triage` subagent — hold every write-capable GitHub tool?

---

## Hands-On Exercise

1. **Reproduce the misconception check.** Without looking back at Section 2, write down from memory what you believe `create_deep_agent()`'s full parameter list is, then compare it against Section 2's verified signature. Note any parameter you incorrectly assumed existed.
2. **Build a two-server client.** Configure a `MultiServerMCPClient` with a GitHub MCP server (stdio, with a token in `env`) and any second server you have access to (a database server, or a second `stdio` server of your choosing). `await get_tools()` and print `[t.name for t in mcp_tools]`, checking for name collisions as Section 3 describes.
3. **Split and scope.** Using your printed tool list, manually classify every tool as read or write (Section 5's table), then write a `triage_subagent` dict whose `tools=` contains only the read-classified tools, and a coordinator `create_deep_agent()` call whose own `tools=` contains only the write-classified ones, with an `interrupt_on` entry for every write tool.
4. **Trigger and resolve an interrupt.** Invoke your agent with a prompt that would require a write-capable tool call, confirm the graph pauses at the `interrupt_on` gate, inspect the pending `ToolCall`'s name and arguments, and resume with an explicit approve/reject decision. Confirm the tool only executes after approval.
5. **Extend the report loop.** Add a coordinator instruction to write a Markdown summary via the built-in `write_file` to `/triage/report.md`, then `read_file` it back in a follow-up turn to confirm the scratch-space round-trip works independently of any MCP server.

---

## Further Reading

- DeepAgents course: **[Chapter 2 — Architecture & Internals](../deepagents-course/02-architecture-and-internals.md)** — the full `create_deep_agent()` signature this chapter builds on
- DeepAgents course: **[Chapter 5 — Filesystem-Backed Context](../deepagents-course/05-filesystem-backed-context.md)** — the built-in tool surface contrasted with MCP servers in Section 6
- DeepAgents course: **[Chapter 8 — Subagent Orchestration](../deepagents-course/08-subagent-orchestration.md)** — the `SubAgent` shape and context-isolation argument this chapter applies to MCP tools
- DeepAgents course: **[Chapter 9 — Human-in-the-Loop](../deepagents-course/09-human-in-the-loop.md)** — `interrupt_on`, `InterruptOnConfig`, and `FilesystemPermission` in full depth
- DeepAgents course: **[Chapter 11 — MCP Integration](../deepagents-course/11-mcp-integration.md)** — the DeepAgents-side companion to this chapter, written from the "you already know MCP" direction
- This course: **[Chapter 14 — MCP Security](./14-mcp-security.md)** — the destructive-action gating pattern this chapter's `interrupt_on` examples extend
- This course: **[Chapter 17 — MCP + LangChain](./17-mcp-with-langchain.md)** — full `MultiServerMCPClient`/`get_tools()` mechanics assumed as background here
- This course: **[Chapter 18 — MCP + LangGraph](./18-mcp-with-langgraph.md)** — the same tool list wired into a bare LangGraph node instead of `create_deep_agent()`
- This course: **[Chapter 20 — Production MCP Architecture](./20-production-mcp-architecture.md)** — what changes about this chapter's patterns once the agent is under real production load
- Official source: `github.com/langchain-ai/deepagents` (`graph.py`) — the verified `create_deep_agent()` signature quoted in Section 2
- Official source: `github.com/langchain-ai/langchain-mcp-adapters` — `MultiServerMCPClient`, `get_tools()`

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./18-mcp-with-langgraph.md">← Previous: MCP + LangGraph</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./20-production-mcp-architecture.md">Next: Production MCP Architecture →</a>
</div>
