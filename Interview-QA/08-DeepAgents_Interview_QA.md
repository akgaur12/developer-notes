# 🕵️ DeepAgents Interview Q&A

## 🔹 Fundamentals

### 1. What is DeepAgents?
`deepagents` is a Python package (from the LangChain team) for building **"deep" agents** — agents capable of long-horizon, multi-step tasks — by giving a standard LLM tool-calling loop four specific capabilities: **planning, sub-agents, a virtual file system, and a detailed system prompt**. It's built on top of LangGraph.

---

### 2. Why do we need "deep agents" instead of a plain ReAct tool-calling agent?
A plain ReAct loop (LLM → tool → LLM → tool → ...) works for short tasks but breaks down on **long-horizon tasks**: the LLM loses track of the overall plan, context gets cluttered with intermediate tool outputs, and there's no way to delegate isolated sub-tasks without polluting the main context. DeepAgents patterns — inspired by systems like Claude Code and OpenAI's Deep Research — solve this with explicit planning, context isolation via sub-agents, and persistent file-based state.

---

### 3. What are the four core pillars of the DeepAgents architecture?
1. **Planning tool** – an explicit todo-list tool the agent uses to write and update a plan
2. **Sub-agents** – the ability to spawn isolated agents for specific sub-tasks (context quarantine)
3. **Virtual file system** – tools to read/write/edit files that persist across the agent's steps
4. **Detailed system prompt** – a carefully engineered prompt (inspired by Claude Code's own prompt) that instructs the model on *how* to use planning, sub-agents, and files effectively

---

### 4. What is `create_deep_agent`?
The main entry point of the library — a function that assembles a LangGraph agent pre-wired with the planning tool, file system tools, sub-agent tool, and system prompt, given a model, a list of custom tools, and optional sub-agent configs.
```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    tools=[internet_search],
    instructions="You are an expert researcher...",
)
agent.invoke({"messages": [{"role": "user", "content": "Research X"}]})
```

---

### 5. Is DeepAgents a replacement for LangGraph or LangChain?
No — it's a **higher-level abstraction built on top of LangGraph**. Under the hood, `create_deep_agent` returns a compiled LangGraph graph (based on the `create_react_agent`-style loop), so everything you know about LangGraph state, streaming, and checkpointers still applies. DeepAgents just pre-bakes in the planning/sub-agent/file-system tools and prompt scaffolding.

---

## 🔹 Planning

### 6. What is the planning tool (`write_todos`) in DeepAgents?
A built-in tool that lets the agent create and update a **structured todo list** (each item with a status: pending/in_progress/completed) as part of its own working state, rather than trying to hold the whole plan implicitly in its reasoning. This mirrors how coding agents like Claude Code track multi-step tasks.

---

### 7. Why does explicit planning improve agent performance on long tasks?
- Forces the model to decompose a large task before acting, reducing wasted/aimless tool calls
- Gives the model (and the user, if surfaced in the UI) a visible source of truth for progress
- Reduces the chance of the model "forgetting" earlier sub-goals as context grows
- Makes it easy to resume a task after an interruption since the plan is part of the persisted state

---

### 8. Is the todo list enforced/executed by code, or is it purely advisory?
It's advisory — the plan is just structured state that the agent (via the system prompt's instructions) chooses to write and update. The graph doesn't force the agent to execute steps in order; the value comes from the LLM being explicitly prompted to plan before acting and to keep the list current.

---

## 🔹 Sub-Agents

### 9. What is a sub-agent in DeepAgents, and why use one?
A sub-agent is a **separate, isolated agent invocation** (with its own context window) that the main agent spawns via a `task` tool to handle a specific, self-contained piece of work. Its job is **context quarantine** — long or noisy intermediate work (e.g. reading through many search results) happens in the sub-agent's own context, and only the final, distilled result is returned to the main agent's context.

---

### 10. How do you define custom sub-agents?
By passing a `subagents` list to `create_deep_agent`, each with a name, description (so the main agent knows when to delegate to it), its own system prompt/instructions, and optionally its own restricted toolset:
```python
research_subagent = {
    "name": "researcher",
    "description": "Used to research a specific sub-question in depth.",
    "prompt": "You are a focused research assistant...",
    "tools": [internet_search],
}
agent = create_deep_agent(tools=[...], subagents=[research_subagent])
```

---

### 11. What's the analogy between DeepAgents sub-agents and orchestration in multi-agent systems?
It's essentially a **supervisor/worker pattern**: the main agent acts as an orchestrator that decides when a task is self-contained enough to delegate, spawns a worker (sub-agent) to do it in isolation, and incorporates the worker's summarized result — similar to the "supervisor" multi-agent pattern in LangGraph, but packaged as a single `task` tool call instead of a custom graph topology.

---

### 12. What happens to a sub-agent's intermediate tool calls/context after it finishes?
They stay **inside the sub-agent's own execution** and are discarded/not merged into the main agent's message history — only the sub-agent's final response is returned as the `task` tool's output. This is the key mechanism that keeps the main agent's context window clean on long tasks.

---

### 13. When should you NOT delegate to a sub-agent?
- The task is small/quick — delegation overhead (extra LLM round trip) isn't worth it
- The task genuinely needs the main agent's full accumulated context to complete correctly
- The sub-task's result needs to influence *how* the main agent asks a follow-up sub-question interactively (sub-agents are typically single-shot, not conversational back-and-forth)

---

## 🔹 Virtual File System

### 14. What is the virtual file system in DeepAgents?
A set of built-in tools (`ls`, `read_file`, `write_file`, `edit_file`) that let the agent **persist and manipulate text content as "files"** across its execution — used for things like saving research notes, draft outputs, or intermediate artifacts that would otherwise have to stay awkwardly stuffed into the conversation context.

---

### 15. Where is the virtual file system actually stored?
By default, files are stored **in the LangGraph state** itself (not the real disk) — so they persist for the duration of a run and across steps of the graph, and are checkpointed along with everything else if a checkpointer is attached. DeepAgents also supports pluggable backends so files can be backed by a real filesystem or other storage when needed.

---

### 16. Why use a virtual file system instead of just keeping everything in the conversation messages?
- Keeps the LLM's context window from bloating with large intermediate artifacts (e.g. a long draft report)
- Lets the agent **selectively re-read** only the parts of a file it needs, rather than re-processing everything
- Enables patterns like "write a draft, review it, edit it" that mirror how a human researcher/engineer works with files
- Provides a natural hand-off point between the main agent and sub-agents (e.g. a sub-agent writes findings to a file, main agent reads a summary)

---

### 17. How does the file system tie into planning and sub-agents?
A typical deep-agent flow: the main agent writes a plan (`write_todos`), delegates research sub-tasks to sub-agents which write their findings to files, and the main agent then reads those files (`read_file`) to synthesize a final answer — combining all three pillars so the main context window only ever holds the plan, tool-call summaries, and final synthesis, not raw intermediate data.

---

## 🔹 System Prompt & Model Configuration

### 18. Why does DeepAgents ship with a detailed default system prompt instead of leaving it to the developer?
Just adding planning/file/sub-agent tools isn't enough — the LLM needs **explicit instructions on when and how to use them** (e.g. "always write a plan before starting multi-step work," "delegate research to a sub-agent instead of doing it inline"). The default prompt encodes these behavioral patterns, similar to how Claude Code's own system prompt teaches the model to use its todo/task tools effectively.

---

### 19. Can you customize or override the system prompt?
Yes — `create_deep_agent` accepts an `instructions` (or `system_prompt`) parameter that is combined with (or can override) the built-in base prompt, letting you add domain-specific guidance while keeping the underlying planning/sub-agent/file-system behavior intact.

---

### 20. Is DeepAgents tied to a specific LLM provider?
No — since it's built on LangChain's model abstraction, `create_deep_agent` accepts any LangChain-compatible chat model (Anthropic, OpenAI, etc.), as long as the model supports **tool/function calling**, which the framework relies on for planning, file, and sub-agent tools.

---

## 🔹 State, Memory & Human-in-the-Loop

### 21. How does DeepAgents handle memory/persistence across turns?
Since the compiled agent is a LangGraph graph, you attach a **checkpointer** (e.g. `MemorySaver`, or a Postgres/SQLite-backed one) exactly as you would for any LangGraph app, keyed by `thread_id` — this persists conversation history, the todo list, and virtual files together as one state object.

---

### 22. Does DeepAgents support human-in-the-loop approval?
Yes — because it's LangGraph under the hood, you can use LangGraph's `interrupt()` mechanism to pause execution (e.g. before a sensitive tool call) for human review/approval before resuming, the same as in any LangGraph application.

---

### 23. How would you add long-term (cross-thread) memory to a DeepAgents agent?
By attaching a LangGraph `Store` (in addition to the per-thread checkpointer), which lets the agent persist and retrieve information (e.g. user preferences, past research) **across different conversation threads**, not just within a single session.

---

## 🔹 Comparisons & Design Decisions

### 24. DeepAgents vs a plain LangGraph `create_react_agent` — what's the actual difference?
| `create_react_agent` | `create_deep_agent` |
|----|----|
| Bare think→act→observe loop | Same loop + planning, sub-agent, and file tools pre-wired |
| No enforced context-management strategy | Built-in context quarantine via sub-agents |
| Minimal default prompt | Detailed prompt teaching planning/delegation/file usage |
| Best for short, simple tool-use tasks | Best for long-horizon, multi-step tasks (research, coding, report writing) |

---

### 25. DeepAgents vs hand-building a custom multi-agent LangGraph — when would you pick DeepAgents?
Pick DeepAgents when your task fits the "planner + delegator + file-based scratchpad" pattern (research assistants, coding agents, report generators) and you want that scaffolding **out of the box**. Hand-build a custom LangGraph when your control-flow needs don't match that shape — e.g. a fixed multi-stage pipeline with very specific branching logic, or a supervisor pattern with many distinct specialist agent types that need bespoke coordination logic.

---

### 26. What real-world systems inspired the DeepAgents design?
The package is explicitly modeled on the architecture patterns observed in **Claude Code** (planning todos, sub-agent delegation, file-based workspace) and **deep research agents** (e.g. OpenAI's Deep Research, Manus-style agents) — i.e., taking the patterns that make coding/research agents effective on long tasks and packaging them as a reusable, model-agnostic library.

---

### 27. What are the trade-offs of the sub-agent context-quarantine approach?
**Pros**: keeps the main agent's context small and focused, allows parallelizable delegation, reduces distraction from irrelevant intermediate data.
**Cons**: extra LLM round trips (latency/cost) for delegation, potential loss of nuance since only a summarized result crosses back into the main context, and sub-agents can't easily ask the main agent clarifying questions mid-task.

---

## 🔹 Practical / Production Considerations

### 28. How do you restrict what tools a sub-agent can use?
Each sub-agent config can specify its own `tools` list (a subset of, or different from, the main agent's tools), so a "research" sub-agent might only get search tools while a "coding" sub-agent gets file-editing and execution tools — limiting blast radius and keeping each sub-agent focused.

---

### 29. How would you debug a DeepAgents agent that isn't planning or delegating effectively?
- Inspect the todo list state and file system contents at each step (same as inspecting LangGraph state)
- Use LangGraph/LangSmith tracing to see whether the model is actually calling `write_todos`/`task` or skipping them
- Check that sub-agent `description` fields are clear enough for the main agent to know when to delegate
- Verify the underlying model reliably supports tool calling — weaker models may ignore multi-tool instructions in the system prompt

---

### 30. What are common pitfalls when building with DeepAgents?
- Treating the todo list as code-enforced instead of advisory — the model can still skip or mis-order steps
- Giving sub-agents vague descriptions, causing the main agent to under- or over-delegate
- Forgetting to attach a checkpointer, losing plan/file state on restart for long-running tasks
- Not restricting sub-agent toolsets, letting sub-agents perform actions outside their intended scope
- Assuming context quarantine is free — every sub-agent call is a full additional LLM invocation, with real latency/cost implications

---
