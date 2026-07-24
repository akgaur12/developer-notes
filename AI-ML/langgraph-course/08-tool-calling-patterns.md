# Chapter 8: Tool Calling Patterns

> "A chain calls a tool once. An agent calls a tool, looks at what came back, and decides what to do next." — the one-sentence reason this chapter exists

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain precisely why `bind_tools` alone is insufficient to *execute* a multi-step tool-calling interaction, and why that execution loop is naturally a graph
- Build the canonical ReAct-style LangGraph pattern from scratch: an agent node, a conditional edge, a `ToolNode`, and the loop-back edge that ties them together
- Describe exactly what `ToolNode` does under the hood — parsing `tool_calls`, dispatching to matching tools, executing them (in parallel when there are several), and appending `ToolMessage` results to state
- Handle multiple simultaneous tool calls from a single LLM turn, including partial failures, without crashing the graph
- Wire heterogeneous tools — a pure-Python calculator, an HTTP-backed weather API, a MongoDB lookup, a parameterized SQL query, and a generic REST tool — into the same agent loop
- Adapt MCP-server-exposed tools into LangChain `BaseTool` objects and drop them into a `ToolNode` exactly like any other tool
- Recognize the common failure modes of tool-calling graphs (infinite loops, schema mismatches, silent tool crashes) and how to guard against them

---

## Prerequisites for the Chapter

This chapter opens **Phase 2: Execution Engine** and assumes you're fluent in everything from Phase 1:

- **StateGraph & State Management** (Chapter 2) — you know how to define state with `TypedDict`/`Pydantic`, and critically, you understand **reducers** (Chapter 6), because the message list in this chapter's state depends entirely on the `add_messages` reducer to append rather than overwrite.
- **Nodes** (Chapter 3) — a node is just a callable that takes state and returns a partial update; the "agent node" in this chapter is nothing more than that, wrapping an LLM call.
- **Edges & Routing, and Commands** (Chapters 4–5) — you know what a conditional edge is and how routing functions inspect state to pick the next node.
- **Compilation & Execution** (Chapter 7) — you understand the super-step execution loop and recursion limits; this chapter's ReAct loop is precisely the kind of construct that can hit `GraphRecursionError` if it never terminates, so that context matters here.

On the LangChain Core side (per the course prerequisites), you should already be comfortable with:

- The `@tool` decorator and building tools from plain Python functions with typed arguments and docstrings
- `llm.bind_tools([...])`, and reading `AIMessage.tool_calls` off a model response
- `ToolMessage`, and how a tool's return value gets threaded back into a conversation

If any of that is fuzzy, this chapter's Section 1 gives a fast recap before diving into what's actually new: **orchestrating** that interaction as a graph rather than a one-shot call.

---

## 1. From `bind_tools` to a Graph: Why a Single Call Isn't Enough

### 1.1 Recap: `bind_tools` at the LangChain Core level

You already know this shape:

```python
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

@tool
def get_weather(city: str) -> str:
    """Return the current weather conditions for a given city."""
    return f"It's sunny and 24°C in {city}."

llm = ChatOpenAI(model="gpt-4o")
llm_with_tools = llm.bind_tools([get_weather])

response = llm_with_tools.invoke("What's the weather in Lisbon?")
print(response.tool_calls)
# [{'name': 'get_weather', 'args': {'city': 'Lisbon'}, 'id': 'call_abc123', 'type': 'tool_call'}]
```

`bind_tools` does exactly one thing: it serializes each tool's name, description, and argument schema into the provider's function-calling format and attaches that schema to every request the model makes. The model, when it decides a tool is useful, doesn't execute anything — it emits a **structured request** to call one (`AIMessage.tool_calls`). Nothing has actually run yet. The weather hasn't been looked up; `get_weather` hasn't been invoked.

To finish the job at the LangChain Core level, you'd manually:

```python
if response.tool_calls:
    call = response.tool_calls[0]
    result = get_weather.invoke(call["args"])
    tool_message = ToolMessage(content=result, tool_call_id=call["id"])
    final = llm_with_tools.invoke([*messages, response, tool_message])
```

That's a fine demo for *one* tool call. But real tool-using agents don't stop after one round: the model might call a tool, read the result, and decide it needs to call a *second* tool before it can answer — or reformulate the same tool call with different arguments, or call three tools at once, or (eventually) decide it has enough information and produce a final natural-language answer with no `tool_calls` at all. That's not a fixed three-step chain. It's an unbounded loop whose length is decided *by the model, at runtime*, one turn at a time.

### 1.2 The problem: tool calling is a loop, not a call

Try to express "call the LLM, execute whatever tools it asks for, feed results back, repeat until it stops asking" as a linear LCEL chain and you hit a wall immediately: LCEL's `RunnableSequence`/`RunnableParallel` primitives describe a fixed, statically-known pipeline shape. They have no native way to say "go back to step 2 an unknown number of times, conditional on inspecting the output of step 3." You *can* hack it with a hand-rolled `while` loop wrapped in a custom `Runnable`, but at that point you're reimplementing — badly, without checkpointing, streaming, or visibility — exactly the thing LangGraph already is.

This is precisely why the ReAct pattern (Yao et al., 2022 — **Re**ason + **Act**) is usually the very first thing taught in LangGraph: it is the smallest possible example of a workflow that *requires* a cycle, and tool calling is the most common reason your graph needs one. The mental model:

```
LLM decides to call a tool
        │
        ▼
   tool executes
        │
        ▼
result goes back to the LLM
        │
        ▼
LLM decides: call another tool, or answer?
        │
   (loop until "answer")
```

Every arrow in that diagram is an edge in a graph. Every box is a node. The "decides" diamond is a conditional edge. That's the entire chapter, informally, before we write a line of code.

---

## 2. The Canonical ReAct Loop in LangGraph

### 2.1 State shape

The state for a tool-calling agent is almost always just a running message list:

```python
from typing import Annotated
from typing_extensions import TypedDict
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

The `add_messages` reducer (Chapter 6) is doing critical, easy-to-miss work here: without it, each node's `{"messages": [...]}` return value would **replace** the whole list rather than append to it, and the model would lose all prior conversation and tool-result context on every super-step. `add_messages` also does ID-aware merging — if a returned message shares an `id` with an existing one, it replaces that message in place instead of duplicating it, which matters once you start streaming partial `AIMessage` chunks (Chapter 11).

LangGraph ships a pre-built equivalent, `langgraph.graph.MessagesState`, which is exactly the class above. Use it directly when you don't need extra state fields; subclass it when you do (e.g., adding a `user_id` or `session_metadata` key alongside `messages`).

### 2.2 The agent node

The agent node is a plain node (Chapter 3) that calls the tool-bound LLM on the current message history and returns its response as a state update:

```python
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(model="claude-sonnet-4-5")
llm_with_tools = llm.bind_tools([calculator, get_weather, lookup_customer])

def call_agent(state: AgentState) -> dict:
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}
```

Note what this node does *not* do: it never inspects `response.tool_calls` itself, never executes a tool, and never decides what happens next. It has exactly one job — produce the next `AIMessage` — and that single-responsibility shape is what makes it trivially testable in isolation (Chapter 17): feed it a fixed message list, assert on the returned message, no graph required.

### 2.3 The conditional edge: inspecting `tool_calls`

The routing decision — "does the last message request a tool, or is this the final answer?" — is a conditional edge (Chapter 4). You can write it by hand:

```python
from langgraph.graph import END

def should_continue(state: AgentState) -> str:
    last_message = state["messages"][-1]
    if getattr(last_message, "tool_calls", None):
        return "tools"
    return END
```

LangGraph ships this exact check pre-built as `langgraph.prebuilt.tools_condition`, which does the same `tool_calls` inspection (plus a couple of defensive edge cases, like tolerating a state dict whose message key isn't literally `"messages"`) and returns the literal string `"tools"` or `END`. Reach for the prebuilt version once you've internalized what it's doing — there's no value in hand-maintaining a duplicate of it once you trust it.

### 2.4 `ToolNode`: the executor

The one genuinely new primitive in this chapter is `ToolNode` — a prebuilt node type from `langgraph.prebuilt` whose entire job is to look at the most recent `AIMessage`, run whichever tools it requested, and produce the corresponding `ToolMessage` objects:

```python
from langgraph.prebuilt import ToolNode

tool_node = ToolNode([calculator, get_weather, lookup_customer])
```

That's it — you build one by handing it a list of `BaseTool` instances (or plain `@tool`-decorated functions, which LangChain coerces to `BaseTool` automatically). `ToolNode` builds an internal name → tool lookup from that list once, at construction time.

### 2.5 Wiring it together

```python
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import tools_condition

graph = StateGraph(AgentState)
graph.add_node("agent", call_agent)
graph.add_node("tools", tool_node)

graph.add_edge(START, "agent")
graph.add_conditional_edges(
    "agent",
    tools_condition,
    {"tools": "tools", END: END},
)
graph.add_edge("tools", "agent")   # the loop-back edge

app = graph.compile()
```

Trace the super-steps for a single query like *"What's 15% of 240, and what's the weather in Lisbon?"*:

1. **Super-step 1** — `START → agent`. The LLM sees the human message, decides it needs two tools, and returns an `AIMessage` with **two** `tool_calls` (one for `calculator`, one for `get_weather`).
2. **Conditional edge** — `tools_condition` sees a non-empty `tool_calls` list, routes to `"tools"`.
3. **Super-step 2** — `agent → tools`. `ToolNode` executes both tool calls (in parallel — Section 3.3) and returns **two** `ToolMessage` objects, one per `tool_call_id`.
4. **Edge** — `tools → agent` (unconditional, always taken).
5. **Super-step 3** — the LLM now sees the original question plus both tool results and produces a final `AIMessage` with `tool_calls == []` and real content.
6. **Conditional edge** — `tools_condition` sees no tool calls, routes to `END`.

Six steps, no tool calls hardcoded anywhere, and it generalizes to zero tool calls (answers immediately), one, or a long chain of dependent lookups — all driven by what the model decides at each turn, not by a shape you had to predict in advance.

---

## 3. `ToolNode` Deep Dive

### 3.1 What `ToolNode` does automatically

It's worth being explicit about the work `ToolNode` removes from you, because it's easy to underrate a primitive that looks like "just a for-loop":

1. **Reads the last message** in `state["messages"]` and asserts it's an `AIMessage` with a `tool_calls` attribute (raising a clear error if the graph was wired wrong and `ToolNode` got invoked on a state with no pending tool calls).
2. **Resolves each requested tool by name** against the tool list you constructed it with, validating that the name the model emitted actually matches a registered tool.
3. **Validates and coerces arguments** against each tool's Pydantic argument schema before invoking it — the same schema validation `bind_tools` published to the model in the first place.
4. **Executes every requested tool call**, synchronously via `.invoke()` for sync tools, or with `.ainvoke()` under `ToolNode.ainvoke`/inside an async graph — and, when there's more than one call in the same turn, executes them **concurrently** (Section 3.3).
5. **Wraps each tool's return value in a `ToolMessage`**, preserving the `tool_call_id` from the original request so the LLM (and the message history) can correlate each result back to the call that produced it.
6. **Catches exceptions per tool call** (by default) and converts them into an error-content `ToolMessage` instead of propagating the exception and crashing the graph run (Section 3.4).
7. **Returns a state update** — `{"messages": [tool_message, ...]}` — that merges into state via the `add_messages` reducer exactly like any other node's output.

None of this is magic; it's the same boilerplate you'd write by hand for one tool call, generalized correctly to N tool calls with proper error isolation. The value of using the prebuilt version isn't cleverness, it's not having to get steps 3 and 6 subtly wrong in your own graph.

### 3.2 Building one from a list of tools

```python
from langgraph.prebuilt import ToolNode

tools = [calculator, get_weather, lookup_customer]
tool_node = ToolNode(tools)
```

`ToolNode` also accepts a `messages_key` argument (default `"messages"`) for state schemas where your message list lives under a different key, and a `name` argument if you want the node registered under something other than `ToolNode`'s default when you add it with `graph.add_node(...)`.

You almost never call `ToolNode` directly outside a graph, but nothing stops you from unit-testing it in isolation the same way you'd test any node — hand it a state dict with a synthetic `AIMessage` carrying a hand-built `tool_calls` list, and assert on the `ToolMessage`s it returns. This is a good pattern to keep in your back pocket for Chapter 17.

### 3.3 Multiple tool calls and parallel execution

Modern tool-calling models routinely return **more than one** `tool_calls` entry in a single `AIMessage` when the user's request decomposes into independent lookups — "what's the weather in Lisbon and Tokyo, and what's 12% of 88?" is naturally three parallel tool calls, not three sequential ones.

`ToolNode` handles this by executing all requested tool calls concurrently for that turn — for synchronous tools, using a thread pool; for `async def` tools inside an async graph run, using `asyncio.gather` under the hood. You don't opt into this; it's the default behavior the moment the last `AIMessage` contains more than one `tool_calls` entry. This matters practically:

- **I/O-bound tools** (an HTTP weather call, a MongoDB query, a REST lookup) benefit enormously — three network calls in parallel instead of serially cuts wall-clock latency roughly to the slowest single call rather than the sum.
- **Order is not guaranteed** across the parallel calls, but correctness doesn't depend on order — each result is a `ToolMessage` tagged with the `tool_call_id` it answers, so the LLM (and you, inspecting state) can always tell which result belongs to which request regardless of completion order.
- **Side-effecting tools that must not run concurrently** (e.g., two tools that both write to the same document, or a "decrement inventory" tool called twice) need you to either avoid emitting them in the same turn (via prompt/tool design) or make the underlying operation safe under concurrent execution (idempotency keys, database-level locking) — `ToolNode` will not serialize them for you.

### 3.4 Error handling: an error `ToolMessage`, not a crash

If a tool raises — the weather API times out, the MongoDB connection drops, the SQL query hits a constraint violation — you do **not** want that exception unwinding the entire graph run and losing conversation state. The whole point of routing the result back through the LLM is that **the model itself can often recover**: told "the weather API timed out," it might retry with different arguments, fall back to a different tool, or apologize and answer with the information it already has.

`ToolNode` defaults to `handle_tool_errors=True`, which catches any exception raised inside a tool call and turns it into a `ToolMessage` whose content describes the error, with `status="error"` set on the message, rather than propagating it:

```python
tool_node = ToolNode(tools, handle_tool_errors=True)  # this is already the default
```

You can customize the behavior:

```python
def format_tool_error(exc: Exception) -> str:
    return f"Tool failed: {type(exc).__name__}: {exc}. Consider trying different arguments."

tool_node = ToolNode(tools, handle_tool_errors=format_tool_error)
```

— or pass a fixed string, or `False` to opt back into "let it raise and crash the graph" (useful in tests, where you *want* a hard failure rather than a swallowed one). This chapter only needs you to know the mechanism exists and why it's the default; **Chapter 18 (Error Handling & Resilience)** goes much deeper — retry policies per tool, distinguishing transient vs. permanent tool failures, circuit breakers around flaky external APIs, and structured error taxonomies the agent can reason about rather than a bare string.

---

## 4. Full Worked Example: Calculator, Weather, and MongoDB Agent

Here's a complete, self-contained agent wiring three heterogeneous tools — a pure computation, an external HTTP-backed API, and a database lookup — into a single ReAct loop.

```python
import re
import requests
from typing import Annotated
from typing_extensions import TypedDict

from langchain_core.tools import tool
from langchain_core.messages import AnyMessage
from langchain_anthropic import ChatAnthropic

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

from pymongo import MongoClient

# --- Tools -------------------------------------------------------------

@tool
def calculator(expression: str) -> str:
    """Evaluate a basic arithmetic expression, e.g. '15 * 0.20' or '(3 + 4) / 2'.
    Only numbers, parentheses, and + - * / are supported."""
    if not re.fullmatch(r"[0-9\.\+\-\*\/\(\)\s]+", expression):
        raise ValueError(f"Unsupported characters in expression: {expression!r}")
    return str(eval(expression, {"__builtins__": {}}, {}))


@tool
def get_weather(city: str) -> str:
    """Look up current weather conditions for a given city name."""
    response = requests.get(
        "https://api.example-weather.com/v1/current",
        params={"q": city},
        timeout=5,
    )
    response.raise_for_status()
    data = response.json()
    return f"{city}: {data['condition']}, {data['temp_c']}°C"


_mongo = MongoClient("mongodb://localhost:27017")
_customers = _mongo["support_db"]["customers"]

@tool
def lookup_customer(customer_id: str) -> str:
    """Look up a customer's account summary by their customer ID from the
    support database. Returns plan tier, signup date, and open ticket count."""
    doc = _customers.find_one({"_id": customer_id})
    if doc is None:
        return f"No customer found with id {customer_id!r}."
    return (
        f"Customer {customer_id}: plan={doc['plan']}, "
        f"signed_up={doc['signup_date']}, open_tickets={doc['open_tickets']}"
    )


tools = [calculator, get_weather, lookup_customer]

# --- State ---------------------------------------------------------------

class AgentState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]

# --- LLM + agent node ------------------------------------------------------

llm = ChatAnthropic(model="claude-sonnet-4-5")
llm_with_tools = llm.bind_tools(tools)

def call_agent(state: AgentState) -> dict:
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}

# --- Graph -----------------------------------------------------------------

tool_node = ToolNode(tools)

builder = StateGraph(AgentState)
builder.add_node("agent", call_agent)
builder.add_node("tools", tool_node)

builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", tools_condition, {"tools": "tools", END: END})
builder.add_edge("tools", "agent")

app = builder.compile()

# --- Run ---------------------------------------------------------------

result = app.invoke({
    "messages": [
        ("user", "Customer C-4821 is asking why their bill is 15% higher "
                  "than last month's $80. Also, what's the weather like "
                  "in Lisbon where they're based?")
    ]
})
print(result["messages"][-1].content)
```

Walking through what happens: the model receives a query that plausibly needs all three tools. It emits an `AIMessage` requesting `calculator("80 * 0.15")` and `get_weather("Lisbon")` in parallel (independent of each other), and separately (or in the same turn, model-dependent) `lookup_customer("C-4821")`. `ToolNode` executes whichever calls arrive together concurrently, appends the resulting `ToolMessage`s, and the loop returns to `agent`. If the customer lookup reveals an open billing dispute ticket, the model might issue a *second* round of tool calls before finally composing a natural-language answer — and the graph handles that unbounded extra round with zero code changes, because the loop condition is "does the last message have tool calls," not "have we called tools exactly once."

---

## 5. Plugging In a SQL Tool and a Generic REST Tool

The pattern generalizes to any tool shape — nothing about `ToolNode` cares whether a tool's body talks to Mongo, a SQL database, or an arbitrary REST endpoint. Two more examples that slot directly into the same `tools` list:

```python
from sqlalchemy import create_engine, text

_engine = create_engine("postgresql://user:pass@localhost:5432/orders")

@tool
def order_status(order_id: str) -> str:
    """Look up an order's current status and expected delivery date by order ID."""
    with _engine.connect() as conn:
        row = conn.execute(
            text("SELECT status, eta FROM orders WHERE order_id = :oid"),
            {"oid": order_id},
        ).fetchone()
    if row is None:
        return f"No order found with id {order_id!r}."
    return f"Order {order_id}: status={row.status}, eta={row.eta}"


@tool
def call_rest_api(method: str, path: str, json_body: dict | None = None) -> str:
    """Call an internal REST API. `method` is GET/POST/PUT/DELETE, `path` is the
    endpoint path (e.g. '/v1/shipments/123'), and `json_body` is an optional
    request payload for POST/PUT. Returns the JSON response as text."""
    url = f"https://internal-api.example.com{path}"
    response = requests.request(method.upper(), url, json=json_body, timeout=10)
    response.raise_for_status()
    return response.text
```

Both are ordinary `@tool` functions with typed arguments and a docstring — that's the entire integration contract. Add them to the `tools` list passed to both `bind_tools(...)` and `ToolNode(...)`, and the ReAct loop, the conditional edge, and the error handling from Section 3.4 apply automatically with no graph changes. This is the payoff of routing tool execution through a graph node instead of hand-wiring each tool's call site: **the orchestration logic is tool-agnostic**, and adding tool #6 is a one-line change to a list, not a new branch of control flow.

A generic REST tool like `call_rest_api` deserves one caution: giving the model a single "call any endpoint with any method and body" tool is powerful but removes the safety of per-tool argument schemas — the model can, in principle, construct any request. For anything beyond an internal read-mostly API behind auth you control, prefer several narrow, purpose-built tools (`get_shipment_status`, `create_return_label`, ...) over one broad tool, for the same reason you'd prefer narrow, well-typed REST endpoints over a single `/execute` endpoint that accepts arbitrary commands.

---

## 6. MCP Tools in the Same Loop

Because you already know MCP, this is worth being explicit about: an MCP server exposes tools as JSON-RPC-described capabilities, not as LangChain `BaseTool` objects — but the `langchain-mcp-adapters` package bridges that gap by connecting to an MCP server and converting each of its exposed tools into a `BaseTool` with a matching name, description, and argument schema:

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

mcp_client = MultiServerMCPClient({
    "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/data"],
        "transport": "stdio",
    },
    "search": {
        "url": "https://mcp.example.com/mcp",
        "transport": "streamable_http",
    },
})

mcp_tools = await mcp_client.get_tools()   # list[BaseTool]

tools = [calculator, get_weather, lookup_customer, *mcp_tools]
tool_node = ToolNode(tools)
llm_with_tools = llm.bind_tools(tools)
```

Once adapted, an MCP tool is indistinguishable from `calculator` or `lookup_customer` as far as `ToolNode`, `bind_tools`, and `tools_condition` are concerned — it's a `BaseTool` with a name, a schema, and an `.invoke()`/`.ainvoke()` method. The same conditional-edge loop, the same parallel execution for multiple simultaneous calls, and the same error-to-`ToolMessage` handling all apply without modification. The practical implication: if you're migrating an existing MCP-based tool ecosystem into LangGraph, the migration is "adapt once, then treat as an ordinary tool list" — not a parallel code path you maintain alongside native LangChain tools.

---

## Examples

**Minimal manual conditional edge** (equivalent to `tools_condition`, useful when you want custom routing — e.g., routing to a human-approval node instead of `tools` for a specific sensitive tool name):

```python
from langgraph.graph import END

def route_after_agent(state: AgentState) -> str:
    last = state["messages"][-1]
    calls = getattr(last, "tool_calls", None)
    if not calls:
        return END
    if any(c["name"] == "issue_refund" for c in calls):
        return "human_review"   # preview of Chapter 12
    return "tools"

builder.add_conditional_edges(
    "agent",
    route_after_agent,
    {"tools": "tools", "human_review": "human_review", END: END},
)
```

**Async agent + async tools**, needed once any tool does real network I/O and you want the whole graph non-blocking end to end:

```python
@tool
async def get_weather_async(city: str) -> str:
    """Look up current weather conditions for a given city name (async)."""
    async with httpx.AsyncClient(timeout=5) as client:
        resp = await client.get("https://api.example-weather.com/v1/current", params={"q": city})
        resp.raise_for_status()
        data = resp.json()
    return f"{city}: {data['condition']}, {data['temp_c']}°C"

async def call_agent_async(state: AgentState) -> dict:
    response = await llm_with_tools.ainvoke(state["messages"])
    return {"messages": [response]}

# builder.add_node("agent", call_agent_async) — ToolNode auto-detects and
# awaits async tools when the graph is run with `await app.ainvoke(...)`.
```

**Inspecting a run's tool traffic** (useful while developing, before you wire up LangSmith tracing in Chapter 20):

```python
for message in result["messages"]:
    if message.type == "ai" and message.tool_calls:
        for call in message.tool_calls:
            print(f"→ calling {call['name']}({call['args']})")
    elif message.type == "tool":
        print(f"← {message.name} returned: {message.content!r}")
```

---

## Diagrams

The core ReAct loop as a graph — note the single cycle between `agent` and `tools`, broken only when the conditional edge routes to `END`:

```mermaid
flowchart TD
    START([START]) --> AGENT[agent node<br/>LLM with tools bound]
    AGENT -->|inspect last message| DECIDE{tool_calls present?}
    DECIDE -->|yes| TOOLS[ToolNode<br/>executes requested tool calls]
    DECIDE -->|no, final answer| END([END])
    TOOLS -->|append ToolMessage results| AGENT

    style AGENT fill:#e1f5ff
    style TOOLS fill:#fff3e0
    style DECIDE fill:#f3e5f5
```

A single agent turn that requests three independent tool calls, showing `ToolNode`'s concurrent execution and the correlation via `tool_call_id`:

```mermaid
sequenceDiagram
    participant U as User
    participant A as agent node (LLM)
    participant T as ToolNode
    participant Cal as calculator
    participant Wx as get_weather
    participant DB as lookup_customer (Mongo)

    U->>A: "Bill 15% higher than $80, weather in Lisbon, check C-4821"
    A->>T: AIMessage.tool_calls = [calc, weather, lookup] (3 calls)
    par concurrent execution
        T->>Cal: invoke(expression="80 * 0.15")
        T->>Wx: invoke(city="Lisbon")
        T->>DB: invoke(customer_id="C-4821")
    end
    Cal-->>T: "12.0"
    Wx-->>T: "Lisbon: Sunny, 24°C"
    DB-->>T: "plan=Pro, open_tickets=1"
    T->>A: 3x ToolMessage (tagged by tool_call_id)
    A->>U: Final natural-language answer
```

---

## Real-World Scenarios

**Customer support copilot (SaaS).** A support platform gives its LLM three tools: `lookup_customer` (Mongo), `order_status` (Postgres), and `search_kb` (a vector-store-backed retriever, foreshadowing Chapter 6's cousin discipline — retrieval-as-a-tool). A single ticket like "Where's my order and why does the invoice look wrong?" triggers two parallel tool calls in the first agent turn, and depending on what `order_status` returns, a follow-up `search_kb` call in the second turn to pull the relevant billing-FAQ passage. The ReAct loop from this chapter is the entire orchestration layer; the "product" is really just tool design plus a good system prompt.

**Data analyst agent over a SQL warehouse.** An internal analytics tool exposes a single `run_sql_query(query: str)` tool backed by a read-replica Postgres connection with a query timeout and a row-count cap enforced inside the tool body (never trust the model to self-limit). Analysts ask natural-language questions; the agent loop lets the model issue an exploratory query, see the (possibly wrong) shape of the result, and refine the query in a second tool call — a pattern that's only possible because the loop feeds results back for another LLM turn instead of running a tool exactly once.

**Ops assistant that mixes read and write tools.** An on-call assistant has `get_service_health` (read-only, safe to auto-execute) and `restart_service` (destructive). The team does **not** let both live in an unguarded `ToolNode` — they route `restart_service` through a conditional edge to a `human_review` node using `interrupt()` (Chapter 12) before it ever reaches execution, while `get_service_health` flows straight through the ordinary `ToolNode`. This chapter's pattern is the default; sensitive tools need the human-in-the-loop variant layered on top, not a different architecture.

**Multi-source enterprise search agent (MCP).** A company already runs MCP servers for its ticketing system and internal wiki (built for use with MCP-aware IDEs and other MCP clients). Rather than reimplementing those integrations as native LangChain tools, the LangGraph agent adapts both servers' tools via `langchain-mcp-adapters` and merges them into the same `tools` list as a couple of natively-defined LangChain tools (a calculator, an internal REST call). One `ToolNode`, one agent loop, tools from two completely different integration mechanisms treated identically.

---

## Best Practices

- **Prefer the prebuilt `ToolNode` and `tools_condition` over hand-rolled equivalents** once you understand what they do — they're already handling the argument-validation and error-isolation edge cases you'd otherwise have to get right yourself.
- **Write tight, single-purpose tool docstrings.** The docstring *is* the model's only signal for when and how to call the tool — vague descriptions ("does customer stuff") produce vague tool-selection behavior. Treat tool docstrings with the same care you'd give a public API's documentation.
- **Validate and sanitize inside the tool, not just in the schema.** A Pydantic argument schema tells the model what shape to send; it doesn't stop a malformed or adversarial value from reaching your database. The `calculator` example above rejects non-arithmetic characters before calling `eval` for exactly this reason — never `eval` a model-supplied string unguarded in production.
- **Keep tools narrow rather than one mega-tool.** Several well-typed tools (`get_order_status`, `list_recent_orders`) give the model — and you, reading traces — much clearer signal than one `execute_database_operation(sql: str)` tool that can do anything.
- **Set a recursion limit you've actually thought about** (Chapter 7). A tool-calling loop is exactly the shape that can spin — a model that keeps re-calling a tool with slightly different arguments, never satisfied — so don't leave the default recursion limit unexamined in production.
- **Let tool failures reach the model as data, not exceptions**, via `handle_tool_errors` — the model can often self-correct ("that city name was misspelled, retry") far more gracefully than your own retry code, and least-privilege error messages ("customer not found") avoid leaking stack traces into a user-facing conversation.
- **Design for parallel execution deliberately.** If two tools you're registering must never run concurrently (shared mutable state, rate-limited API, non-idempotent write), don't rely on the model never emitting them in the same turn — make the tool implementation itself safe under concurrency, or split them across separate agent turns via prompt/tool design.
- **Test nodes in isolation before testing the graph.** `call_agent` and `tool_node` are both plain callables you can invoke directly against a synthetic state dict, with no graph, no LLM API key, and no network — reserve full-graph integration tests for testing the *loop and routing*, not each tool's internal correctness (Chapter 17).

---

## Common Mistakes

- **Forgetting the loop-back edge.** Adding `agent`, `tools`, and the conditional edge but omitting `graph.add_edge("tools", "agent")` leaves the graph unable to feed tool results back to the model — it'll execute tools once and then have no path forward, raising an error about a node with no outgoing edge (or silently dead-ending, depending on how you structured the fallback).
- **Binding tools to one LLM instance and invoking a different, un-bound one.** `llm.bind_tools([...])` returns a *new* runnable; forgetting to use that returned object (and instead calling the original `llm.invoke(...)`) means the model never even sees the tool schemas and will never emit `tool_calls` no matter how the prompt is worded.
- **Mismatched tool lists between `bind_tools` and `ToolNode`.** If the LLM is bound with tools `[a, b, c]` but `ToolNode` was constructed with only `[a, b]`, a call to `c` reaches `ToolNode` and fails to resolve — always construct both from the same single source list.
- **No recursion limit awareness.** Shipping the ReAct loop with the default recursion limit unexamined, then being surprised in production when a model gets stuck in a call-tool-reinterpret-call-tool-again cycle and the run aborts with `GraphRecursionError` partway through a user-facing request.
- **Swallowing tool errors silently with no signal to the model.** Setting `handle_tool_errors` to return an empty string or a generic "error" with no detail gives the model nothing to act on — it'll often just repeat the identical failing call. A useful error `ToolMessage` says *what* went wrong in a way the model can adapt to (bad argument, transient timeout, not found).
- **Treating tool arguments as pre-validated because the schema exists.** A JSON schema constrains shape and type, not semantic validity — a `customer_id` string can be schema-valid and still be a SQL-injection attempt or a nonexistent ID. Validate/escape inside the tool body regardless of what the schema promises.
- **One giant "do anything" tool instead of several narrow ones**, which both confuses tool selection (the model has to guess the right sub-behavior from a single description) and removes your ability to reason about or restrict what the model can do at the individual-capability level.
- **Running non-idempotent side-effecting tools (charge a card, send an email, delete a record) through a plain `ToolNode` with no review gate** — this chapter's loop auto-executes whatever the model requests; anything destructive or irreversible belongs behind the human-in-the-loop pattern from Chapter 12, not the bare agent/tools loop.

---

## Summary

- `bind_tools` only gets a model to *emit* a structured request to call a tool (`AIMessage.tool_calls`); nothing has executed yet. Turning that into a working multi-turn tool-using agent requires a **loop**: call the LLM, execute requested tools, feed results back, repeat until the model stops requesting tools — and that unbounded, conditional loop is naturally expressed as a graph, not a linear chain.
- The canonical **ReAct pattern** in LangGraph is four pieces: an **agent node** (LLM with tools bound), a **conditional edge** that inspects the last message's `tool_calls`, a **`ToolNode`** that executes whatever was requested, and an **edge back to the agent node** that closes the loop.
- **`ToolNode`** parses `tool_calls` off the last `AIMessage`, resolves and validates each call against the matching registered tool, executes all of a turn's calls **concurrently**, wraps each result in a `ToolMessage` tagged with the originating `tool_call_id`, and — by default — converts any tool exception into an error-content `ToolMessage` instead of crashing the graph (`handle_tool_errors`, deepened in Chapter 18).
- The pattern is **tool-agnostic**: a calculator, an HTTP weather API, a MongoDB lookup, a parameterized SQL query, and a generic REST tool all plug into the same `tools` list, the same `bind_tools` call, and the same `ToolNode` with zero orchestration changes.
- **MCP-exposed tools**, once adapted via `langchain-mcp-adapters` into `BaseTool` instances, behave identically to natively defined LangChain tools inside this same loop — no parallel code path needed.
- Sensitive, destructive, or irreversible tools should not flow through the unguarded loop in this chapter — route them to a human-approval node instead, a pattern this chapter deliberately sets up for and Chapter 12 completes.

---

## Knowledge Check

1. `bind_tools` has been called and the model returns an `AIMessage` with a populated `tool_calls` list. What, concretely, has and has not happened at that point? Why can't a linear LCEL chain alone finish the job for an arbitrary number of tool-calling rounds?
2. Draw (in words or as a small diagram) the four-node/edge shape of the canonical ReAct loop in LangGraph, and explain what would go wrong at runtime if the `tools → agent` edge were omitted.
3. A single agent turn produces an `AIMessage` with three `tool_calls`. Explain, step by step, what `ToolNode` does with all three, including how the resulting `ToolMessage`s stay correctly associated with the right original request.
4. One of the three tool calls in question 3 raises an exception (say, a MongoDB connection timeout). With `handle_tool_errors` at its default setting, what does the graph do next, and why is that usually preferable to letting the exception propagate and crash the run?
5. You need to add a fourth tool — a REST call to an internal inventory API — to the agent from Section 5. List every place in the code that needs to change to include it, and every place that does *not* need to change.
6. A teammate proposes wiring a `delete_user_account` tool into the same unguarded `ToolNode` used for read-only lookups. What risk does this introduce, and what pattern (previewed in this chapter, completed in a later one) should be used instead?

---

## Hands-On Exercises

1. **Build the loop from scratch.** Starting from an empty file, define `AgentState`, two tools of your choosing (e.g., a unit converter and a mock stock-price lookup), an agent node, and wire up the full ReAct loop using `tools_condition` and `ToolNode`. Run it with a query that requires zero tools, then a query that requires exactly one, then a query that plausibly requires both in sequence. Print the full `messages` list after each run and verify the `ToolMessage`s appear where you expect.

2. **Force and observe a parallel turn.** Add a third tool to your Exercise 1 graph, then craft a single user query specifically designed to make the model request all three tools in one `AIMessage` (e.g., "Convert 10 miles to km, look up AAPL's price, and tell me if 10% of that price is over $20"). Instrument your agent node to print `len(response.tool_calls)` and confirm you get more than one. Then instrument (or log inside) one of your tools to print a timestamp on entry and exit, and confirm from the timestamps that the calls overlapped rather than running strictly one after another.

3. **Break it on purpose, then fix it.** Modify one of your tools so it deterministically raises an exception for a specific input (e.g., your stock-price tool raises `ConnectionError` if the ticker is `"BAD"`). First, run with `ToolNode`'s default error handling and confirm the graph completes with a final answer rather than crashing — inspect the error `ToolMessage`'s content. Then set `handle_tool_errors=False` on the same `ToolNode`, re-run the identical query, and observe the graph raise instead. Write down, in your own words, which behavior you'd want in a production customer-facing deployment versus a CI test asserting a specific tool never fails.

---

## Further Reading

- [LangGraph Documentation — Tool calling](https://docs.langchain.com/oss/python/langgraph/overview) — overview of prebuilt agent/tool-calling components
- [LangGraph Application Structure Guide](https://docs.langchain.com/oss/python/langgraph/application-structure) — where `ToolNode` and related prebuilts fit in a larger application
- [LangGraph GitHub Repository](https://github.com/langchain-ai/langgraph) — source for `ToolNode`, `tools_condition`, and `create_react_agent` in `langgraph.prebuilt`
- Yao et al., *"ReAct: Synergizing Reasoning and Acting in Language Models"* (2022) — the original paper behind the reason/act loop this chapter's graph shape implements
- [`langchain-mcp-adapters` (PyPI/GitHub)](https://github.com/langchain-ai/langchain-mcp-adapters) — the adapter library used in Section 6 to convert MCP tools into LangChain `BaseTool` instances
- Related course in this repo: [LangChain Core — From LLM/FastAPI Engineer to Production LCEL Practitioner](../langchain-core-course/00-index.md) — for a deeper recap of `@tool`, `bind_tools`, and message types assumed as prerequisite here

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./07-compilation-and-execution.md">← Previous: Compilation & Execution</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./09-checkpointing-and-durable-execution.md">Next: Checkpointing & Durable Execution →</a>
</div>
