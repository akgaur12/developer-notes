# 🕸️ LangGraph Interview Q&A

## 🔹 Fundamentals

### 1. What is LangGraph?
LangGraph is a library (from the LangChain team) for building **stateful, multi-step LLM applications** as a graph of nodes and edges, giving explicit, developer-controlled flow — including branches, loops/cycles, persistence, and human-in-the-loop steps — that's hard to express with a simple linear chain or a black-box agent loop.

---

### 2. Why use LangGraph instead of a plain LangChain chain or `AgentExecutor`?
- `AgentExecutor` gives the LLM a fixed think→act→observe loop with limited control over branching/looping logic
- LCEL chains are great for **linear/DAG** pipelines but can't express cycles
- LangGraph models the app as an explicit **graph with state**, so you control exactly when to loop, branch, pause for human approval, or fan out to multiple agents — while still using LangChain components (models, tools, retrievers) inside nodes

---

### 3. What are the core building blocks of a LangGraph application?
- **State** – a shared data structure (usually a `TypedDict` or Pydantic model) passed between nodes
- **Nodes** – functions (or Runnables) that receive state and return updates to it
- **Edges** – connections that define which node runs next (normal, conditional, or entry/finish points)
- **Graph** – the compiled structure (`StateGraph`) tying nodes and edges together into a runnable app
- **Checkpointer** – persists state across steps/runs for memory, replay, and human-in-the-loop

---

### 4. What is `StateGraph`?
The main class used to define a LangGraph application: you declare a state schema, add nodes (`add_node`), add edges (`add_edge` / `add_conditional_edges`), set an entry point, and then `.compile()` it into a runnable graph.
```python
from langgraph.graph import StateGraph, START, END

graph = StateGraph(MyState)
graph.add_node("retrieve", retrieve_node)
graph.add_node("generate", generate_node)
graph.add_edge(START, "retrieve")
graph.add_edge("retrieve", "generate")
graph.add_edge("generate", END)
app = graph.compile()
```

---

### 5. What is the "State" in LangGraph and how is it defined?
A structured object (typically a `TypedDict`, `dataclass`, or Pydantic `BaseModel`) representing all the data flowing through the graph — e.g. messages, retrieved documents, intermediate results. Each node receives the current state and returns a **partial update**, which LangGraph merges into the overall state.
```python
from typing import TypedDict, Annotated
from operator import add

class MyState(TypedDict):
    messages: Annotated[list, add]
    question: str
```

---

### 6. What are `START` and `END` in LangGraph?
Special sentinel nodes representing the **entry point** and **termination point** of the graph. `add_edge(START, "node")` defines where execution begins; `add_edge("node", END)` defines where it can stop.

---

### 7. What is a Node in LangGraph?
A Python function (or Runnable) that takes the current state (and optionally a config) as input and returns a dict of state updates. Nodes contain the actual logic — calling an LLM, invoking a tool, running a retriever, etc.

---

### 8. What is an Edge, and what types of edges exist?
An edge defines the transition from one node to the next.
- **Normal edge** – always goes from node A to node B
- **Conditional edge** – a routing function inspects the state and returns the name of the next node (or `END`) dynamically
- **Entry point** – edge from `START` to the first node
- Edges are what allow LangGraph to express **branches and cycles**, unlike a strictly linear LCEL chain

---

## 🔹 State Management

### 9. How does LangGraph merge state updates from nodes?
Each field in the state schema can have a **reducer function** (via `Annotated[type, reducer]`) that defines how a node's returned value for that field is combined with the existing value — e.g. `operator.add` to append to a list (common for chat `messages`), or the default "overwrite" behavior if no reducer is specified.

---

### 10. What is `add_messages` in LangGraph?
A built-in reducer specifically for chat message lists that appends new messages to existing ones (and intelligently updates a message in place if it shares the same ID) — used almost universally for the `messages` field in chatbot/agent state.
```python
from langgraph.graph.message import add_messages
class State(TypedDict):
    messages: Annotated[list, add_messages]
```

---

### 11. What happens if two nodes update the same state key without a reducer?
By default, the **last write wins** (the value is simply overwritten) — which is why any field that needs to accumulate across nodes (like message history) must declare an explicit reducer such as `add` or `add_messages`.

---

### 12. Can nodes run in parallel and update state simultaneously?
Yes — when a node has multiple outgoing edges to different nodes that don't depend on each other, LangGraph can execute them **concurrently**, and their state updates are merged using the field reducers. Without proper reducers, concurrent writes to the same key would conflict/overwrite unpredictably.

---

## 🔹 Control Flow

### 13. How do you implement branching logic in LangGraph?
Using `add_conditional_edges`, which takes the source node and a routing function that inspects state and returns the name of the next node (or a list of names for fan-out):
```python
def route(state):
    return "tools" if state["messages"][-1].tool_calls else END

graph.add_conditional_edges("agent", route, {"tools": "tools", END: END})
```

---

### 14. How do you implement a loop/cycle in LangGraph?
By adding an edge that routes back to an earlier node — e.g. an agent node that conditionally routes to a "tools" node, which then routes back to "agent", repeating until the routing function decides to go to `END`. This is exactly how a ReAct-style tool-calling agent loop is built in LangGraph.

---

### 15. What is the prebuilt `create_react_agent` (or ToolNode pattern) in LangGraph?
A high-level prebuilt helper that wires up the common "LLM decides to call a tool → ToolNode executes it → result goes back to LLM" loop automatically, so you don't have to hand-build the graph for standard tool-calling agents.

---

### 16. What is a `ToolNode`?
A prebuilt LangGraph node that takes tool calls from the last AI message in state, executes the corresponding tools, and returns their outputs as `ToolMessage`s appended to state — the standard way to execute tools inside a graph.

---

### 17. How do you fan out to multiple nodes dynamically at runtime (map-style)?
Using the `Send` API, which lets a routing function dispatch a **variable number of parallel invocations** of a node, each with different input state — e.g. running the same "process item" node once per item in a list, without hard-coding the number of branches at graph-definition time.

---

## 🔹 Persistence & Memory

### 18. What is a Checkpointer in LangGraph?
A component that **persists the graph's state** after every step (e.g. `MemorySaver` for in-memory, or SQLite/Postgres/Redis-backed savers for production), keyed by a thread ID — enabling conversation memory across turns, resuming interrupted runs, and time-travel/replay.

---

### 19. How does LangGraph provide conversation memory across turns?
By compiling the graph with a checkpointer and invoking it with a consistent `thread_id` in the config; the graph automatically loads the previous state for that thread before running and saves the updated state after — no manual memory-object wiring needed.
```python
app = graph.compile(checkpointer=MemorySaver())
app.invoke({"messages": [...]}, config={"configurable": {"thread_id": "user-1"}})
```

---

### 20. What is "time travel" in LangGraph?
The ability to inspect or resume execution from **any previous checkpoint** in a thread's history (via `get_state_history`), useful for debugging, branching alternate conversation paths, or rolling back after an error.

---

### 21. What is Human-in-the-Loop (HITL) in LangGraph, and how is it implemented?
A pattern where graph execution **pauses** to let a human review/approve/edit state before continuing — implemented via `interrupt()` (or the older `interrupt_before` / `interrupt_after` compile options) which halts execution at a given node; a human (or external system) inspects/modifies the checkpointed state, then execution resumes.

---

### 22. Why is persistence important for agentic applications specifically?
Agentic workflows can be long-running, involve external side effects (tool calls), and need to survive process restarts, support pausing for approval, and allow retries from a known-good state — a checkpointer makes the graph **resumable and durable** instead of losing all progress on failure.

---

## 🔹 Multi-Agent & Advanced Patterns

### 23. How does LangGraph support multi-agent systems?
Each agent can be modeled as a node (or a subgraph) in a larger graph, with edges/routing logic controlling **which agent acts next** and how information passes between them — common topologies include supervisor (a router agent delegates to specialist agents) and network (agents can hand off to each other directly).

---

### 24. What is the "supervisor" multi-agent pattern?
A pattern where a central **supervisor/orchestrator node** (usually an LLM) examines the current state/task and decides which specialist agent node should act next, looping until the task is complete — useful when different sub-tasks require different tools/expertise.

---

### 25. What is a Subgraph in LangGraph?
A compiled graph that is **embedded as a node** inside a larger parent graph, enabling modular composition and reuse — e.g. a reusable "RAG subgraph" plugged into multiple larger agent graphs, each with its own internal state schema.

---

### 26. How do you handle errors/retries in a LangGraph node?
- Wrap node logic in try/except and route to an error-handling node via conditional edges
- Use LangGraph's retry policies (`RetryPolicy`) on `add_node` to automatically retry transient failures
- Persist state via a checkpointer so a failed run can be resumed rather than restarted from scratch

---

## 🔹 Streaming & Execution

### 27. How does streaming work in LangGraph?
`app.stream()` / `app.astream()` can emit updates in different modes:
- `"values"` – the full state after each step
- `"updates"` – only the diff/update each node produced
- `"messages"` – token-by-token streaming of LLM output from within nodes
This allows a UI to show intermediate reasoning/tool calls as they happen, not just the final answer.

---

### 28. What's the difference between `.invoke()`, `.stream()`, and `.batch()` on a compiled LangGraph app?
Same `Runnable` interface as LangChain: `.invoke()` runs to completion and returns final state, `.stream()` yields incremental updates as the graph executes, and `.batch()` runs multiple independent inputs concurrently.

---

### 29. How do you visualize a LangGraph graph?
Compiled graphs expose `.get_graph().draw_mermaid()` (or `draw_png()`), generating a diagram of nodes and edges — useful for debugging complex control flow before running it.

---

### 30. What is `recursion_limit` in LangGraph, and why does it matter?
A safety limit on the number of **super-steps** (graph traversal steps) allowed in a single run, preventing infinite loops in cyclic graphs (e.g. an agent stuck repeatedly calling tools without converging) from running forever.

---

## 🔹 Comparisons & Design Decisions

### 31. LangGraph vs LangChain's `AgentExecutor` — when do you pick which?
| `AgentExecutor` | LangGraph |
|----|----|
| Simple think→act→observe loop | Fully custom graph topology |
| Limited control over branching | Explicit conditional edges, cycles, fan-out |
| No built-in persistence/checkpointing | First-class checkpointer/persistence |
| Good for quick, standard tool-calling agents | Needed for multi-agent, HITL, long-running, or complex workflows |

---

### 32. LangGraph vs plain LCEL chains — when do you need LangGraph?
LCEL composes Runnables into a **DAG** (no cycles) — perfect for RAG pipelines, simple sequential/parallel logic. The moment you need a **loop** (e.g. "keep calling tools until the LLM says it's done"), conditional state-dependent routing, persisted long-running state, or multi-agent coordination, you need LangGraph's explicit graph + state model.

---

### 33. Can LangGraph nodes use LangChain components internally?
Yes — nodes are just Python functions, so they commonly wrap LCEL chains, chat models, retrievers, or tools inside them. LangGraph handles the **orchestration/control-flow layer**; LangChain components still handle prompting, retrieval, and model calls.

---

### 34. What is LangGraph Platform / LangGraph Cloud (at a high level)?
A deployment/hosting layer for LangGraph applications that provides managed infrastructure for running graphs as APIs, along with built-in persistence, horizontal scaling, and integration with LangGraph Studio for visual debugging — for teams that don't want to self-host the checkpointer/server layer.

---

### 35. What are common pitfalls when building with LangGraph?
- Forgetting reducers on accumulating state fields → data gets overwritten instead of appended
- Not setting a `recursion_limit` → infinite loops in cyclic graphs
- Overusing a single flat graph instead of subgraphs → hard-to-follow control flow
- Mutating state objects in place inside a node instead of returning updates → breaks reducer-based merging
- Skipping a checkpointer in production → losing state on crashes and no HITL support

---
