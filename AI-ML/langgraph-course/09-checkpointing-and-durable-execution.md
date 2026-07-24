# Chapter 9: Checkpointing & Durable Execution

> "The question is not whether your service will crash. It's whether your workflow notices."

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain why long-running agentic workflows need durable execution, and what happens to an uncheckpointed graph when the process hosting it dies mid-run
- Choose the right checkpointer — `MemorySaver`, `SqliteSaver`, or `PostgresSaver` — for a given environment (unit tests, a local CLI tool, a multi-instance production service)
- Wire a checkpointer into a graph at compile time and scope every invocation to a `thread_id`
- Trace the full durable execution flow: super-step execution → checkpoint write → crash → reload → resume from the last checkpoint
- Use `get_state()` and `update_state()` to inspect and manually correct checkpointed state, as an admin/debugging operator would
- Use `get_state_history()` to "time travel" — replay execution from an earlier checkpoint to explore an alternate path or recover from a bad state
- Recognize where checkpointing sits underneath human-in-the-loop `interrupt()` calls (full treatment in Chapter 12)

---

## Prerequisites for the Chapter

This chapter builds on the execution model from **Chapter 7 (Compilation & Execution)**, where you learned that `graph.compile()` produces a runnable `Pregel`-style graph that executes in **super-steps** — each super-step runs one wave of ready nodes to completion, applies their state updates through reducers (Chapter 6), and determines the next wave. Everything in this chapter hangs off one fact you already know: **the graph pauses between super-steps to figure out what runs next.** That pause is exactly where a checkpointer gets to act.

You should also be comfortable with:

- `StateGraph`, `TypedDict`/`dataclass`/Pydantic state schemas, and reducers via `Annotated[...]` (Chapters 2 and 6)
- Node functions that return partial state updates (Chapter 3)
- `graph.invoke()` and `graph.stream()` (Chapter 7)
- Basic familiarity with SQLite and PostgreSQL connection strings (helpful, not required — we'll show the exact calls)

No new Python packages are required to *read* this chapter. If you want to run the SQLite examples yourself later, you'll need `langgraph-checkpoint-sqlite`; for Postgres, `langgraph-checkpoint-postgres` and a reachable Postgres instance. Per this course's ground rules, none of the code below has been executed — read it as a precise reference, not a copy-paste guarantee, and consult the installed version's docs for exact signatures before shipping it.

---

## 1. Why Durable Execution Matters

### 1.1 The problem: long-running agentic workflows are fragile by default

A simple LCEL chain runs in milliseconds to a few seconds. An agentic LangGraph workflow is a different animal: a research assistant that plans, searches, reads, and synthesizes across five or six LLM calls can easily run for two or three minutes. A multi-step approval workflow can span *days* — waiting on a human to click "approve" in some other system entirely. The moment your graph's lifetime is measured in minutes-to-days instead of milliseconds, a new class of failure becomes not just possible but *routine*:

- The container running your FastAPI process gets redeployed mid-request because a new version shipped.
- The pod is rescheduled by Kubernetes because a node became unhealthy.
- An unhandled exception three nodes into a five-node graph crashes the worker process.
- The process is simply restarted for a routine security patch.
- A human reviewer in a human-in-the-loop workflow doesn't respond for six hours, and in that window the service that was holding their session in memory gets recycled.

Without any durability mechanism, every one of these events has the same consequence: **all progress since the last response returned to the caller is gone.** If your research assistant had completed four of five research steps and the fifth one crashed the process, restarting means going back to step one — burning the LLM tokens, tool calls, and wall-clock time you already spent, and potentially producing a different (non-reproducible) result the second time around because LLM calls aren't deterministic.

This is qualitatively different from a stateless web request failing. A stateless request failing means the client retries and gets a fresh, cheap attempt. An agentic workflow failing mid-way means you throw away real, expensive, partially-completed work.

### 1.2 What "durable execution" means in LangGraph

LangGraph's answer is **checkpointing**: after every super-step, the graph's *entire state* is persisted to durable storage, tagged with an identifier for "which run this belongs to." If the process dies, a new process — anywhere, at any point in the future — can load that same identifier and continue exactly where execution left off, rather than starting over.

This reframes the mental model from Chapter 7 in one important way. You already know that `.invoke()` runs the graph until it reaches `END` or a pause point. With a checkpointer attached, you should think of `.invoke()` as doing something closer to:

1. Load the most recent checkpoint for this run (if one exists).
2. Resume super-step execution from there.
3. After each super-step, write a new checkpoint.
4. Repeat until `END` (or an `interrupt()`, covered in Chapter 12).

Nothing about how you write nodes changes. Durability is a property you turn on for the *graph*, at compile time — not something you hand-roll inside every node with `try/except` and manual state serialization.

---

## 2. The Checkpointer Abstraction

### 2.1 What gets checkpointed, and when (the super-step boundary)

A **checkpoint** is a serialized snapshot of the graph's channel values (its state) at a specific point in execution, plus metadata: which node(s) produced it, which step number it corresponds to, and a pointer to the checkpoint that came immediately before it (its "parent"). Checkpoints form a linked list — really a tree, once you consider time travel in Section 6 — per **thread** (Section 3).

The write happens **after every super-step**, not after every individual node. If a super-step runs three nodes concurrently (a fan-out from Chapter 6's reducer discussion), all three nodes' updates are merged through their reducers first, and the *resulting* combined state is what gets checkpointed — one checkpoint per super-step, not per node.

This has a direct consequence for resumability: if a node crashes partway through its own body (say, an unhandled `requests.exceptions.Timeout` inside a tool call), there is no partial checkpoint for that node's incomplete work. The graph resumes from the *last completed super-step* and re-runs the failed super-step's nodes from scratch. We'll return to why this makes node idempotency important in Section 8 (Best Practices).

### 2.2 `MemorySaver` — in-process, non-persistent

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)
```

`MemorySaver` stores every checkpoint in a plain Python dictionary held in process memory. It is the fastest checkpointer by far (no serialization to disk, no network round trip) and requires zero setup, which makes it the correct default for:

- Unit tests and integration tests (Chapter 17) where you want deterministic, fast checkpoint behavior without touching the filesystem
- Local development and notebooks, where you're iterating on graph logic and don't need state to survive a kernel restart
- Quick demos where "does resuming even work" matters more than "does it survive a real crash"

Its defining limitation is right there in the name: it lives in one process's memory. Kill the process — the exact failure mode this whole chapter exists to protect against — and every checkpoint it held is gone. `MemorySaver` durability is a contradiction in terms; it demonstrates the *mechanics* of checkpointing without providing the *guarantee* checkpointing exists to give you. Never use it for anything a user's actual work depends on.

### 2.3 `SqliteSaver` — single-file, persistent, single-instance

```python
from langgraph.checkpoint.sqlite import SqliteSaver

with SqliteSaver.from_conn_string("checkpoints.db") as checkpointer:
    graph = builder.compile(checkpointer=checkpointer)
    result = graph.invoke(initial_state, config={"configurable": {"thread_id": "session-42"}})
```

`SqliteSaver` writes checkpoints to a single `.db` file on disk using Python's standard `sqlite3` module under the hood. `from_conn_string` is a context manager — it opens the connection, hands you a ready-to-use saver, and closes the connection cleanly on exit. This is the right choice when:

- You're building a local CLI tool, a desktop app, or a single-process background worker where "production" means "runs reliably on one machine"
- You want real crash survival (kill `-9` the process, restart it, state is still on disk) without standing up a database server
- Your team is prototyping a service before deciding on production infrastructure

The load-bearing limitation: SQLite serializes writes at the file level, so it is **not safe for multiple independent process instances writing concurrently** — the exact topology of a horizontally-scaled FastAPI deployment behind a load balancer. If two instances of your app both try to advance the same thread at the same time, you'll hit lock contention or corruption risk, not graceful concurrent access. Treat `SqliteSaver` as strictly single-instance. There is also an async counterpart, `AsyncSqliteSaver` (`langgraph.checkpoint.sqlite.aio`), for use inside `async def` nodes and `ainvoke`/`astream` call sites.

### 2.4 `PostgresSaver` — production, multi-instance safe

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://langgraph_user:secret@db.internal:5432/agent_checkpoints"

with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()  # one-time: creates the checkpoint tables if they don't exist
    graph = builder.compile(checkpointer=checkpointer)
```

`PostgresSaver` is the checkpointer for real production deployments: a proper database server handles concurrent writes from many application instances correctly, gives you the operational tooling (backups, replicas, monitoring) your infra team already runs for everything else, and scales independently of any single app process. `checkpointer.setup()` is a one-time migration step — it creates the tables LangGraph needs (checkpoints, checkpoint writes, blobs) if they aren't already present; run it once during deployment/bootstrap, not on every request. As with SQLite, there's an async variant, `AsyncPostgresSaver`, for fully async services — and it's the natural pairing for a FastAPI app that's already `async def` all the way down.

The trade-off is operational weight: you now depend on a Postgres instance being reachable, correctly provisioned, and backed up. That's the correct trade for production; it's overkill for a unit test.

| Checkpointer | Persistence | Multi-instance safe | Setup cost | Use for |
|---|---|---|---|---|
| `MemorySaver` | None (process memory) | No | None | Tests, notebooks, throwaway demos |
| `SqliteSaver` | Single file on disk | No | Minimal (a file path) | Local apps, single-instance services, prototypes |
| `PostgresSaver` | Database server | Yes | A running Postgres + `.setup()` | Production, horizontally-scaled services |

### 2.5 Compile-time injection

In every case, the pattern is identical: a checkpointer is a plug-in you hand to `.compile()`, and the compiled graph gains persistence behavior without any change to node code:

```python
builder = StateGraph(ResearchState)
builder.add_node("plan", plan_research)
builder.add_node("gather", gather_sources)
builder.add_edge(START, "plan")
builder.add_edge("plan", "gather")
builder.add_edge("gather", END)

# Same graph definition, three different durability profiles:
dev_graph = builder.compile(checkpointer=MemorySaver())
local_graph = builder.compile(checkpointer=sqlite_checkpointer)
prod_graph = builder.compile(checkpointer=postgres_checkpointer)
```

This is a deliberate design choice: durability is an *infrastructure* decision, orthogonal to the *graph logic* decision. You can develop and test your entire workflow against `MemorySaver`, then swap in `PostgresSaver` for the production build with a one-line change at the composition root — nothing about `plan_research` or `gather_sources` needs to know or care which checkpointer is behind them.

---

## 3. `thread_id`: Scoping State to a Conversation

### 3.1 The config dict

A checkpointer needs to know *which* run a checkpoint belongs to — otherwise two unrelated users' in-progress workflows would collide into a single blob of state. That scoping key is `thread_id`, passed through the `config` argument every time you call the graph:

```python
config = {"configurable": {"thread_id": "user-8231-session-1"}}

result = graph.invoke({"topic": "impact of interest rates on SaaS valuations"}, config=config)
```

Every checkpoint written during this call is tagged with `thread_id = "user-8231-session-1"`. Call `.invoke()` again later with that *same* `config`, and the checkpointer loads that thread's latest checkpoint before doing anything else — that's the entire mechanism behind both "resume this conversation" (Chapter 10, Memory Management) and "resume this crashed workflow" (this chapter). They're the same primitive used for two different purposes.

Pick `thread_id` values that map to whatever unit of durability makes sense for your product: a chat session, a support ticket ID, a document-review workflow ID, a batch job ID. It's just a string — you own the namespacing scheme.

### 3.2 What happens without a `thread_id`

If you compile a graph with a checkpointer but call `.invoke()` without a `config["configurable"]["thread_id"]`, LangGraph raises an error — a checkpointer with no thread to scope to has nothing to key its writes on. This is a common first-run mistake (Section 10) precisely because ungated graphs (no checkpointer at all) don't require `config` for anything, so it's easy to forget the requirement flips on the moment you add persistence.

### 3.3 One thread, many checkpoints

A single `thread_id` doesn't correspond to one checkpoint — it corresponds to a **history** of checkpoints, one per super-step, going back to the very first invocation on that thread. Section 6 (time travel) is entirely about walking back through that history. For now, the mental model is: `thread_id` names a lineage; each super-step appends one more checkpoint to it.

---

## 4. The Durable Execution Flow: Crash and Resume

Here is the sequence that makes durable execution real, spelled out step by step:

1. **Invoke.** A caller starts a run: `graph.invoke(initial_input, config={"configurable": {"thread_id": "job-77"}})`.
2. **Execute a super-step.** The graph runs the next wave of ready nodes.
3. **Checkpoint.** Immediately after the super-step completes, the checkpointer persists the resulting merged state, tagged with `thread_id = "job-77"` and a new `checkpoint_id`, with a `parent_config` pointer back to the previous checkpoint.
4. **Repeat 2–3** for each subsequent super-step.
5. **Crash.** At some point — say, right after step 3 of 5 has been checkpointed but before step 4 starts — the process is killed: a deploy, an OOM kill, a crash in an unrelated part of the app, doesn't matter which.
6. **Reload.** A new process starts. It rebuilds the *same graph definition* (same nodes, same edges) and compiles it with a checkpointer pointed at the *same durable store* (the same SQLite file or the same Postgres database) — this is why the checkpointer must be backed by storage outside the process for this step to work at all.
7. **Resume.** Something calls `graph.invoke(None, config={"configurable": {"thread_id": "job-77"}})` — note `None` as the input, not the original input again. The checkpointer loads the latest checkpoint for `thread_id="job-77"` (the one from the end of step 3), and the graph resumes super-step execution from there — running step 4, then step 5, exactly as if the crash never happened.
8. **Complete.** The graph reaches `END`. The caller gets a result that reflects *all five* steps' work, even though steps 1–3 and steps 4–5 happened in two entirely different OS processes, possibly minutes or hours apart, possibly on two different machines.

The critical detail in step 7 is passing `None` (or omitting new input) rather than re-supplying the original input. Passing the original input again would be interpreted as *starting a new invocation on top of the existing state* — for graphs with reducers that accumulate (like the running list of research notes in Section 7's example), this can double-apply data. Resuming after a crash means "continue this thread with no new input," which is exactly what `None` signals.

```mermaid
sequenceDiagram
    participant Caller
    participant Process A (pre-crash)
    participant Checkpointer (Postgres)
    participant Process B (post-crash)

    Caller->>Process A (pre-crash): invoke(input, thread_id="job-77")
    Process A (pre-crash)->>Process A (pre-crash): run super-step 1
    Process A (pre-crash)->>Checkpointer (Postgres): write checkpoint 1
    Process A (pre-crash)->>Process A (pre-crash): run super-step 2
    Process A (pre-crash)->>Checkpointer (Postgres): write checkpoint 2
    Process A (pre-crash)->>Process A (pre-crash): run super-step 3
    Process A (pre-crash)->>Checkpointer (Postgres): write checkpoint 3
    Note over Process A (pre-crash): 💥 process killed (deploy/OOM/crash)
    Note over Process B (post-crash): new process starts, same DB URI
    Caller->>Process B (post-crash): invoke(None, thread_id="job-77")
    Process B (post-crash)->>Checkpointer (Postgres): load latest checkpoint (checkpoint 3)
    Process B (post-crash)->>Process B (post-crash): run super-step 4
    Process B (post-crash)->>Checkpointer (Postgres): write checkpoint 4
    Process B (post-crash)->>Process B (post-crash): run super-step 5
    Process B (post-crash)->>Checkpointer (Postgres): write checkpoint 5
    Process B (post-crash)-->>Caller: final result (all 5 steps reflected)
```

---

## 5. Inspecting and Editing State: `get_state` and `update_state`

Checkpointing isn't only a crash-recovery mechanism — once state is externalized to durable storage, you get a second capability for free: you can look at it, and change it, without running the graph at all.

### 5.1 `get_state(config)`

```python
config = {"configurable": {"thread_id": "job-77"}}
snapshot = graph.get_state(config)

snapshot.values          # the current state dict, e.g. {"topic": "...", "research_notes": [...], ...}
snapshot.next             # tuple of node names that would run next, e.g. ("synthesize_draft",); empty if done
snapshot.config           # config dict including this exact checkpoint_id
snapshot.metadata         # dict with "source", "step", "writes", etc.
snapshot.created_at       # ISO timestamp of this checkpoint
snapshot.parent_config    # config pointing at the previous checkpoint in this thread, or None if it's the first
```

`get_state` returns a `StateSnapshot` — a read-only view of exactly what's stored for that thread's latest checkpoint (or a specific one, if you pass a config carrying a specific `checkpoint_id`). `snapshot.next` is particularly useful operationally: an empty tuple means the thread has run to completion; a non-empty tuple tells you precisely which node(s) are queued up next — invaluable for an admin dashboard showing "where is job 77 stuck?"

### 5.2 `update_state(config, values, as_node=None)`

```python
graph.update_state(
    config,
    {"research_notes": ["Manually added: analyst consensus revised down 15bps on 2026-07-20"]},
    as_node="gather_sources",
)
```

`update_state` writes a *new* checkpoint on top of the thread's current state, applying `values` through the same reducers the graph would use if a node had returned that update — without actually running any node. The `as_node` argument tells LangGraph which node's "position" this update should be attributed to, which matters for figuring out what runs next (the graph's edges route based on which node just "produced" a checkpoint).

This is the mechanism that makes several operator workflows possible:

- **Admin correction**: an operator notices a research assistant pulled a stale figure into `research_notes` and patches it in directly, without re-running the (expensive) gathering step.
- **Debugging**: while diagnosing a bug, you manually set a field to a known value and resume, to isolate whether a downstream node is misbehaving independent of what upstream nodes produced.
- **Human correction inside human-in-the-loop flows**: this is precisely the primitive Chapter 12's `interrupt()`/`Command(resume=...)` pattern is built on — a human reviews paused state and the system calls `update_state` (directly or via the resume helpers) to inject the human's decision before execution continues.

### 5.3 A note on safety

`update_state` bypasses whatever validation or business logic a node would normally apply to its own output — it writes directly. Treat it like a direct database write (because that's what it is): appropriate for trusted, structured admin tooling; not a substitute for input validation on user-facing paths.

---

## 6. Time Travel: Replaying From an Earlier Checkpoint

### 6.1 Walking the history with `get_state_history`

Because a thread's checkpoints form a chain (each pointing to its parent), you can enumerate the entire history of a run:

```python
config = {"configurable": {"thread_id": "job-77"}}

for snap in graph.get_state_history(config):
    print(snap.config["configurable"]["checkpoint_id"], snap.metadata.get("step"), snap.next)
```

`get_state_history` yields `StateSnapshot`s newest-first. Each one is a fully addressable point in the thread's past — you can pass any of their `config` values back into `get_state` to inspect that exact moment, or into `invoke` to resume *from* that moment.

### 6.2 Forking execution from an earlier checkpoint

"Time travel" means taking one of those earlier configs and continuing execution from it — not from the thread's latest checkpoint, but from wherever you point:

```python
history = list(graph.get_state_history(config))

# Suppose history[2] is the checkpoint right after "gather_sources" completed,
# before "analyze_findings" ran with what turned out to be a bad synthesis prompt.
earlier_config = history[2].config

# Resume forward from that point. This does NOT overwrite the original thread's
# later checkpoints -- it creates a new branch whose parent is `earlier_config`.
graph.invoke(None, config=earlier_config)
```

The key behavioral detail: this does not destroy or rewrite history. The graph continues forward from the chosen checkpoint, producing a *new* sequence of checkpoints whose `parent_config` chains back to `earlier_config` — a fork, not an overwrite. The original branch (the one that led to the bad synthesis) is still there in history if you need to compare the two. This is why the "tree," not "list," framing from Section 2.1 matters once you start forking.

### 6.3 Why this is useful in practice

- **Exploring alternate paths**: re-run the last two steps of a workflow with a different prompt or a different tool configuration, without re-paying for the (identical, expensive) earlier steps.
- **Recovering from a bad state**: if a node's output was corrupted by a flaky upstream API, roll back to the last-known-good checkpoint and re-run forward instead of restarting the entire workflow from scratch.
- **Auditing and debugging in production**: reconstruct exactly what the graph believed at each point in a run that produced a wrong answer, which is often the fastest way to find *which* node introduced the error.

---

## 7. Checkpointing's Relationship to Human-in-the-Loop

Chapter 12 covers `interrupt()` in full, but it's worth previewing the dependency now because it clarifies *why* this chapter comes before it in the course: **an `interrupt()` call is only possible because the graph is already checkpointed.** When a node calls `interrupt()`, LangGraph checkpoints the state at that exact point and suspends execution — conceptually the same "pause after a super-step" behavior you've seen throughout this chapter, just triggered explicitly by node code instead of happening between every super-step. Resuming later with `Command(resume=<value>)` is mechanically the same operation as the crash-recovery resume in Section 4: load the checkpoint for this `thread_id`, and continue forward. A human clicking "approve" three days later and a server crash-and-restart three seconds later are, to the checkpointer, the same event: *resume this thread from its last checkpoint.* Everything you just learned about `get_state`, `update_state`, and time travel applies directly to inspecting and steering a workflow that's paused on a human decision.

---

## Examples

### Worked Example: The Long-Running Research Assistant

We'll build a graph that performs multi-step research on a topic — plan → gather → analyze → synthesize → finalize — checkpointing after every step, and walk through what happens when the process crashes partway through.

#### State schema

```python
from typing import TypedDict, Annotated
import operator

class ResearchState(TypedDict):
    topic: str
    plan: str
    research_notes: Annotated[list[str], operator.add]  # accumulates across steps
    draft: str
    final_report: str
```

`research_notes` uses the `operator.add` reducer (Chapter 6) so that each research-related node can *append* findings rather than overwrite them — important because if a crash forces a super-step to re-run, we want to reason clearly about what accumulates versus what's idempotently overwritten.

#### Nodes

```python
def plan_research(state: ResearchState) -> dict:
    # In production this would call an LLM to break the topic into sub-questions.
    plan = f"Plan for '{state['topic']}': (1) market size (2) competitors (3) regulatory risk"
    print("[plan_research] done")
    return {"plan": plan}

def gather_sources(state: ResearchState) -> dict:
    # In production: web search / retriever calls (Chapter 8's tool-calling patterns).
    print("[gather_sources] done")
    return {"research_notes": ["Source A: TAM estimated at $4.2B, growing 18% YoY."]}

def analyze_findings(state: ResearchState) -> dict:
    print("[analyze_findings] done")
    return {"research_notes": ["Analysis: growth rate outpaces two of three named competitors."]}

def synthesize_draft(state: ResearchState) -> dict:
    notes = "\n".join(state["research_notes"])
    draft = f"Draft report on {state['topic']}:\n{notes}"
    print("[synthesize_draft] done")
    return {"draft": draft}

def finalize_report(state: ResearchState) -> dict:
    print("[finalize_report] done")
    return {"final_report": state["draft"] + "\n\n-- End of Report --"}
```

#### Graph assembly with a persistent checkpointer

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.sqlite import SqliteSaver

builder = StateGraph(ResearchState)
builder.add_node("plan_research", plan_research)
builder.add_node("gather_sources", gather_sources)
builder.add_node("analyze_findings", analyze_findings)
builder.add_node("synthesize_draft", synthesize_draft)
builder.add_node("finalize_report", finalize_report)

builder.add_edge(START, "plan_research")
builder.add_edge("plan_research", "gather_sources")
builder.add_edge("gather_sources", "analyze_findings")
builder.add_edge("analyze_findings", "synthesize_draft")
builder.add_edge("synthesize_draft", "finalize_report")
builder.add_edge("finalize_report", END)

with SqliteSaver.from_conn_string("research_jobs.db") as checkpointer:
    graph = builder.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "research-2026-07-24-001"}}
    initial_state = {"topic": "impact of on-device LLMs on cloud inference spend"}

    # --- Simulate a crash after 2 of 5 steps ---
    # In a real crash, the process would simply die here. We simulate it by
    # only streaming through the first two super-steps and then stopping,
    # never reaching finalize_report or even synthesize_draft/analyze_findings.
    for step in graph.stream(initial_state, config=config, stream_mode="updates"):
        print("super-step result:", step)
        if "gather_sources" in step:
            print(">>> Simulating a crash right after gather_sources checkpointed <<<")
            break
```

At this point, exactly two checkpoints exist for `thread_id="research-2026-07-24-001"`: one after `plan_research`, one after `gather_sources`. `analyze_findings`, `synthesize_draft`, and `finalize_report` have not run.

#### Resuming after the "crash"

A fresh process (or, here, a fresh call reusing the same on-disk file) reconnects to the same SQLite file and resumes:

```python
with SqliteSaver.from_conn_string("research_jobs.db") as checkpointer:
    graph = builder.compile(checkpointer=checkpointer)  # same builder, rebuilt fresh
    config = {"configurable": {"thread_id": "research-2026-07-24-001"}}

    # Confirm where we left off before resuming.
    snapshot = graph.get_state(config)
    print("Resuming; next node(s):", snapshot.next)   # -> ("analyze_findings",)
    print("State so far:", snapshot.values)

    # Resume with None input -- continue this thread, don't restart it.
    final_state = graph.invoke(None, config=config)
    print(final_state["final_report"])
```

`plan_research` and `gather_sources` do **not** re-run. Execution picks up at `analyze_findings`, runs `synthesize_draft`, then `finalize_report`, and `final_state["final_report"]` reflects the complete pipeline — even though the two halves of the run happened in two separate `with` blocks (standing in for two separate process lifetimes).

#### Peeking at history and correcting a mistake

```python
history = list(graph.get_state_history(config))
for snap in history:
    print(snap.metadata.get("step"), snap.next, list(snap.values.get("research_notes", [])))

# Suppose we realize the "gather_sources" note had a stale TAM figure.
# We can patch state directly rather than re-running the (slow, costly) tool call:
graph.update_state(
    config,
    {"research_notes": ["Correction: TAM revised to $4.6B per Q2 filing."]},
    as_node="gather_sources",
)
```

---

## Diagrams

Graph structure for the Research Assistant, annotated with where checkpoints are written:

```mermaid
flowchart LR
    START((START)) --> A[plan_research]
    A -->|checkpoint 1| B[gather_sources]
    B -->|checkpoint 2| C[analyze_findings]
    C -->|checkpoint 3| D[synthesize_draft]
    D -->|checkpoint 4| E[finalize_report]
    E -->|checkpoint 5| END((END))

    style A fill:#e1f5ff
    style B fill:#e1f5ff
    style C fill:#fff3e0
    style D fill:#fff3e0
    style E fill:#f3e5f5
```

Time travel as a branching tree of checkpoints — the original run hit a bad `analyze_findings` result; a new branch is forked from the checkpoint right after `gather_sources`:

```mermaid
flowchart TB
    C1["checkpoint 1<br/>(after plan_research)"] --> C2["checkpoint 2<br/>(after gather_sources)"]
    C2 --> C3A["checkpoint 3 — original branch<br/>(after analyze_findings, BAD result)"]
    C3A --> C4A["checkpoint 4 — original branch<br/>(after synthesize_draft)"]
    C2 -.fork from here.-> C3B["checkpoint 3' — new branch<br/>(after analyze_findings, re-run with fixed prompt)"]
    C3B --> C4B["checkpoint 4'<br/>(after synthesize_draft)"]
    C4B --> C5B["checkpoint 5'<br/>(after finalize_report)"]

    style C3A fill:#ffd6d6
    style C3B fill:#d6ffd6
    style C4B fill:#d6ffd6
    style C5B fill:#d6ffd6
```

---

## Real-World Scenarios

**Scenario 1 — The mid-deploy research job.** An internal analytics team runs a LangGraph-based research assistant behind a FastAPI endpoint, checkpointed to `PostgresSaver`. A routine deploy rolls out while three users' research jobs are mid-flight (each 4–6 steps, each step involving a 20–40 second LLM call). Without checkpointing, all three jobs would silently vanish and each user would need to restart from scratch, unaware anything went wrong until they refreshed and saw nothing. With `PostgresSaver`, the new pods that come up after the deploy simply keep serving `invoke(None, config)` calls against the same `thread_id`s — each job resumes exactly where it was, and no user notices the deploy happened at all.

**Scenario 2 — The multi-day approval workflow.** A procurement-approval graph pauses (via `interrupt()`, Chapter 12) waiting for a manager's sign-off, which can take anywhere from minutes to several business days. The service hosting the graph is redeployed, autoscaled down to zero overnight, and back up multiple times during that window. None of that matters: the paused state lives in `PostgresSaver`, keyed by the approval request's `thread_id`, entirely independent of which process (or how many processes) are currently running. When the manager finally responds, whichever instance handles that request loads the checkpoint and resumes.

**Scenario 3 — Debugging a bad output with time travel.** A user reports that a report-generation graph produced a nonsensical financial figure. An engineer pulls up `get_state_history` for that `thread_id`, inspects each checkpoint's `research_notes`, and finds the exact super-step where a tool call returned a malformed number that then propagated forward unchecked. Rather than trying to reproduce the bug from scratch (potentially non-reproducible, since the LLM calls aren't deterministic), the engineer forks execution from the checkpoint right before the bad tool result, patches in a corrected value via `update_state`, and resumes forward to confirm the rest of the pipeline behaves correctly downstream of the fix.

---

## Best Practices

- **Match the checkpointer to the environment, not to convenience.** `MemorySaver` in a production path is a silent landmine — it "works" in every demo and then loses everything on the first real restart. Default to `SqliteSaver` for anything single-instance and durable, `PostgresSaver` for anything multi-instance.
- **Always pass a `thread_id`, and choose it deliberately.** Tie it to a stable business identifier (session ID, ticket ID, job ID) that you can look up later — not a random UUID you'll never be able to correlate back to "which user's workflow was this."
- **Resume with `None`, not the original input.** Re-supplying the original input on resume risks double-applying reducer-based updates (e.g., appending research notes twice) and, for graphs with side effects gated on input, can re-trigger logic that should only run once per thread.
- **Design nodes to be idempotent where practical.** Because a crash mid-super-step causes that entire super-step's nodes to re-run from their last checkpointed input, a node that calls a non-idempotent external API (e.g., "charge the customer," "send the email") should guard against duplicate execution — check-then-act against an external system of record, or use idempotency keys on the downstream call itself.
- **Run `checkpointer.setup()` once, during deployment/migration, not on the request path.** It's a schema-creation step, not a per-request operation.
- **Treat `update_state` like a direct database write.** Reserve it for trusted admin tooling and the human-in-the-loop resume path (Chapter 12); don't expose it as a general-purpose "edit my session" API without validation.
- **Prune or archive old threads.** Every super-step of every thread accumulates a new checkpoint row; a long-lived production system needs a retention policy (TTL, archival job) so your checkpoint table doesn't grow unbounded.
- **Test crash-and-resume explicitly**, not just the happy path. In Chapter 17 (Testing), simulate a crash by stopping execution partway through a `stream()` loop, then asserting that a fresh `invoke(None, config)` produces the correct final state — this is the single highest-value test a durable-execution system can have.

---

## Common Mistakes

- **Using `MemorySaver` in production "because it was already there from development."** It silently loses all state on every restart with no warning — the exact failure mode this chapter is about avoiding.
- **Forgetting `thread_id` entirely once a checkpointer is attached.** Calling `.invoke()` without `config={"configurable": {"thread_id": ...}}` on a checkpointed graph raises an error; the fix is trivial but the mistake is common when copy-pasting from an earlier, uncheckpointed version of the same graph.
- **Reusing the same `thread_id` across unrelated conversations/jobs.** This silently merges two users' or two jobs' state into one lineage — data leaks across sessions rather than a clean error, which makes it dangerous.
- **Re-invoking with the original input instead of `None` on resume.** For accumulating reducers (`operator.add`, custom list-append reducers), this duplicates data; for gated one-time side effects, it can re-trigger them.
- **Assuming SQLite is "good enough" for a multi-instance deployment.** It works fine in every load test that only ever hits one instance, then produces intermittent lock errors or state inconsistency the moment you scale horizontally — often discovered in production, not staging.
- **Never calling `checkpointer.setup()` on a fresh Postgres database**, then being confused by a "relation does not exist" error on first use.
- **Treating checkpointed state as a substitute for node idempotency.** Checkpointing guarantees you resume from the *last completed super-step* — it does not guarantee the super-step that was interrupted didn't have a partial side effect (e.g., an email half-sent) before it crashed. Durable execution and idempotent side effects are complementary, not the same guarantee.
- **Forgetting that `update_state` bypasses node logic entirely.** Manually writing values that a node would normally validate or transform can leave state in a shape downstream nodes don't expect, causing confusing failures several steps later.

---

## Summary

- Long-running agentic workflows fail routinely — deploys, restarts, crashes, and multi-day human waits are the normal operating environment, not edge cases — and without checkpointing, every one of those events destroys all progress.
- A **checkpointer** persists the graph's full state after every super-step; LangGraph ships `MemorySaver` (in-process, non-persistent), `SqliteSaver` (single-file, single-instance persistent), and `PostgresSaver` (production, multi-instance safe), injected via `graph.compile(checkpointer=...)`.
- **`thread_id`**, passed through `config={"configurable": {"thread_id": ...}}`, scopes checkpoints to a specific conversation/job lineage; it's required on every call once a checkpointer is attached.
- The durable execution flow is: execute a super-step → checkpoint the merged state → (possible crash) → reload the graph with the same checkpointer backend → `invoke(None, config)` with the same `thread_id` → resume from the last checkpoint instead of restarting.
- **`get_state(config)`** inspects a thread's current (or a specific historical) checkpoint; **`update_state(config, values, as_node=...)`** writes a manual correction as a new checkpoint without running a node — the backbone of admin tooling, debugging, and human correction.
- **`get_state_history(config)`** enumerates a thread's full checkpoint lineage; passing an earlier checkpoint's config into `invoke()` performs **time travel** — forking a new branch of execution from that point without destroying the original history.
- Checkpointing is the foundation Chapter 12's `interrupt()`/`Command(resume=...)` human-in-the-loop pattern is built on: pausing for a human and resuming from a crash are, to the checkpointer, the identical operation.

---

## Knowledge Check

1. A team ships a LangGraph agent using `MemorySaver` because "it worked in every demo." Explain precisely what will happen the first time the hosting process restarts mid-workflow, and why the demos never revealed this problem.
2. Why must every `.invoke()`/`.stream()` call include a `thread_id` once a checkpointer is attached to a graph, even for graphs that used to run fine without one?
3. Walk through, in your own words, why a checkpoint is written after a *super-step* rather than after each individual node — and what that implies for a graph with two nodes running concurrently in the same super-step.
4. A developer resumes a crashed job by calling `graph.invoke(original_input, config=config)` instead of `graph.invoke(None, config=config)`. For a state field defined as `Annotated[list[str], operator.add]`, what goes wrong?
5. Explain the difference between what `update_state` does and what actually running a node does. Why is `update_state` explicitly the mechanism a human-in-the-loop resume relies on (per Chapter 12's forward pointer)?
6. You're asked to add "replay this job from three steps ago with a corrected prompt" as an admin feature. Which two APIs from this chapter would you combine, and in what order, to implement it without destroying the original run's history?

---

## Hands-on Exercises

1. **Crash-and-resume with SQLite.** Build a 3-node sequential graph (any simple state and node logic you like — e.g., accumulating a list of "steps completed"). Compile it with `SqliteSaver` pointed at a file on disk. Write a script that: (a) streams through only the first node, printing state after each super-step, then deliberately stops (simulating a crash); (b) in a second, separate script invocation (a real second process, or at minimum a fresh `compile()` call reusing the same DB file), calls `get_state` to confirm where execution paused, then calls `invoke(None, config)` to resume to completion. Confirm the final state reflects all three nodes' work.

2. **Build an admin inspector.** Using the Research Assistant example (or your own graph from Exercise 1), write a small function `describe_thread(graph, thread_id)` that uses `get_state` and `get_state_history` to print: the current state values, which node(s) are queued to run next, and a step-by-step log of every checkpoint in the thread's history with its step number and a short summary of what changed at that step.

3. **Time travel and fork.** Extend Exercise 1 or 2: run a graph to completion, then use `get_state_history` to locate the checkpoint corresponding to the middle node's completion. Call `update_state` on that checkpoint's config to inject a deliberately different value for one state field (simulating "correcting a mistake"), then call `invoke(None, config=that_config)` to resume forward from the fork. Print both the original run's final state and the forked run's final state, and confirm the original checkpoint history is still intact (i.e., you haven't destroyed the first branch).

---

## Further Reading

- [LangGraph Persistence Documentation](https://docs.langchain.com/oss/python/langgraph/persistence) — official reference for checkpointers, threads, and the state APIs covered in this chapter
- [LangGraph Application Structure Guide](https://docs.langchain.com/oss/python/langgraph/application-structure) — how checkpointer configuration fits into a deployable LangGraph service
- [LangGraph GitHub Repository](https://github.com/langchain-ai/langgraph) — source for `langgraph-checkpoint-sqlite` and `langgraph-checkpoint-postgres` packages, useful for reading the exact `Checkpoint`/`CheckpointTuple` data structures
- **Chapter 6 (Reducers)** in this course — required background for understanding how checkpointed state merges across super-steps
- **Chapter 7 (Compilation & Execution)** in this course — the super-step execution loop this chapter's checkpoint timing depends on
- **Chapter 12 (Human-in-the-Loop)** in this course — the full treatment of `interrupt()` and `Command(resume=...)`, built directly on the checkpointing primitives introduced here

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./08-tool-calling-patterns.md">← Previous: Tool Calling Patterns</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./10-memory-management.md">Next: Memory Management →</a>
</div>
