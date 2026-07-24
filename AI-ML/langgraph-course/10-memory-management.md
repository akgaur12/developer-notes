# Chapter 10: Memory Management

> "A conversation without memory is just a sequence of unrelated interruptions." — paraphrased from every frustrated user of a stateless chatbot

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain the difference between **short-term memory** (thread-scoped conversation state) and **long-term memory** (cross-session, cross-thread facts about a user), and why LangGraph treats them as two separate subsystems rather than one
- Describe precisely how short-term memory is backed by the checkpointer and `thread_id` you learned in Chapter 9, and what happens to it when a new `thread_id` is used
- Use the `Store` interface (`InMemoryStore`, and a persistent backend such as `PostgresStore`) with its four core operations: `put`, `get`, `search`, and `delete`
- Design hierarchical **namespaces** for long-term memory (e.g., `(user_id, "preferences")` vs. `("users", user_id, "memories")`) and explain why sloppy namespacing silently degrades retrieval quality and leaks data across users
- Write a single node that reads **both** short-term state (this conversation) and long-term store (facts from past conversations) and writes back to both in the same turn
- Decide what's worth persisting long-term — explicit preferences, LLM-extracted facts/summaries — and explain why dumping raw conversation transcripts into long-term memory doesn't scale
- Configure **semantic search** over long-term memory using an embedding index, and connect that mechanism to the embedding/RAG concepts you already know
- Build a "Personal Assistant" node that combines short- and long-term memory, and a "User Preference Memory" component that survives across sessions

---

## Prerequisites for the Chapter

This chapter assumes you're comfortable with:

- **Chapter 2 (StateGraph & State Management)**: how state schemas, reducers, and `Annotated` types work — short-term memory is, mechanically, just a graph state field.
- **Chapter 9 (Checkpointing & Durable Execution)**: the checkpointer (`MemorySaver`, `SqliteSaver`, `PostgresSaver`), the `thread_id` concept in `config["configurable"]`, and the idea that a checkpointer persists a graph's state across separate `.invoke()` calls. This chapter builds directly on that mechanism — short-term memory in LangGraph *is* checkpointed state, viewed through a different lens.
- **Chapter 3 (Nodes)**: node function signatures, and how nodes receive `state` and `config` and return partial state updates.
- General familiarity with **embeddings and vector similarity search** (cosine similarity, nearest-neighbor retrieval) is helpful for Section 5 of this chapter, but not required — the concepts are re-introduced at the level needed here.

No new installation is required beyond what Chapter 9 already set up (`langgraph`, `langgraph-checkpoint-sqlite` or `langgraph-checkpoint-postgres` if you're following along with a persistent backend). If you want to follow the semantic-search examples, you'll also want an embeddings-capable chat model integration (e.g., `langchain-openai`) available, though the concepts stand on their own without running the code.

---

## 1. Two Kinds of Memory: A Mental Model

Every conversational AI system needs to answer two very different questions:

1. **"What have we been talking about in *this* conversation?"** — turn 3 needs to remember what was said in turn 1.
2. **"What do I know about *this user*, independent of which conversation we're currently having?"** — a user who told the assistant three weeks ago, in a completely different session, that they're vegetarian shouldn't have to repeat that fact today.

LangGraph gives you two distinct, purpose-built mechanisms for these two questions, and conflating them is the single most common design mistake newcomers make:

| | Short-term memory | Long-term memory |
|---|---|---|
| **Scope** | One `thread_id` | One user (or org, or agent) — spans many `thread_id`s |
| **Lifetime** | As long as that thread's checkpoints are retained | Indefinite — until explicitly deleted |
| **Backed by** | The **checkpointer** (Chapter 9) + graph **state** | The **Store** (introduced this chapter) |
| **Typical content** | The running message list, scratch variables, in-progress tool call results | User preferences, durable facts, summaries of past conversations |
| **Access pattern** | Automatically loaded/saved every super-step, keyed by `thread_id` | Explicit `put`/`get`/`search` calls inside a node, keyed by a namespace you design |
| **Analogy** | RAM for *this process* | A database row for *this user*, queryable from any process |

The key insight: **short-term memory is a side effect of checkpointing you already learned.** You don't need a new API to get it — it's simply the `messages` field (or any other field) in your graph's state, persisted across invocations by the checkpointer under a given `thread_id`. **Long-term memory is a genuinely new API surface** — the `Store` — because by definition it must outlive any single thread and be addressable independently of the checkpointing mechanism.

```mermaid
flowchart LR
    subgraph ST["Short-Term Memory (this conversation)"]
        direction TB
        T1["thread_id = 'sess-001'"] --> S1["State: messages[], scratch vars"]
        S1 -->|"checkpointer saves after each super-step"| CP[("Checkpoint DB\n(Ch. 9)")]
    end

    subgraph LT["Long-Term Memory (across all conversations)"]
        direction TB
        U["user_id = 'user-42'"] --> N1["namespace: (user-42, preferences)"]
        U --> N2["namespace: (user-42, memories)"]
        N1 --> ST2[("Store\nput / get / search")]
        N2 --> ST2
    end

    NODE["Graph node"] -->|"reads/writes via thread_id"| ST
    NODE -->|"reads/writes via user_id + namespace"| LT
```

Notice both memory types can be in play **inside the same node, in the same turn** — that combination is the subject of Section 6, and it's what separates a merely functional chatbot from a genuinely useful assistant.

---

## 2. Short-Term Memory: State Scoped to a Thread

### 2.1 Mechanics recap: it's just checkpointed state

There is no separate "short-term memory API" in LangGraph. Short-term memory is whatever you put in your `StateGraph`'s state schema, combined with a checkpointer and a stable `thread_id`. Here's the minimal shape:

```python
from typing import Annotated
from typing_extensions import TypedDict

from langchain_core.messages import AnyMessage, HumanMessage
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver


class ChatState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]


def chat_node(state: ChatState):
    # In a real graph this calls an LLM; kept minimal here for clarity.
    last_user_msg = state["messages"][-1].content
    reply = f"You said: {last_user_msg!r}. I have {len(state['messages'])} messages of context."
    return {"messages": [("ai", reply)]}


builder = StateGraph(ChatState)
builder.add_node("chat", chat_node)
builder.add_edge(START, "chat")
builder.add_edge("chat", END)

checkpointer = MemorySaver()
app = builder.compile(checkpointer=checkpointer)
```

`messages` is short-term memory. The `add_messages` reducer (Chapter 6) appends new messages instead of overwriting the list, and the checkpointer persists the resulting list after every super-step, keyed by `thread_id`.

### 2.2 The running message list, in practice

```python
config = {"configurable": {"thread_id": "sess-001"}}

app.invoke({"messages": [HumanMessage("My name is Priya.")]}, config)
# -> messages: [Human("My name is Priya."), AI("You said: 'My name is Priya.'. I have 1 messages of context.")]

app.invoke({"messages": [HumanMessage("What's my name?")]}, config)
# -> the checkpointer loaded the PRIOR two messages before running the node,
#    so state["messages"] now has 3 entries when chat_node executes, then 4 after.
```

Every `.invoke()` against the same `thread_id` implicitly does: **load latest checkpoint for this thread → merge new input into state via reducers → run the graph → save new checkpoint.** That load/merge/save cycle *is* short-term memory. You already learned every piece of this mechanism in Chapter 9; this chapter just names the pattern.

### 2.3 Bounding growth: trimming and summarization

Short-term memory has a real cost: every message in `state["messages"]` typically gets sent to the LLM on every subsequent turn, so an unbounded message list means unbounded (and eventually context-window-breaking) token costs. Two common mitigation patterns:

**Trimming** — keep only the most recent N messages or the most recent N tokens:

```python
from langchain_core.messages import trim_messages

def chat_node_with_trimming(state: ChatState):
    trimmed = trim_messages(
        state["messages"],
        max_tokens=4000,
        strategy="last",
        token_counter=count_tokens,   # your model's tokenizer
        include_system=True,
    )
    response = llm.invoke(trimmed)
    return {"messages": [response]}
```

Note that trimming here only affects what's *sent to the LLM this turn* — `trim_messages` doesn't mutate `state["messages"]` itself unless you explicitly return trimmed content as the new state. The full history still accumulates in the checkpoint (which is usually what you want — you don't want to have *actually deleted* turn 2 just because turn 40 doesn't need it in the prompt).

**Summarization** — periodically collapse older messages into a single summary message, and actually shrink the persisted state using `RemoveMessage`:

```python
from langchain_core.messages import RemoveMessage, SystemMessage

def summarize_if_long(state: ChatState):
    if len(state["messages"]) <= 20:
        return {}

    older, recent = state["messages"][:-6], state["messages"][-6:]
    summary_text = llm.invoke(
        [SystemMessage("Summarize this conversation excerpt in 3-4 sentences."), *older]
    ).content

    # RemoveMessage deletes by id from the persisted state (add_messages reducer
    # understands RemoveMessage specially); the summary replaces the deleted span.
    return {
        "messages": [RemoveMessage(id=m.id) for m in older]
        + [SystemMessage(f"Earlier conversation summary: {summary_text}")]
    }
```

This is the short-term-memory equivalent of what Section 7 discusses for long-term memory: raw accumulation doesn't scale, so you compress.

### 2.4 Isolation: a new `thread_id` is a blank slate

This is the property that trips up almost everyone the first time:

```python
config_a = {"configurable": {"thread_id": "sess-001"}}
config_b = {"configurable": {"thread_id": "sess-002"}}

app.invoke({"messages": [HumanMessage("My name is Priya.")]}, config_a)
app.invoke({"messages": [HumanMessage("What's my name?")]}, config_b)
# The node has NO idea "Priya" was ever mentioned — sess-002 has never been
# checkpointed before, so it starts with an empty messages list.
```

This isn't a bug — it's the entire point of thread-scoping. `sess-001` and `sess-002` might be two different browser tabs, two different customer support tickets, or two different days for the same user. Short-term memory answers "what happened in *this* thread," full stop. If you need "what do we know about this *user* regardless of thread," you need long-term memory — Section 3 onward.

---

## 3. Long-Term Memory: The Store Interface

### 3.1 `BaseStore` and its backends

LangGraph's answer to cross-thread persistence is the **`Store`** — a key-value store purpose-built for agent memory, addressed by hierarchical **namespaces** rather than a single flat table. Every store implementation satisfies the `BaseStore` interface, so you can develop against an in-memory store and swap in a persistent one for production without changing node code — exactly the same pattern Chapter 9 used for checkpointers.

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()
```

`InMemoryStore` is the development/testing default — fast, zero setup, but gone the moment the process restarts. For anything that needs to survive a restart, LangGraph ships persistent store backends that mirror the checkpointer backends from Chapter 9:

```python
from langgraph.store.postgres import PostgresStore

DB_URI = "postgresql://user:password@localhost:5432/langgraph_memory"

with PostgresStore.from_conn_string(DB_URI) as store:
    store.setup()  # one-time schema migration — same idea as checkpointer.setup() in Ch. 9
    # ... use store ...
```

An async equivalent (`AsyncPostgresStore`) exists for use inside `async def` nodes, following the same sync/async split you saw with `PostgresSaver`/`AsyncPostgresSaver` in Chapter 9.

### 3.2 The four core operations

Every store exposes the same small, deliberately simple API:

```python
# put(namespace, key, value) — write or overwrite an item
store.put(
    namespace=("user-42", "preferences"),
    key="communication_style",
    value={"tone": "concise", "preferred_language": "en"},
)

# get(namespace, key) -> Item | None — exact-key read
item = store.get(("user-42", "preferences"), "communication_style")
print(item.value)          # {"tone": "concise", "preferred_language": "en"}
print(item.created_at)     # datetime the item was first written
print(item.updated_at)     # datetime of the most recent put()

# search(namespace_prefix, query=None, filter=None, limit=10) -> list[SearchItem]
results = store.search(("user-42", "preferences"))
for r in results:
    print(r.namespace, r.key, r.value, r.score)   # score is None without a query/index

# delete(namespace, key) — remove an item
store.delete(("user-42", "preferences"), "communication_style")
```

`namespace` is always a tuple of strings, evaluated as a **path**, not a single opaque key — `search` with a namespace *prefix* returns every item stored under that prefix or deeper. This hierarchical addressing is what makes namespace *design* (Section 4) a real engineering decision rather than an afterthought.

### 3.3 Wiring a store into the graph

A store is attached at compile time, alongside (and independently of) the checkpointer:

```python
app = builder.compile(checkpointer=checkpointer, store=store)
```

Once compiled this way, any node whose function signature declares a `store` keyword-only parameter of type `BaseStore` gets it **automatically injected** by the runtime — you never pass it explicitly in `.invoke()`:

```python
from langgraph.store.base import BaseStore
from langchain_core.runnables import RunnableConfig

def personalized_node(state: ChatState, config: RunnableConfig, *, store: BaseStore):
    user_id = config["configurable"]["user_id"]
    prefs = store.get(("user-" + user_id, "preferences"), "settings")
    ...
```

This mirrors how the checkpointer is invisible to node code (it operates above the node, at the super-step boundary) — the store is visible *inside* the node, because unlike the checkpointer, reading and writing long-term memory is a deliberate decision your node logic makes on a case-by-case basis, not an automatic side effect of running a super-step.

---

## 4. Namespacing Conventions for Long-Term Memory

### 4.1 Namespaces are a path, design them like one

A namespace tuple is the *only* thing standing between "the right user gets the right memories" and "user A reads user B's private preferences." Treat it with the same care you'd give a database schema or a multi-tenant partition key. Common conventions, in increasing order of structure:

```python
# Flat: one namespace per user, one key per fact type
(user_id, "preferences")
(user_id, "memories")

# Hierarchical: group under a top-level "users" segment (avoids collisions with
# other top-level namespace families you might add later, e.g. ("orgs", org_id, ...))
("users", user_id, "preferences")
("users", user_id, "memories")

# Further scoped by memory subtype, when a user's memory has distinct categories
# that you want to search or manage independently
("users", user_id, "memories", "dietary")
("users", user_id, "memories", "work_context")

# Multi-tenant / shared-agent scenarios: scope by organization AND user
("orgs", org_id, "users", user_id, "preferences")
```

### 4.2 Why namespace design matters: isolation

The most important property a namespace scheme must guarantee is **isolation** — a query scoped to `("users", "user-42", ...)` must be structurally incapable of returning `("users", "user-99", ...)` data. This isn't just about correctness; it's a privacy and security boundary. Two rules follow directly:

- **Never let `user_id` be optional or inferred late.** Derive it once, early (typically from `config["configurable"]["user_id"]`, set by your API layer from an authenticated session — never from the LLM's own output or from conversation content), and use that single source of truth to build every namespace tuple in every node.
- **Prefer namespace-level isolation over filtering after the fact.** `store.search(("users", "user-42", "memories"), query=...)` is safe by construction. `store.search(("memories",), query=..., filter={"user_id": "user-42"})` (a flat namespace with a filter bolted on) relies on every call site remembering to apply the filter correctly — one missed filter and you've leaked cross-user data.

### 4.3 Why namespace design matters: retrieval quality

Beyond isolation, namespace granularity directly affects **what gets compared against what** during `search()`. If every fact about a user — dietary restrictions, UI theme preference, a passing mention of their dog's name, a project deadline — lives in one giant `("users", user_id, "memories")` bucket, a semantic search for "what should I avoid recommending for dinner" has to rank the one relevant dietary memory against dozens of irrelevant ones, diluting precision and increasing the chance an irrelevant-but-superficially-similar memory outranks the correct one.

Splitting by category (`.../memories/dietary`, `.../memories/work_context`, `.../memories/social`) lets a node search only the relevant sub-namespace when it already knows the category it cares about, and search the parent namespace when it doesn't — `search` naturally includes everything nested beneath the prefix you pass. This is the long-term-memory analog of chunking strategy in RAG: how you partition the corpus directly shapes what a similarity search can and can't find efficiently.

---

## 5. Semantic Search Over Long-Term Memory

### 5.1 Exact-key lookup vs. semantic recall

`store.get(namespace, key)` only works if you already know the exact key — fine for structured settings (`"communication_style"`, `"timezone"`) where the key is a stable, known field name. It's useless for the far more common case of "recall whatever the user told us that's *relevant* to what they're asking right now," where you don't know in advance which of potentially hundreds of stored memory items applies. That's exactly the retrieval problem embeddings solve in RAG systems, applied here to a user's memory instead of a document corpus.

### 5.2 Configuring an embedding index on the store

Stores support an optional `index` configuration that tells them how to embed values on write and compare them on search:

```python
from langgraph.store.memory import InMemoryStore
from langchain_openai import OpenAIEmbeddings

store = InMemoryStore(
    index={
        "embed": OpenAIEmbeddings(model="text-embedding-3-small"),
        "dims": 1536,
        "fields": ["text"],   # which key(s) inside the stored value get embedded
    }
)
```

`fields` tells the store which field(s) of the `value` dict to run through the embedding model — you can store structured metadata alongside the embedded text without polluting the vector (e.g., a `created_at` or `source_thread_id` field that rides along but isn't itself embedded).

### 5.3 Writing and searching semantically

```python
import uuid

store.put(
    ("users", "user-42", "memories"),
    str(uuid.uuid4()),
    {"text": "User mentioned they are severely allergic to peanuts.", "category": "dietary"},
)
store.put(
    ("users", "user-42", "memories"),
    str(uuid.uuid4()),
    {"text": "User's favorite programming language is Rust.", "category": "work_context"},
)
store.put(
    ("users", "user-42", "memories"),
    str(uuid.uuid4()),
    {"text": "User is traveling to Lisbon next month for a conference.", "category": "personal"},
)

results = store.search(
    ("users", "user-42", "memories"),
    query="Is there anything I should be careful about when suggesting a restaurant?",
    limit=3,
)
for r in results:
    print(f"{r.score:.3f}  {r.value['text']}")
# 0.812  User mentioned they are severely allergic to peanuts.
# 0.203  User is traveling to Lisbon next month for a conference.
# 0.118  User's favorite programming language is Rust.
```

If you've worked with embeddings for RAG-style document retrieval, this should feel immediately familiar: `query` is embedded with the same model configured on `index["embed"]`, compared against every stored item's embedded `fields` using cosine similarity (or whatever metric the backend implements), and the top-`limit` matches come back ranked by `score` — the same embed → compare → rank pipeline, just applied to a user's remembered facts instead of document chunks. The peanut-allergy memory scores highest not because it shares words with the query, but because it's semantically about what to avoid when food is involved — precisely the "car repair" / "automobile maintenance" phenomenon you'd expect from any embedding-backed retrieval system.

Without an `index` configured, `store.search()` still works, but falls back to structural/filter-based matching only (or, depending on backend, a simple substring/metadata match) — `score` comes back as `None` and there's no notion of semantic relevance ranking. Configuring the index is what upgrades `search` from "list items matching this filter" to "find the memories that actually matter for this query."

---

## 6. Reading Short-Term and Long-Term Memory Together in One Node

This is where the two mechanisms combine into something genuinely useful. A single node can, in one execution:

1. Read the **short-term** state (`state["messages"]`) to know what's been said in *this* conversation.
2. Read the **long-term** store (via `search`/`get`) to recall facts about *this user* from *any* prior conversation.
3. Call the LLM with both as context.
4. Optionally write new facts back to the store for future turns to use — in any thread.

```python
from langchain_core.messages import SystemMessage
from langchain_core.runnables import RunnableConfig
from langgraph.store.base import BaseStore


def assistant_node(state: ChatState, config: RunnableConfig, *, store: BaseStore):
    user_id = config["configurable"]["user_id"]
    memory_namespace = ("users", user_id, "memories")
    prefs_namespace = ("users", user_id, "preferences")

    latest_user_message = state["messages"][-1].content

    # --- LONG-TERM READ: semantic recall of relevant facts about this user ---
    relevant_memories = store.search(memory_namespace, query=latest_user_message, limit=5)
    memory_block = "\n".join(f"- {m.value['text']}" for m in relevant_memories) or "(none yet)"

    # --- LONG-TERM READ: exact-key lookup of explicit preferences ---
    prefs_item = store.get(prefs_namespace, "settings")
    prefs = prefs_item.value if prefs_item else {"tone": "neutral", "verbosity": "normal"}

    system_prompt = (
        "You are a personal assistant for this user across all their conversations.\n"
        f"Respond in this style: {prefs}.\n"
        f"Known long-term facts about this user:\n{memory_block}"
    )

    # --- SHORT-TERM READ: the full running conversation for THIS thread ---
    response = llm.invoke([SystemMessage(system_prompt), *state["messages"]])

    # --- LONG-TERM WRITE: decide if this turn revealed something worth remembering ---
    decision = memory_extraction_llm.invoke(
        f"Conversation turn:\nUser: {latest_user_message}\nAssistant: {response.content}\n\n"
        "Does this turn reveal a durable fact about the user worth remembering "
        "for future conversations (e.g., a preference, constraint, or personal detail)? "
        "If yes, return the fact as one sentence. If no, return exactly: NONE."
    )
    if decision.content.strip() != "NONE":
        store.put(
            memory_namespace,
            str(uuid.uuid4()),
            {"text": decision.content.strip(), "source_thread": config["configurable"]["thread_id"]},
        )

    # --- SHORT-TERM WRITE: append the response to this thread's message list ---
    return {"messages": [response]}
```

Notice the four distinct memory operations happening in one node execution, and that they use two entirely different scoping keys: `config["configurable"]["thread_id"]` (implicitly, via the checkpointer, for `state["messages"]`) and `config["configurable"]["user_id"]` (explicitly, via the store namespace) for everything long-term. A single user can have many threads; the store doesn't care which thread produced a memory, only which user it belongs to (though, as shown above, it's often useful to tag the *source* thread in the stored value for traceability/debugging).

Compiling and invoking this graph requires both a checkpointer and a store, and a config carrying both keys:

```python
app = builder.compile(checkpointer=checkpointer, store=store)

config = {
    "configurable": {
        "thread_id": "sess-2026-07-24-a",   # scopes short-term memory
        "user_id": "user-42",                # scopes long-term memory
    }
}
app.invoke({"messages": [HumanMessage("I'm thinking about dinner tonight, any ideas?")]}, config)
```

A week later, in a brand-new thread (`thread_id="sess-2026-07-31-b"`), the same `user_id` means `store.search` still surfaces the peanut allergy memory extracted a week earlier — even though the checkpointer for this new thread starts with a completely empty `messages` list, exactly as Section 2.4 described.

---

## 7. What to Store Long-Term: Patterns and Anti-Patterns

Not everything belongs in long-term memory, and how you shape what you store matters as much as the store/retrieve mechanics themselves.

### 7.1 Explicit preferences (structured, exact-key)

Settings the user directly stated or configured — units, language, notification cadence, tone. These are best stored as **structured values under known keys**, retrieved with `get`, not `search`:

```python
store.put(
    ("users", user_id, "preferences"),
    "settings",
    {"units": "metric", "language": "en", "tone": "concise", "timezone": "Asia/Kolkata"},
)
```

Exact-key lookup here is a *feature*, not a limitation — you never want "what units does this user prefer" to be answered by a semantic search that might occasionally rank a different memory higher.

### 7.2 Extracted facts and summaries (unstructured, semantic)

Facts an LLM inferred or extracted from conversation — the "user mentioned X" pattern used in Section 6. The key design decision here is **when to extract**, not just what: running an extraction LLM call on every single turn is wasteful and noisy (most turns reveal nothing durable). Common patterns:

- **Inline, gated extraction** — as in Section 6, ask a cheap/fast model each turn "is this worth remembering," and only write on a positive answer.
- **End-of-conversation summarization** — run one extraction pass when a thread is closed/idle, summarizing the whole conversation into a handful of durable facts rather than evaluating turn-by-turn.
- **Explicit triggers** — only extract when the user says something that matches a pattern worth flagging (e.g., "remember that...", "for future reference...").

### 7.3 Raw conversation dumps (anti-pattern)

It's tempting to just `store.put` the entire message history of every thread into long-term memory "in case it's useful later." This breaks down for concrete, predictable reasons:

- **Retrieval quality collapses.** Semantic search over thousands of raw conversational turns (small talk, clarifying questions, "yes," "thanks") returns noisy results — the signal (a durable fact) is diluted by volume of near-duplicate, low-information text.
- **Storage and embedding cost scale without bound.** Every turn of every conversation, forever, embedded and stored, grows linearly with usage and never gets smaller — unlike a bounded set of extracted facts, which grows much more slowly (most turns produce zero new facts).
- **It pushes the hard problem downstream instead of solving it.** "What do we actually know about this user" is still a question you have to answer at read time, and now you have to answer it by searching a haystack of raw transcript instead of a curated set of facts — you've deferred the summarization work, not eliminated it.
- **Staleness and contradiction accumulate silently.** If a user says "I'm vegetarian" in March and "I started eating fish again" in June, a raw dump contains both with no mechanism to reconcile them; a curated fact store gives you a natural place to update or supersede the March entry when the June statement is extracted.

The practical rule: **store conclusions, not transcripts.** Long-term memory should read like a well-maintained user profile a human case worker might keep — a short, high-signal set of facts and preferences — not like a call center's full recording archive.

---

## Examples

### Project sketch: Personal Assistant (short-term + long-term combined)

Putting Sections 2, 3, 5, and 6 together into a complete, runnable-shape graph:

```python
import uuid
from typing import Annotated
from typing_extensions import TypedDict

from langchain_core.messages import AnyMessage, HumanMessage, SystemMessage
from langchain_core.runnables import RunnableConfig
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver
from langgraph.store.base import BaseStore
from langgraph.store.memory import InMemoryStore


class AssistantState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]


llm = ChatOpenAI(model="gpt-4o-mini")
extraction_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)


def assistant_node(state: AssistantState, config: RunnableConfig, *, store: BaseStore):
    user_id = config["configurable"]["user_id"]
    memory_ns = ("users", user_id, "memories")
    prefs_ns = ("users", user_id, "preferences")

    latest = state["messages"][-1].content

    memories = store.search(memory_ns, query=latest, limit=5)
    memory_block = "\n".join(f"- {m.value['text']}" for m in memories) or "(none yet)"

    prefs_item = store.get(prefs_ns, "settings")
    prefs = prefs_item.value if prefs_item else {}

    response = llm.invoke([
        SystemMessage(
            f"You are a helpful personal assistant.\n"
            f"User preferences: {prefs}\n"
            f"Known facts about this user:\n{memory_block}"
        ),
        *state["messages"],
    ])

    decision = extraction_llm.invoke(
        f"User said: {latest!r}\nAssistant replied: {response.content!r}\n"
        "In one short sentence, state a durable fact worth remembering about the "
        "user for future conversations, or reply exactly NONE if nothing qualifies."
    )
    if decision.content.strip().upper() != "NONE":
        store.put(memory_ns, str(uuid.uuid4()), {"text": decision.content.strip()})

    return {"messages": [response]}


builder = StateGraph(AssistantState)
builder.add_node("assistant", assistant_node)
builder.add_edge(START, "assistant")
builder.add_edge("assistant", END)

store = InMemoryStore(
    index={"embed": OpenAIEmbeddings(model="text-embedding-3-small"), "dims": 1536, "fields": ["text"]}
)
checkpointer = MemorySaver()
app = builder.compile(checkpointer=checkpointer, store=store)

config = {"configurable": {"thread_id": "sess-001", "user_id": "user-42"}}
app.invoke({"messages": [HumanMessage("I'm severely allergic to peanuts, just so you know.")]}, config)

# ... days later, a brand-new thread, same user ...
config_new_thread = {"configurable": {"thread_id": "sess-088", "user_id": "user-42"}}
app.invoke({"messages": [HumanMessage("Can you suggest a snack for my flight?")]}, config_new_thread)
# The assistant recalls the peanut allergy via long-term search, despite sess-088
# never having seen that message in its own (empty) short-term history.
```

### Project sketch: User Preference Memory (settings that persist across sessions)

A narrower, common building block: a dedicated node (or even a plain function called from a FastAPI route, outside the graph entirely) that manages structured preferences independent of any conversation:

```python
def get_user_preferences(store: BaseStore, user_id: str) -> dict:
    item = store.get(("users", user_id, "preferences"), "settings")
    return item.value if item else {"tone": "neutral", "units": "metric", "language": "en"}


def update_user_preference(store: BaseStore, user_id: str, key: str, value) -> None:
    current = get_user_preferences(store, user_id)
    current[key] = value
    store.put(("users", user_id, "preferences"), "settings", current)


def preferences_node(state: AssistantState, config: RunnableConfig, *, store: BaseStore):
    """Detects explicit preference statements ('please always reply in Spanish')
    and persists them immediately, separate from the general fact-extraction path."""
    user_id = config["configurable"]["user_id"]
    latest = state["messages"][-1].content

    parsed = extraction_llm.invoke(
        f"User message: {latest!r}\n"
        "If this message states an explicit preference/setting change (language, "
        "tone, units, notification frequency, etc.), return it as JSON: "
        '{"key": "...", "value": "..."}. Otherwise return exactly NONE.'
    )
    if parsed.content.strip().upper() != "NONE":
        import json
        pref = json.loads(parsed.content)
        update_user_preference(store, user_id, pref["key"], pref["value"])

    return {}
```

Because `preferences` lives under its own namespace (`("users", user_id, "preferences")`, distinct from `("users", user_id, "memories")`), a settings read (`get`, exact key, no embedding cost) never competes with or gets diluted by semantic memory search, and vice versa — a direct illustration of the namespace-design principle from Section 4.

---

## Diagrams

### Short-term vs. long-term memory: where each lives

```mermaid
flowchart TB
    subgraph Runtime["Single graph invocation"]
        CFG["config = {'configurable': {'thread_id': ..., 'user_id': ...}}"]
        NODE["Node function\n(state, config, *, store)"]
    end

    CFG -->|thread_id| CKPT[("Checkpointer\nMemorySaver / SqliteSaver / PostgresSaver")]
    CFG -->|user_id| STORE[("Store\nInMemoryStore / PostgresStore")]

    CKPT -->|"load/save state\nkeyed by thread_id"| NODE
    STORE -->|"put / get / search\nkeyed by namespace tuple"| NODE

    NODE --> ST_OUT["Short-term result:\nupdated messages[] for THIS thread only"]
    NODE --> LT_OUT["Long-term result:\nnew/updated facts for THIS user,\nvisible from ANY thread"]

    style CKPT fill:#e1f5ff
    style STORE fill:#fff3e0
```

### One turn, four memory operations (Section 6 sequence)

```mermaid
sequenceDiagram
    participant U as User
    participant N as assistant_node
    participant CP as Checkpointer (thread_id)
    participant ST as Store (user_id namespace)
    participant L as LLM

    U->>N: New message (via .invoke, same thread_id)
    CP->>N: Load prior messages[] for this thread
    N->>ST: search(("users", uid, "memories"), query=latest_message)
    ST-->>N: Top-K relevant memories (semantic)
    N->>ST: get(("users", uid, "preferences"), "settings")
    ST-->>N: Preference dict (exact key)
    N->>L: Invoke with system prompt (prefs + memories) + full thread history
    L-->>N: Response
    N->>L: "Extract a durable fact, or NONE"
    L-->>N: Fact or NONE
    alt fact extracted
        N->>ST: put(("users", uid, "memories"), new_id, {text: fact})
    end
    N->>CP: Return {"messages": [response]} -> checkpointed for this thread
    N-->>U: Response
```

---

## Real-World Scenarios

**Scenario 1 — Customer support bot that "forgets" mid-ticket.** A support team builds a LangGraph-based bot and wires up a checkpointer, but every message from the widget arrives with a fresh, randomly generated `thread_id` because the frontend engineer assumed `thread_id` was just "a request ID." Every reply loses all prior context, and the bot repeatedly asks customers to re-explain their issue. The fix has nothing to do with long-term memory — it's a short-term memory bug: the frontend needs to generate `thread_id` once per support ticket (e.g., using the ticket ID itself) and reuse it for every message in that ticket, not per HTTP request.

**Scenario 2 — Preferences that should have been long-term but were made short-term.** A team stores a user's preferred response language inside the graph state (`state["language"]`) rather than the store. It works fine within one thread, but every time the user starts a new conversation (new `thread_id` — e.g., a new day, or a new device), the language preference resets to the default and the user has to restate it. The fix: move `language` out of thread-scoped state and into a `("users", user_id, "preferences")` store entry, read once at the top of the graph regardless of which thread is running.

**Scenario 3 — A memory store that leaks across users.** An engineer prototypes long-term memory with a single flat namespace, `("memories",)`, storing `{"user_id": ..., "text": ...}` and filtering by `user_id` at query time. During a demo, a `search` call omits the filter by accident (a copy-pasted query from a debugging script), and one user's private medical note is briefly displayed to another user in a live session. Post-incident, the team switches to namespace-level isolation — `("users", user_id, "memories")` — so that omitting the correct namespace produces an empty result (or an outright key error), not another user's data, making the failure mode "nothing comes back" instead of "the wrong thing comes back."

**Scenario 4 — Retrieval quality degrading as memory grows.** A personal assistant that's been running for a user for six months has accumulated 4,000 raw extracted "facts," many near-duplicates ("user likes coffee," "user mentioned enjoying coffee again," "user ordered a latte") because the extraction prompt was too permissive. Semantic search increasingly returns a jumble of near-identical low-value memories crowding out the one memory that actually matters for the current query. The fix combines a stricter extraction prompt (only genuinely new, durable, non-redundant facts), periodic deduplication/consolidation passes, and splitting the memory namespace by category (Section 4.3) so searches can be scoped more narrowly when the category is known.

---

## Best Practices

- **Never conflate `thread_id` and `user_id`.** `thread_id` scopes short-term memory (one conversation); `user_id` scopes long-term memory (one person across all conversations). Pass both explicitly in `config["configurable"]` and derive both from your authenticated request context — never let the LLM or the message content set either one.
- **Design namespaces before you write any memory code**, the same way you'd design database table/partition keys up front. Prefer `("users", user_id, ...)` over a flat namespace with a `user_id` filter field — isolation should be structural, not a filter someone can forget to apply.
- **Use `get` for known, structured keys; use `search` for open-ended recall.** Don't semantically search for something you already know the exact key for (wastes an embedding call and introduces unnecessary ranking uncertainty); don't try to `get` something you don't have a stable key for.
- **Gate long-term writes.** Don't extract-and-store on every turn unconditionally — use a cheap classification/extraction step to decide *whether* a turn is worth persisting, and prefer distilled facts over raw transcript.
- **Bound short-term memory growth deliberately** (trimming or periodic summarization) rather than letting `state["messages"]` grow without limit — it's both a cost problem and, eventually, a context-window correctness problem.
- **Version and audit what you extract.** Store `source_thread_id` and a timestamp alongside long-term facts so you can trace where a memory came from and build a path to let users (or you, during debugging) inspect, correct, or delete it.
- **Treat memory writes as data with the same privacy sensitivity as the conversation itself.** Long-term memory is often more sensitive than any single conversation, because it aggregates facts across every conversation a user has ever had — apply access control, retention policy, and deletion (e.g., "forget me") support accordingly.
- **Pick the store backend to match your deployment**, exactly as you did with checkpointers in Chapter 9: `InMemoryStore` for local development and tests, a persistent backend (`PostgresStore` or equivalent) for anything that must survive a process restart.

---

## Common Mistakes

- **Assuming the checkpointer alone gives you cross-session memory.** It doesn't — it gives you cross-*invocation*, same-*thread* memory. A new `thread_id` is, by design, a blank slate. Long-term persistence requires the Store, deliberately wired with a stable identifier like `user_id`.
- **Storing long-term facts under a namespace keyed only by `thread_id`.** This silently reduces the "long-term" store to thread-scoped memory in disguise — the moment a new thread starts, `search`/`get` calls against `(thread_id, ...)` come back empty, because that thread has no history in the store either.
- **Letting namespace values come from unauthenticated or LLM-controlled input.** If `user_id` in the namespace tuple is ever derived from something the model can influence (e.g., extracted from message content instead of the session's auth context), a crafted input can make the node read or write another user's memory namespace.
- **Treating `search` as guaranteed-relevant without configuring an embedding index.** Without `index` configured on the store, `search` has no semantic ranking capability — silently falling back to structural or no-op matching, which looks like it "works" in a quick test with one item but breaks down the moment there are many.
- **Extracting and storing on every single turn, unconditionally.** This floods long-term memory with low-signal, near-duplicate entries, which both inflates storage/embedding cost and actively degrades retrieval quality for the memories that matter (Section 7.3 / Scenario 4).
- **Dumping full raw conversation transcripts into long-term memory "just in case."** As covered in Section 7.3, this doesn't scale, dilutes retrieval quality, and defers rather than solves the "what do we actually know about this user" problem.
- **Forgetting to scope preferences and memories into separate namespaces**, causing exact-key preference lookups and semantic memory search to interfere with (or be needlessly slowed down by) each other.
- **Never revisiting or consolidating stored memories.** Facts go stale (a preference changes, a fact is superseded) — a long-term memory system with no update/consolidation/expiry path accumulates contradictions indefinitely.

---

## Summary

- LangGraph splits memory into two purpose-built subsystems: **short-term memory**, which is graph state persisted by the checkpointer and scoped to a single `thread_id` (Chapter 9's mechanism, viewed through a memory lens), and **long-term memory**, which is explicit, cross-thread, cross-session persistence scoped to a user (or other durable entity) via the **Store**.
- The **Store interface** (`InMemoryStore` for development, `PostgresStore`/`AsyncPostgresStore` for production) exposes four operations — `put`, `get`, `search`, `delete` — addressed by hierarchical **namespace** tuples you design, not a single flat keyspace.
- **Namespace design** is a first-class engineering decision: it determines isolation between users (structural safety, not filter-based safety) and retrieval quality (how much irrelevant material a semantic search has to rank against).
- A store configured with an **embedding `index`** upgrades `search` from filter-matching into genuine **semantic recall** — the same embed-query, compare-by-similarity, rank-by-score mechanism you already know from RAG retrieval, applied to a user's remembered facts instead of document chunks.
- A single node can read **both** memory types in one execution — short-term `state["messages"]` for this conversation, long-term `store.search`/`store.get` for durable facts about this user — and write back to both, which is the pattern behind any assistant that "remembers you" across sessions.
- What you persist long-term matters as much as how: prefer **explicit structured preferences** (exact-key) and **LLM-extracted, gated facts/summaries** (semantic) over **raw conversation dumps**, which scale poorly and actively hurt retrieval quality as volume grows.

---

## Knowledge Check

1. A user reports that the assistant "forgot" their name between two messages sent five minutes apart in what they consider the same conversation. What are the two most likely root causes, given everything you now know about `thread_id` and the checkpointer, and how would you distinguish between them?
2. Explain why `store.get(namespace, key)` and `store.search(namespace, query=...)` are not interchangeable, and give one example of information that belongs behind each.
3. A colleague proposes a single flat namespace, `("all_memories",)`, storing a `user_id` field inside each value and filtering by it at query time, instead of `("users", user_id, "memories")`. What specific risk does this introduce, and why does namespace-level isolation avoid it structurally rather than by convention?
4. Why doesn't storing the full raw transcript of every conversation as long-term memory scale, even if storage cost were free? Name at least two distinct problems beyond storage cost.
5. Walk through, in order, everything that happens — both short-term and long-term — when `assistant_node` from Section 6 executes for a returning user in a brand-new thread. What's loaded, what's searched, and what (if anything) gets written back to each subsystem?
6. What does configuring `index={"embed": ..., "fields": [...]}` on a store actually change about the behavior of `search()`, compared to a store with no `index` configured?

---

## Hands-on Exercises

1. **Personal Assistant.** Build the graph from the Examples section end-to-end: a `StateGraph` with one node that reads short-term `messages` and long-term memories/preferences, calls an LLM, and conditionally writes a new long-term memory. Compile with both a `MemorySaver` and an `InMemoryStore` (embedding index configured). Simulate: (a) two messages in the same thread where the second correctly uses short-term context from the first; (b) a brand-new thread, same `user_id`, where the assistant still recalls a fact extracted during (a). Print out exactly what `store.search` returns at each step to verify the recall is working as intended.

2. **User Preference Memory.** Implement the `preferences_node` pattern from the Examples section as a standalone component. Feed it a sequence of messages, some containing explicit preference statements ("please keep answers short from now on," "reply in French going forward") and some not. Verify: (a) preference-bearing messages update the `("users", user_id, "preferences")` namespace via `get`/`put`, (b) non-preference messages leave it untouched, and (c) a preference set in one thread is visible and applied when a different thread starts for the same user.

3. **Namespace isolation stress test.** Set up an `InMemoryStore` with memories for two distinct users under `("users", "user-A", "memories")` and `("users", "user-B", "memories")`. Write a function that takes a `user_id` and a query and calls `store.search` scoped to that user's namespace only. Deliberately try to construct a call (e.g., a wrong or mismatched namespace tuple) that would leak user-B's data into a search intended for user-A, observe that it either returns nothing or raises, and write a short note on what code-level guardrail (e.g., a single trusted helper function that always builds the namespace from an authenticated `user_id`, never called with a raw tuple elsewhere in the codebase) would prevent this class of mistake in a real application.

---

## Further Reading

- [LangGraph Documentation — Memory](https://docs.langchain.com/oss/python/langgraph/overview) — official conceptual overview of short-term vs. long-term memory in LangGraph
- [LangGraph Documentation — Persistence](https://docs.langchain.com/oss/python/langgraph/application-structure) — checkpointer and store configuration as part of application structure
- [LangGraph GitHub Repository](https://github.com/langchain-ai/langgraph) — source for `langgraph.store.memory`, `langgraph.store.postgres`, and the `BaseStore` interface definitions
- [LangSmith Documentation](https://docs.smith.langchain.com/) — useful for tracing and inspecting what a node actually read from/wrote to a store during a run, covered operationally in Chapter 20
- Related chapter in this course: **Chapter 9 — Checkpointing & Durable Execution** — the mechanism short-term memory is built on
- Related chapter in this course: **Chapter 6 — Reducers** — how `add_messages` (and `RemoveMessage`) shape the running message list discussed in Section 2
- Related course in this repo: the RAG course's embeddings chapter — for a deeper treatment of cosine similarity, embedding models, and semantic search mechanics that underlie Section 5 of this chapter

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./09-checkpointing-and-durable-execution.md">← Previous: Checkpointing & Durable Execution</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./11-streaming.md">Next: Streaming →</a>
</div>
