# Subagent Orchestration

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain, in concrete mechanical terms, why a single flat agent handling ten unrelated concerns degrades — and
  why subagent delegation is the structural fix, not just a stylistic preference
- Distinguish the three subagent shapes accepted by `create_deep_agent(subagents=[...])` — `SubAgent`,
  `CompiledSubAgent`, `AsyncSubAgent` — and choose correctly among them for a given design problem
- Trace exactly what happens between the parent agent calling the `task` tool and a final report landing back in
  the parent's message list, including what is deliberately thrown away along the path
- Reason precisely about "context isolation": what a subagent's compiled graph sees, what it doesn't, and why
  none of its intermediate tool calls ever reach the parent's context window
- Exploit the shared-`backend`/separate-middleware-stack nuance — a subagent can `write_file` something the
  parent later `read_file`s, even though nothing about that file passed through either agent's message history
- Decide when to override the auto-added `general-purpose` default subagent, and when to leave it alone
- Apply per-subagent `model=` overrides for cost/latency tuning across a mixed-capability pipeline
- Build a three-subagent "Code Review Agent" coordinator end to end, with focused system prompts, scoped tool
  lists, and a coordinator prompt that routes work through `task` correctly

---

## Prerequisites for This Chapter

This chapter assumes Chapters 1–7. Specifically, you should already be comfortable with:

- The middleware-stack framing of `create_deep_agent()` from Chapter 2 — `SubAgentMiddleware` is one component
  in that stack, sitting alongside `FilesystemMiddleware`, summarization, and (from Chapter 7) `MemoryMiddleware`
- `create_deep_agent()`'s core call shape from Chapter 3 — `model`, `tools`, `system_prompt` — since every
  subagent definition in this chapter reuses that same shape internally
- Backends (`StateBackend`, `FilesystemBackend`, `StoreBackend`, `CompositeBackend`) from Chapter 6, since
  Section 5 of this chapter depends directly on the fact that a `backend` is a shared resource, not a
  per-agent-instance one
- `FilesystemPermission` at a passing level — Chapter 6 introduced it for backend-level access control; this
  chapter uses it again inside a `SubAgent` dict's `permissions` key without re-deriving it

This chapter does not re-explain LangGraph's `StateGraph`/`Command` mechanics, MCP, or general multi-agent
architecture patterns — Chapter 12 (Multi-Agent Systems) goes further into sync-vs-async subagent architecture
and coordinator design patterns; this chapter's job is narrower: understand *precisely* what a subagent is,
what the `task` tool does, and why context isolation is the actual point.

---

## 1. Why Flat Agents Degrade

Before any API surface, sit with the failure mode subagents exist to fix.

Imagine a single deep agent asked to do code review end to end: search the codebase for related patterns, read
relevant docs, propose a fix, write the fix, write tests, run the tests, and summarize a verdict. Cram all of
that into one `system_prompt` and one flat tool-calling loop, and two independent problems compound:

**Problem 1 — the system prompt becomes unwieldy.** A system prompt trying to cover "how to search effectively,"
"how to propose a safe fix," and "how to write and validate a test" simultaneously must hold all three concerns
active in the model's attention at once, for every single turn, even the turns where only one of those concerns
is actually relevant. The model can't selectively "forget" the testing instructions while it's doing research —
they're all sitting in context, competing for attention, all the time.

**Problem 2 — every tool call from every concern piles into one shared message history.** Searching the
codebase for related functions might take fifteen `grep`/`read_file` calls before anything useful surfaces.
Every one of those fifteen calls — and their (often verbose) results — stays in the message list forever,
consuming context budget that the *coding* and *testing* phases now have to compete for, even though none of
that research noise is relevant to writing the fix itself. By the time the agent reaches the testing phase, its
context window is mostly filled with research exhaust and fix-drafting back-and-forth it doesn't need anymore.

Subagents fix both problems at the structural level, not by prompting harder:

- Each subagent gets its **own, narrow `system_prompt`** — the research subagent's prompt only ever needs to
  describe how to search and summarize; it never has to also carry testing conventions.
- Each subagent gets its **own private context window**. Its fifteen `grep` calls, its false starts, its
  intermediate reasoning — none of it is visible to the parent. The parent only ever receives one thing: the
  subagent's final report.

This is the trade the rest of the chapter unpacks mechanically: the parent pays a fixed, small cost (one `task`
call, one final report) no matter how much internal back-and-forth a subagent needed to reach that report. A
flat agent pays for every step of every concern, forever, in the same shared budget.

---

## 2. The Three Subagent Shapes

`create_deep_agent(subagents=[...])` accepts a list that can freely mix three distinct shapes. All three are
dispatched through the same `task` tool (Section 3), but they differ sharply in how they're constructed and what
kind of workload they suit.

### 2.1 `SubAgent` — the declarative, common case

A `TypedDict`. This is what you reach for by default — you describe a subagent, and `deepagents` assembles a
fully middleware-wrapped agent from that description, the same way `create_deep_agent()` assembles the parent.

**Required keys:**

| Key | Type | Meaning |
|---|---|---|
| `name` | `str` | Identifier the `task` tool's `subagent_type` argument must match exactly |
| `description` | `str` | Shown to the parent model — this is how the parent decides *which* subagent fits a task, so write it the same way you'd write a tool docstring: precise, decision-relevant, no fluff |
| `system_prompt` | `str` | This subagent's own, focused system prompt — entirely separate from the parent's |

**Optional keys:**

| Key | Type | Meaning |
|---|---|---|
| `tools` | `list` | If omitted, the subagent **inherits the parent's full tool list**. If provided, it replaces the inherited list — scope it down deliberately (Section 8's `research` subagent, for example, gets no write-capable tools at all) |
| `model` | `str \| BaseChatModel` | Overrides the parent's model for this subagent only — Section 7 covers this in depth |
| `middleware` | `list[AgentMiddleware]` | Subagent-specific middleware, layered onto its own stack |
| `interrupt_on` | (same shape as the parent's) | **Overrides the parent's `interrupt_on` entirely if set — it does NOT merge.** See Common Mistakes; this is a sharp edge |
| `skills` | `list[str]` | Skills available to this subagent (Chapter 14 territory) |
| `permissions` | `list[FilesystemPermission]` | Filesystem access scoping for this subagent, on top of the shared backend (Chapter 6, Chapter 9) |
| `response_format` | (structured output spec) | Constrains this subagent's final output shape |

```python
research_subagent: SubAgent = {
    "name": "research",
    "description": (
        "Searches the codebase and any available docs for context relevant to "
        "a code review — related functions, existing tests, prior art, and "
        "relevant design docs. Use this before proposing any fix, to confirm "
        "you understand existing conventions. Returns a written summary, not code."
    ),
    "system_prompt": (
        "You are a research specialist supporting a code review. Search the "
        "codebase and any docs using the tools available to you, and produce "
        "a concise written summary of what you find: relevant functions, "
        "existing patterns, and anything that looks related to the reported "
        "issue. You do not write or modify any files — you only report findings."
    ),
    "tools": [grep, glob, read_file, ls],  # read-only — no write_file/edit_file
    "permissions": [FilesystemPermission(path="/", mode="read")],
}
```

### 2.2 `CompiledSubAgent` — bring your own subgraph

Instead of letting `deepagents` assemble a subagent from a declarative description, you supply an already
**pre-compiled `Runnable`/graph** directly. Use this when the subagent's internal topology needs to be something
`create_deep_agent()` itself can't express — a custom LangGraph `StateGraph` with conditional branching, a
hand-assembled `create_agent` call with middleware combinations `SubAgent`'s optional keys don't cover, or a
graph you've already built and validated independently and simply want to slot into the delegation tree.

Both shapes are exposed through the **same** `task` tool — the parent model doesn't need to know or care whether
a given `subagent_type` resolves to a `SubAgent`-assembled graph or a hand-built `CompiledSubAgent`; the
dispatch mechanism is identical either way.

### 2.3 `AsyncSubAgent` — remote, background subagents

A `TypedDict` for a fundamentally different kind of subagent: one that isn't invoked synchronously through
`task` at all.

**Required key:** `graph_id: str` — identifies a remote graph, intended for a LangSmith-deployed graph.
**Optional keys:** `url: str`, `headers: dict[str, str]` — for reaching that remote deployment.

When any `AsyncSubAgent`s are supplied, `deepagents` installs a completely separate `AsyncSubAgentMiddleware`
(only installed if async subagents are actually configured — it isn't part of the default stack). That
middleware exposes its **own** set of tools for launching, checking status on, updating, cancelling, and listing
these remote tasks — not the `task` tool. This is the mechanism for genuinely long-running background work you
do not want the parent blocking on synchronously: a remote graph that might run for minutes or hours, where the
parent should be able to launch it, go do something else, and check back later rather than sitting inside one
blocking `task` call the whole time.

The exact tool names this middleware exposes weren't pinned down beyond "launch/check/update/cancel/list" at the
source level for this chapter — treat that as the right conceptual granularity rather than guessing at specific
tool names; verify exact names against your installed `deepagents` version if you're wiring this up.

### 2.4 Choosing among the three

| Shape | Dispatch | Use when |
|---|---|---|
| `SubAgent` | `task` tool (synchronous) | The common case — a focused sub-task with its own prompt/tools/model, assembled from a plain declarative description. Start here always. |
| `CompiledSubAgent` | `task` tool (synchronous) | You need custom graph topology `SubAgent`'s declarative keys can't express — conditional branching, bespoke middleware composition, or a graph you've already built and validated elsewhere |
| `AsyncSubAgent` | Separate launch/check/update/cancel/list tools (asynchronous, remote) | Genuinely long-running background work — a LangSmith-deployed graph you don't want blocking the parent's synchronous `task` call |

---

## 3. The `task` Tool — the Delegation Mechanism

Both `SubAgent` and `CompiledSubAgent` entries are invoked through exactly one tool: `task`, built by
`_build_task_tool` inside `SubAgentMiddleware`. There is no other dispatch mechanism for these two shapes — the
parent model never calls a subagent "directly"; it always goes through `task`.

The tool's input schema is deliberately small:

```python
class TaskInput(TypedDict):
    description: str       # the task to hand off, in natural language
    subagent_type: str      # must match a declared subagent's `name` exactly
```

The parent model picks `subagent_type` by reading each declared subagent's `description` field — exactly the
same "read the docstring, pick a name" mechanism you already rely on for ordinary tool selection (Chapter 3,
Section 5). This is why a subagent's `description` matters as much as a tool's docstring: it's the only signal
the parent has for routing.

---

## 4. Context Isolation — the Core Mechanism

This is the chapter's central idea, and it's confirmed directly in the `task` tool's own description string:

> "Each invocation is stateless: the agent sees only the prompt you give it and returns a single final
> report... The agent's report is not shown to the user."

Unpack that sentence mechanically, step by step.

### 4.1 What happens on a `task` call

1. The parent model emits a tool call: `task(description="...", subagent_type="research")`.
2. `SubAgentMiddleware` looks up the `SubAgent`/`CompiledSubAgent` registered under that `name`.
3. A **fresh `subagent_state`** is constructed for this invocation — not a reference into the parent's live
   state, a distinct state object built specifically for this one call.
4. Before that fresh state is constructed, certain **parent state keys are stripped** via `private_state_keys` —
   subagents do not automatically inherit everything the parent's state happens to hold.
5. `subagent.invoke(subagent_state, subagent_config)` runs — a full, separate invocation of a **separate
   `create_agent`-compiled graph**, built fresh for this `SubAgent` in `graph.py`'s subagent-processing loop.
   This compiled graph has its **own entire middleware stack**: its own `FilesystemMiddleware` instance, its own
   summarization middleware, everything — assembled independently of the parent's stack, even though (Section 5)
   it shares the same underlying `backend`.
6. That subagent's graph runs its **own full model-tools loop internally** — as many internal tool calls,
   reasoning steps, and intermediate messages as it needs. None of this is visible outside the subagent's own
   invocation.
7. Only the **extracted final result** — the single final report — is returned via a `Command` state update back
   into the parent's message list.

The intermediate tool calls, the reasoning, the entire internal message history of step 6: none of it leaks. The
parent's context window never grows by the cost of the subagent's internal work — only by the cost of the one
final report.

### 4.2 Sequence diagram

```mermaid
sequenceDiagram
    participant Parent as Parent Agent (Coordinator)
    participant TaskTool as task tool (SubAgentMiddleware)
    participant SubGraph as Subagent's own compiled graph
    participant SubModel as Subagent's bound model + tools

    Parent->>TaskTool: task(description="...", subagent_type="research")
    TaskTool->>TaskTool: look up SubAgent by name; strip private_state_keys
    TaskTool->>SubGraph: construct fresh subagent_state; subagent.invoke(subagent_state, subagent_config)
    loop Subagent's own internal model-tools loop (invisible to Parent)
        SubGraph->>SubModel: model call with subagent's own system_prompt + tools
        SubModel-->>SubGraph: tool call (grep, read_file, ...)
        SubGraph->>SubGraph: execute tool, append ToolMessage
    end
    SubModel-->>SubGraph: final AIMessage (the report)
    SubGraph-->>TaskTool: extracted final result only
    TaskTool-->>Parent: Command update — final report appended to Parent's messages
    Note over Parent: Parent never sees any message from the loop above —<br/>only the single final report string
```

The note on the diagram is the entire point of the chapter: everything inside the `loop` block is real
execution, real tokens, real latency — but it is **structurally invisible** to the parent. The parent's context
budget is charged for exactly one thing from this whole exchange: the final report.

### 4.3 The nuance: this is not a new checkpointed thread

It's tempting to describe this as "the subagent spawns a new checkpointed thread," because durable execution
(Chapter 10) also involves separate invocations and separate state. Resist that description — it's wrong in an
important way.

A `task` call is a separate invocation of a **separate compiled graph object**, happening **within the same
overall `task` tool call that the parent is synchronously waiting on**. It is not a new LangGraph `thread_id`,
it is not checkpointed independently for crash-recovery purposes, and the parent does not "resume" it later the
way it would resume an interrupted durable run. The parent blocks on this one `task` invocation until the
subagent's internal loop finishes and returns its report — same request, same overall execution, just a nested
graph invocation with its own isolated state and message history. Chapter 10's checkpointing model is a
different axis entirely: durability across process restarts, not context isolation within one live request.

---

## 5. Shared Backend, Separate Middleware Stack

Here's the nuance that makes subagents genuinely useful beyond "smaller prompts": each subagent gets its **own**
`FilesystemMiddleware` instance — a distinct object, with its own lifecycle — but that instance is constructed
against the **same underlying `backend`** the parent uses. Files are not scoped per-agent; the backend is a
shared resource (Chapter 6).

Concretely: a subagent can `write_file` something, return its final report with no mention of the file's
contents, and the **parent can later `read_file` that exact path** and see everything the subagent wrote — even
though not one byte of the file's content ever appeared in either agent's message history. The file is the
channel; the message list is not.

```python
from deepagents import create_deep_agent, SubAgent

coding_subagent: SubAgent = {
    "name": "coding",
    "description": (
        "Proposes and writes code fixes for a reported issue. Writes the "
        "actual patched file(s) to the shared filesystem and returns a short "
        "summary of what changed and why — the caller should read the files "
        "directly for the full diff rather than expecting it inline in the report."
    ),
    "system_prompt": (
        "You are a coding specialist. Given a description of an issue and any "
        "research summary provided, write the fixed version of the affected "
        "file(s) using write_file/edit_file. Keep changes minimal and scoped "
        "to the reported issue. In your final report, name exactly which "
        "file paths you wrote or edited — do not paste the full file contents "
        "into your report; the caller will read the files directly."
    ),
    "tools": [read_file, write_file, edit_file, grep, glob, ls],
    "model": "anthropic:claude-opus-4-6",  # stronger model for the harder task — Section 7
}

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    subagents=[coding_subagent],
    system_prompt="You are a code review coordinator...",
)

result = agent.invoke({
    "messages": [{
        "role": "user",
        "content": "Fix the off-by-one bug in paginate() in utils/pagination.py",
    }]
})

# The parent's own message history never contains the patched file's contents —
# only the coding subagent's short report. But because both agents share the
# same backend, the parent (or a human, or a later `testing` subagent call)
# can retrieve the actual patch directly:
# read_file("utils/pagination.py")  -> full patched contents, no message-history cost
```

This is exactly why the `coding` subagent's system prompt above explicitly tells it *not* to paste file contents
into its report — doing so would defeat the entire point of context isolation by re-inflating the parent's
context with something the shared filesystem already makes available on demand.

---

## 6. The Default `general-purpose` Subagent

If your `subagents=` list does not contain an entry named exactly `"general-purpose"`, `deepagents`
**auto-adds one** (`GENERAL_PURPOSE_SUBAGENT`). Its description:

> "General-purpose agent for researching complex questions, searching for files and content, and executing
> multi-step tasks... This agent has access to all tools as the main agent."

This exists so that `subagents=[]` (or an omitted `subagents` argument) still leaves the `task` tool with
*something* to dispatch to, rather than a delegation tool with zero valid targets. It's a sensible, unscoped
fallback — full tool access, general description — not a specialist.

**When to override or disable it:**

- If you supply your own explicit subagents (as in Section 8) and don't want the parent falling back to an
  unscoped general-purpose delegate for tasks that should go through one of your specialists, you can disable
  the auto-add via a `HarnessProfile`'s `GeneralPurposeSubagentProfile(enabled=False)` mechanism. This is Chapter
  2/13 territory in full — the point to hold onto here is narrower: the mechanism exists, and it only makes
  sense to use once you've already supplied explicit sync subagents of your own. Disabling it with zero other
  subagents declared leaves the `task` tool with nothing to dispatch to at all.
- If you're fine with an unscoped fallback existing alongside your specialists — e.g. for genuinely
  miscellaneous requests that don't cleanly fit any of your named subagents — leave it as is. It costs nothing
  unless the parent actually routes a task to it.

---

## 7. Model Overrides Per Subagent

A `SubAgent`'s optional `model` key — a `"provider:model"` string or a live `BaseChatModel` instance — overrides
the parent's model **for that one subagent only**. This is a direct cost/latency lever, not just a capability
lever: a subagent doing simple, mechanical work doesn't need the same model tier as one doing the hardest
reasoning in the pipeline.

```python
research_subagent: SubAgent = {
    "name": "research",
    "description": "Searches code and docs for context relevant to a review. Read-only.",
    "system_prompt": "...",
    "tools": [grep, glob, read_file, ls],
    "model": "anthropic:claude-haiku-4-6",  # cheap, fast — this is mechanical search-and-summarize work
}

coding_subagent: SubAgent = {
    "name": "coding",
    "description": "Proposes and writes code fixes.",
    "system_prompt": "...",
    "tools": [read_file, write_file, edit_file, grep, glob, ls],
    "model": "anthropic:claude-opus-4-6",  # strongest available — correctness-critical reasoning
}
```

Nothing about `subagents=[...]` requires every subagent to share the parent's model. In a multi-subagent
pipeline where one subagent's job is "grep around and summarize" and another's is "write a correct patch to
production code," treating both as equally deserving of your most expensive model is a wasted spend — the
research subagent's job doesn't get meaningfully better with a stronger model, but the coding subagent's
correctness very much can.

---

## 8. Project: The Code Review Agent

Assemble a coordinator with three declarative subagents — `research`, `coding`, `testing` — each with a focused
`system_prompt` and a deliberately scoped `tools` list.

### 8.1 The `research` subagent — read-only

```python
from deepagents import SubAgent
from langchain_core.tools import tool

# Assume grep/glob/read_file/ls are the built-in filesystem tools, always
# present on any deep agent — listed here explicitly to show exactly what
# this subagent's *scoped* tools list contains.
from deepagents.tools import grep, glob, read_file, ls

research_subagent: SubAgent = {
    "name": "research",
    "description": (
        "Searches the codebase and any available docs for context relevant to "
        "a code review — related functions, existing tests, prior conventions, "
        "and anything resembling the reported issue elsewhere in the codebase. "
        "Use this FIRST, before delegating to 'coding', so the fix respects "
        "existing patterns. Returns a written summary only; never writes files."
    ),
    "system_prompt": (
        "You are a research specialist supporting a code review. Given a "
        "description of an issue, search the codebase using grep/glob/read_file "
        "to find: (1) the specific code implicated, (2) any similar patterns "
        "elsewhere in the codebase that suggest the intended convention, and "
        "(3) any existing tests that already cover related behavior. Produce a "
        "concise written summary covering all three. Do not propose a fix "
        "yourself — that is the 'coding' subagent's job. You have no write "
        "access; only read/search tools are available to you."
    ),
    "tools": [grep, glob, read_file, ls],
    "model": "anthropic:claude-haiku-4-6",
}
```

### 8.2 The `coding` subagent — can write files

```python
from deepagents.tools import write_file, edit_file

coding_subagent: SubAgent = {
    "name": "coding",
    "description": (
        "Proposes and writes a code fix for a reported issue, given a research "
        "summary. Writes the patched file(s) directly to the shared filesystem "
        "and reports back only the file paths changed and a short rationale — "
        "not the full diff inline. Use this AFTER 'research' has run."
    ),
    "system_prompt": (
        "You are a coding specialist fixing a specific reported issue. You will "
        "be given a description of the issue and, typically, a prior research "
        "summary. Write the smallest correct fix using edit_file/write_file — "
        "do not refactor unrelated code. In your final report, name exactly "
        "which file paths you modified and give a one-paragraph rationale. Do "
        "not paste full file contents in your report; the caller reads files "
        "directly from the shared filesystem."
    ),
    "tools": [read_file, write_file, edit_file, grep, glob, ls],
    "model": "anthropic:claude-opus-4-6",
}
```

### 8.3 The `testing` subagent — writes and runs test code

```python
from deepagents.tools import execute

testing_subagent: SubAgent = {
    "name": "testing",
    "description": (
        "Writes and runs test code to validate a fix made by the 'coding' "
        "subagent. Reads the patched file(s), writes appropriate test cases, "
        "executes them, and reports pass/fail with any failure details. Use "
        "this AFTER 'coding' has written a fix, as the final verification step."
    ),
    "system_prompt": (
        "You are a testing specialist. Given a description of a fix that was "
        "just made, read the relevant file(s) with read_file, write test cases "
        "covering the fixed behavior (and, where sensible, the previously "
        "broken behavior as a regression check), and run them with execute. "
        "Your final report must state clearly whether tests passed or failed, "
        "and if failed, the specific failure output — the caller cannot see "
        "your intermediate execute calls, only this final report."
    ),
    "tools": [read_file, write_file, execute, grep, glob, ls],
    "model": "anthropic:claude-sonnet-4-6",
}
```

Note `testing` needs a sandbox-capable `backend` for `execute` to function at all — Chapter 6 covers this; if
your `backend` doesn't support code execution, `execute` errors, which is expected, not a bug.

### 8.4 The coordinator

```python
from deepagents import create_deep_agent

COORDINATOR_PROMPT = """You are a code review coordinator. You never search
code, write fixes, or run tests yourself — you only delegate, using the `task`
tool, to the right specialist for each part of the review:

1. Always delegate to subagent_type="research" first, to gather context about
   the reported issue before anything else happens.
2. Once you have the research summary, delegate to subagent_type="coding" with
   a description that includes the original issue plus the research findings,
   so the fix respects existing conventions.
3. Once the coding subagent reports which files it changed, delegate to
   subagent_type="testing" with a description naming those files, to validate
   the fix.
4. Only after all three steps report back do you summarize the full review
   for the user: what was found, what was changed, and whether tests passed.

Do not skip steps, and do not attempt any of the three specialists' work
directly — your only tool for making progress is `task`."""

code_review_agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    subagents=[research_subagent, coding_subagent, testing_subagent],
    system_prompt=COORDINATOR_PROMPT,
)

result = code_review_agent.invoke({
    "messages": [{
        "role": "user",
        "content": (
            "Review and fix this issue: paginate() in utils/pagination.py "
            "returns one extra item on the last page when total items is an "
            "exact multiple of page_size."
        ),
    }]
})

print(result["messages"][-1].content)
```

Because `general-purpose` isn't among the three declared names, it is still auto-added (Section 6) — the
coordinator has four possible `subagent_type` values available to `task`, though the `COORDINATOR_PROMPT` above
deliberately never mentions it, steering the model toward the three named specialists for this workflow.

Walk the diagram from Section 4.2 mentally against this project: three separate `task` calls, three separate
fresh `subagent_state`s, three separate compiled graphs each running their own internal loop — and the
coordinator's own message history only ever grows by three final reports plus its own final summary, regardless
of how many `grep` calls `research` needed or how many test iterations `testing` ran internally.

---

## Real-World Scenario

A team ports an existing single-prompt "review bot" — one flat agent with `grep`/`read_file`/`write_file`/
`execute` all bound directly, and a system prompt trying to cover research, fixing, and testing conventions all
at once — to the subagent pattern above. Two problems they were fighting disappear immediately, and one new
consideration appears:

**Problem that disappeared — prompt drift.** In the flat version, the single system prompt had grown to over
600 lines trying to cover "how to search," "how to propose safe fixes," and "how to write tests" without the
three concerns bleeding into each other. Every rewrite of the testing guidance had a habit of subtly weakening
the fix-safety guidance nearby. Splitting into three subagent-scoped prompts — none longer than a paragraph or
two — ended the drift entirely, because each prompt now only has to be internally consistent with itself.

**Problem that disappeared — context exhaustion mid-review.** The flat agent frequently ran out of useful
context budget by the testing phase, because dozens of research-phase `grep`/`read_file` calls (and their often
large results) were still sitting in the message history, competing with the test-writing task for attention.
With subagents, the coordinator's context after the `research` call contains exactly one thing from that
phase — the summary — regardless of how many searches it took to produce.

**New consideration — the coding subagent's report discipline.** Early runs of the ported system had the
`coding` subagent paste the full diff into its report "to be helpful." This defeated the isolation benefit
immediately — the coordinator's context ballooned right back up with file contents it didn't need, since the
same content was one `read_file` call away via the shared backend. The fix was exactly the system-prompt
constraint shown in Section 8.2: name the files changed, don't paste their contents.

---

## Best Practices

- **Write subagent `description` fields with the same care as a tool docstring** — it's the parent model's only
  signal for choosing `subagent_type` correctly; vague descriptions produce misrouted `task` calls.
- **Scope `tools` deliberately on every subagent that shouldn't have full access** — omitting `tools` inherits
  the parent's *entire* tool list, which is rarely what you want for a subagent meant to be read-only (like
  `research` in Section 8.1).
- **Instruct write-capable subagents not to paste file contents into their final report** — the shared backend
  already makes that content available on demand; repeating it in the report defeats context isolation.
- **Reserve `CompiledSubAgent` for genuine topology needs**, not as a default — most subagents are well served
  by the plain declarative `SubAgent` shape; reach for a hand-compiled graph only when `SubAgent`'s optional keys
  can't express what you need.
- **Use `AsyncSubAgent` only for work you're comfortable not blocking on synchronously** — it's a different
  execution model (remote, background), not a drop-in alternative to a slow `SubAgent`.
- **Tune `model=` per subagent based on task difficulty, not habit** — a cheap/fast model for mechanical
  search-and-summarize work, your strongest model for correctness-critical work, rather than defaulting every
  subagent to the parent's model.
- **Only disable the auto-added `general-purpose` subagent after you've supplied your own explicit subagents** —
  disabling it with none declared leaves `task` with nothing to dispatch to.
- **Write the coordinator's own `system_prompt` to explicitly name which `subagent_type` handles which part of
  the workflow** (Section 8.4) — don't rely on the model inferring the intended sequencing from the subagents'
  descriptions alone.

---

## Common Mistakes

- **Assuming a subagent's intermediate message history leaks back to the parent.** It does not, by design — only
  the single final report crosses the boundary (Section 4). Debugging "why didn't the parent see the research
  subagent's individual `grep` results" as if something broke is a misunderstanding of the feature, not a bug.
- **Assuming a `task` call spawns a new checkpointed LangGraph thread.** It doesn't (Section 4.3) — it's a
  separate invocation of a separate compiled graph, nested inside the same overall `task` call the parent is
  synchronously waiting on. Durable-execution/crash-recovery semantics (Chapter 10) are a different axis
  entirely.
- **Setting `interrupt_on` on a `SubAgent` dict and expecting it to merge with the parent's `interrupt_on`.** It
  does not merge — a subagent-level `interrupt_on`, if set, **fully overrides** the parent's for that subagent.
  If the parent has approval gates configured and a `SubAgent` sets its own (possibly empty or narrower)
  `interrupt_on`, the parent's gates simply do not apply inside that subagent's invocation unless you've
  explicitly re-declared them there too.
- **Letting a write-capable subagent paste full file contents into its final report** "to be helpful" — this
  re-inflates the parent's context with exactly the content the shared backend already makes available on
  demand, defeating the isolation benefit (Section 5, Real-World Scenario).
- **Omitting `tools` on a subagent that should be read-only or otherwise restricted**, then being surprised it
  wrote a file — omitting `tools` inherits the *parent's entire tool list*, not a safe empty default.
- **Treating `AsyncSubAgent` as a "slower `SubAgent`."** It is routed through an entirely separate
  `AsyncSubAgentMiddleware` with its own launch/check/update/cancel/list tools, not through `task` at all —
  code written assuming a uniform dispatch path across all three shapes will not work for `AsyncSubAgent`.

---

## Summary

- A flat single-agent design degrades along two independent axes as concerns pile up: the system prompt becomes
  unwieldy holding every concern's instructions active at once, and every intermediate tool call from every
  concern shares one context budget. Subagents fix both by giving each concern its own focused prompt and its
  own private context window.
- `create_deep_agent(subagents=[...])` mixes three shapes: `SubAgent` (declarative, the common case),
  `CompiledSubAgent` (bring-your-own compiled graph, for custom topology), and `AsyncSubAgent` (remote,
  background, dispatched through a separate `AsyncSubAgentMiddleware`, not `task`).
- The `task` tool (`description`, `subagent_type`) is the *only* dispatch mechanism for `SubAgent`/
  `CompiledSubAgent` entries — the parent picks `subagent_type` by reading each subagent's `description`, the
  same way it picks any other tool by docstring.
- Context isolation is concrete and mechanical: a fresh `subagent_state` is built (after stripping
  `private_state_keys`), a separate compiled graph runs its own full internal loop, and only the extracted final
  result returns via a `Command` update — none of the internal tool calls or reasoning reach the parent.
- This is not a new checkpointed thread — it's a nested invocation of a separate compiled graph within the same
  overall `task` call the parent is synchronously waiting on.
- Subagents get their own middleware stack but share the parent's underlying `backend` — so a subagent's
  `write_file` is visible to the parent's later `read_file`, entirely outside the message-history channel.
- If no subagent named `"general-purpose"` is supplied, one is auto-added with full parent tool access; disable
  it via a `HarnessProfile` only once you've supplied your own explicit subagents.
- Per-subagent `model=` overrides let you tune cost/latency per sub-task — cheap models for mechanical work,
  strong models for correctness-critical work — independently of the parent's model choice.

---

## Knowledge Check

1. Describe, mechanically, what happens to a subagent's intermediate tool calls and reasoning between the moment
   it starts running and the moment the parent sees a result. What specifically crosses the parent/subagent
   boundary, and what doesn't?
2. Why is it wrong to describe a `task` call as "spawning a new checkpointed thread"? What is it more precisely,
   and how does that distinction matter for how you'd reason about Chapter 10's durable-execution model?
3. A `SubAgent` dict sets its own `interrupt_on`. The parent deep agent also has `interrupt_on` configured. What
   happens when this subagent runs — do the two merge, and if not, which one applies?
4. A subagent writes a file via `write_file` but its final report never mentions the file's contents. Explain
   precisely how the parent can still access that content, and why this doesn't contradict context isolation.
5. You have three subagents: one doing simple keyword search and summarization, one doing correctness-critical
   code generation, and one doing exploratory design brainstorming. How would you assign `model=` across the
   three, and why?
6. Give one concrete scenario where `CompiledSubAgent` is the right choice over a plain `SubAgent` dict, and one
   concrete scenario where `AsyncSubAgent` is the right choice over both.

---

## Hands-On Exercise

Extend the Code Review Agent from Section 8 with a **fourth subagent**, and verify context isolation empirically
rather than just trusting the explanation.

1. **Add a `"docs"` subagent** whose job is to update a `CHANGELOG.md`-style file after `testing` confirms a fix
   passes. Give it a focused `description` and `system_prompt`, and scope its `tools` to `[read_file,
   write_file, ls]` — it doesn't need `grep`/`glob`/`execute`.

2. **Update the coordinator's `system_prompt`** to add a fourth delegation step: after `testing` reports success,
   delegate to `subagent_type="docs"` with a description of what was fixed, so it can append a changelog entry.

3. **Add the new subagent to `subagents=[...]`** alongside the other three, and re-run the Section 8.4 example
   end to end with a request that should trigger all four steps.

4. **Verify context isolation directly, not just by assumption.** Inspect `result["messages"]` after the run
   (or, better, stream with `stream_mode="values"` and log each state's `messages` length at every step) and
   confirm:
   - The parent's message list contains exactly four `task`-related exchanges (one per subagent call) plus the
     coordinator's own final summary — not dozens of entries reflecting every internal `grep`/`read_file`/
     `write_file`/`execute` call any subagent made internally.
   - None of the `docs` subagent's own internal tool calls (e.g. its `read_file` calls to check what changed)
     appear anywhere in the parent's `messages` list — only its final report text does.

5. **Bonus:** Temporarily add logging/print statements inside one subagent's own invocation (e.g. by wrapping a
   tool it uses) to confirm, side by side, that the tool actually executed multiple times internally while the
   parent's message list shows only one net addition for that entire `task` call — this is the most convincing
   direct evidence of Section 4's claims, rather than taking the mechanism on faith.

---

## Further Reading

- [DeepAgents Overview (LangChain Docs)](https://docs.langchain.com/oss/python/deepagents/overview) — official
  overview of `create_deep_agent()` and its defaults, including subagent configuration
- [`langchain-ai/deepagents` GitHub repository](https://github.com/langchain-ai/deepagents) — read
  `libs/deepagents/deepagents/middleware/subagents.py` and `middleware/async_subagents.py` directly for the
  exact, current `SubAgent`/`CompiledSubAgent`/`AsyncSubAgent` definitions and the `task` tool's construction
- Related chapter in this course: [Chapter 6 — Backends & Storage Architecture](./06-backends-and-storage-architecture.md)
  — the shared-`backend` mechanics Section 5 of this chapter depends on
- Related chapter in this course: [Chapter 10 — Checkpointing & Durable Execution](./10-checkpointing-and-durable-execution.md)
  — the actual checkpointed-thread model this chapter's Section 4.3 explicitly contrasts against
- Related chapter in this course: [Chapter 12 — Multi-Agent Systems](./12-multi-agent-systems.md) — coordinator
  design patterns and sync-vs-async subagent architecture at greater depth than this chapter's single project
- Related chapter in this course: [Chapter 9 — Human-in-the-Loop](./09-human-in-the-loop.md) — the full
  `interrupt_on`/`FilesystemPermission` mechanics this chapter's Common Mistakes section only touches on

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./07-memory-and-persistence.md">← Previous: Memory & Persistence</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./09-human-in-the-loop.md">Next: Human-in-the-Loop →</a>
</div>
