# 🦜 LangChain Interview Q&A

## 🔹 Fundamentals

### 1. What is LangChain?
LangChain is an **open-source framework** for building applications powered by LLMs. It provides abstractions for chaining together LLM calls, prompts, tools, memory, and external data sources (retrievers, vector stores, APIs) into composable pipelines/agents.

---

### 2. Why do we need a framework like LangChain instead of calling the LLM API directly?
- Standardizes common patterns: prompting, output parsing, retries, streaming
- Provides ready-made integrations (100+ LLMs, vector stores, tools, loaders)
- Composability via chains/LCEL instead of hand-rolled glue code
- Built-in support for memory, agents, and tool calling
- Observability/tracing via LangSmith

---

### 3. What are the core building blocks of LangChain?
- **Models** – LLMs / Chat Models / Embedding models
- **Prompts** – PromptTemplates
- **Chains** – sequences of calls (LLM, tool, parser)
- **Memory** – conversation state across turns
- **Retrievers / Vector Stores** – for RAG
- **Agents** – LLM decides which tools to call and in what order
- **Callbacks** – hooks for logging, streaming, tracing

---

### 4. What is the difference between an LLM and a Chat Model in LangChain?
| LLM | Chat Model |
|----|----|
| Takes a raw string, returns a string | Takes a list of messages (System/Human/AI), returns a message |
| e.g. legacy `OpenAI` wrapper | e.g. `ChatOpenAI`, `ChatAnthropic` |
| Simpler completion-style APIs | Standard interface for modern chat-tuned LLMs |

---

### 5. What is a PromptTemplate?
A reusable, parameterized template for constructing prompts with placeholder variables, e.g.:
```python
from langchain_core.prompts import PromptTemplate
template = PromptTemplate.from_template("Translate '{text}' to {language}")
prompt = template.format(text="Hello", language="French")
```

---

### 6. What is a ChatPromptTemplate?
Like a PromptTemplate but constructs a **list of chat messages** (System, Human, AI, Placeholder) instead of a single string — used with Chat Models.
```python
from langchain_core.prompts import ChatPromptTemplate
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    ("human", "{question}")
])
```

---

### 7. What is an OutputParser?
A component that converts the LLM's raw text output into a **structured format** (JSON, list, Pydantic object, etc.), e.g. `StrOutputParser`, `JsonOutputParser`, `PydanticOutputParser`.

---

### 8. What is LCEL (LangChain Expression Language)?
A declarative syntax for composing LangChain components using the pipe operator `|`, where the output of one component feeds into the next — similar to Unix pipes.
```python
chain = prompt | model | output_parser
result = chain.invoke({"question": "What is LangChain?"})
```

---

### 9. Why was LCEL introduced (vs older `Chain` classes like `LLMChain`)?
- Unified `Runnable` interface across all components (sync/async, streaming, batch)
- Automatic support for `.invoke()`, `.batch()`, `.stream()`, `.ainvoke()` on any chain
- Easier composition, parallelism (`RunnableParallel`), and branching (`RunnableBranch`)
- Native tracing/observability with LangSmith
- Note: legacy `Chain` classes (e.g. `LLMChain`, `SequentialChain`) are now deprecated in favor of LCEL

---

### 10. What is the `Runnable` interface?
The core protocol in LangChain (LCEL) that every component (prompt, model, parser, retriever) implements, exposing a consistent set of methods: `invoke`, `batch`, `stream`, `ainvoke`, `abatch`, `astream` — enabling any two Runnables to be composed with `|`.

---

## 🔹 Chains

### 11. What is a Chain in LangChain?
A sequence of calls — to an LLM, a tool, a data source, or another chain — combined to accomplish a task, where the output of one step becomes the input of the next.

---

### 12. What is `RunnableSequence`?
The LCEL object created when you pipe Runnables together with `|`; it runs each step in order, passing output to input.

---

### 13. What is `RunnableParallel`?
An LCEL construct that runs multiple Runnables **concurrently** on the same input and returns a dict of their outputs — useful for fetching context from multiple sources at once (e.g. running a retriever and a question-passthrough in parallel before a prompt).
```python
from langchain_core.runnables import RunnableParallel, RunnablePassthrough
chain = RunnableParallel(context=retriever, question=RunnablePassthrough())
```

---

### 14. What is `RunnablePassthrough`?
A Runnable that simply **forwards its input unchanged** (optionally merging additional keys), commonly used in RAG chains to pass the original question through alongside retrieved context.

---

### 15. What is `RunnableBranch`?
A Runnable that implements **conditional routing** — it evaluates a list of (condition, runnable) pairs and executes the first branch whose condition is true, falling back to a default.

---

### 16. What is `RunnableLambda`?
A wrapper that turns any plain Python function into a Runnable, so custom logic can be inserted into an LCEL chain.

---

### 17. What was `SimpleSequentialChain` / `SequentialChain` (legacy)?
Legacy (pre-LCEL) chain classes for running steps in sequence — `SimpleSequentialChain` for single input/output per step, `SequentialChain` for multiple named inputs/outputs. Now superseded by LCEL's `|` composition.

---

### 18. What is a `RetrievalQA` chain (legacy)?
A legacy prebuilt chain that combines a retriever and an LLM to answer questions using retrieved documents — the classic RAG chain. Modern code builds the equivalent using LCEL: `retriever | prompt | model | parser`.

---

### 19. What is `create_stuff_documents_chain`?
A helper that builds a chain which **"stuffs"** all retrieved documents directly into a single prompt (as one big context block) before passing to the LLM — simplest document-combination strategy, works well when documents fit the context window.

---

### 20. What are the document-combination strategies for QA over many documents?
- **Stuff** – concatenate all docs into one prompt (simple, limited by context window)
- **Map-Reduce** – summarize/answer each doc individually ("map"), then combine those answers ("reduce")
- **Refine** – iteratively update an answer by feeding it and the next document, one doc at a time
- **Map-Rerank** – answer from each doc individually, score confidence, return the highest-scoring answer

---

## 🔹 Memory

### 21. What is Memory in LangChain, and why is it needed?
LLM calls are stateless by default — Memory components **persist conversation history/state** across turns so a chatbot can reference earlier messages.

---

### 22. What are common Memory types in LangChain?
- **ConversationBufferMemory** – stores the full raw conversation history
- **ConversationBufferWindowMemory** – keeps only the last *k* interactions
- **ConversationSummaryMemory** – summarizes older history using an LLM to save tokens
- **ConversationSummaryBufferMemory** – hybrid: keeps recent messages verbatim + summarizes older ones
- **ConversationTokenBufferMemory** – trims history based on a token limit
- **VectorStoreRetrieverMemory** – stores/retrieves relevant past messages via semantic similarity

---

### 23. How is memory handled in modern LCEL-based chains?
Via `RunnableWithMessageHistory`, which wraps a chain and automatically loads/saves chat history (keyed by a `session_id`) using a `BaseChatMessageHistory` implementation (in-memory, Redis, SQL, etc.), replacing the older `ConversationChain` + Memory classes.

---

### 24. Why prefer summary memory over buffer memory for long conversations?
Buffer memory grows unbounded, eventually exceeding the context window and increasing cost/latency. Summary memory keeps a **condensed representation** of older turns, bounding token growth at the cost of some detail loss.

---

## 🔹 RAG, Retrievers & Vector Stores

### 25. What is a Retriever in LangChain?
An abstraction (`BaseRetriever`) that takes a query string and returns a list of relevant `Document` objects — implementations include vector store retrievers, BM25, multi-query retrievers, and ensemble retrievers.

---

### 26. What is a VectorStore in LangChain?
An abstraction over a vector database (FAISS, Chroma, Pinecone, Weaviate, Milvus, pgvector) that stores document embeddings and supports similarity search. Any VectorStore can expose itself `.as_retriever()`.

---

### 27. What are Document Loaders?
Components that load raw data from various sources (PDFs, web pages, CSVs, Notion, databases, S3) into LangChain's standard `Document` object (page_content + metadata).

---

### 28. What are Text Splitters, and why are they needed?
Utilities that break large documents into smaller chunks before embedding, since (a) embedding models have input limits and (b) smaller, focused chunks improve retrieval precision. Common ones:
- `CharacterTextSplitter`
- `RecursiveCharacterTextSplitter` (most commonly used — splits on paragraph → sentence → word as needed)
- `TokenTextSplitter`
- `MarkdownHeaderTextSplitter` / semantic splitters

---

### 29. What is chunk overlap and why use it?
Including a small overlapping portion of text between consecutive chunks so that context/meaning **isn't cut off at chunk boundaries**, improving retrieval quality for information spanning a boundary.

---

### 30. What is a MultiQueryRetriever?
A retriever that uses an LLM to **generate multiple reformulations** of the user's query, retrieves documents for each, and takes the union — improving recall when a single query phrasing might miss relevant documents.

---

### 31. What is an EnsembleRetriever?
A retriever that combines results from **multiple retrievers** (e.g. a keyword-based BM25 retriever + a dense vector retriever) using a weighted reciprocal rank fusion, balancing keyword precision with semantic recall (hybrid search).

---

### 32. What is a Self-Query Retriever?
A retriever that uses an LLM to translate a natural-language query into a **structured query** (semantic search term + metadata filters), e.g. turning "cheap sci-fi movies after 2015" into a vector search plus `genre=sci-fi AND year>2015` filter.

---

### 33. What is a Parent Document Retriever?
A retrieval strategy that indexes **small chunks** for precise similarity search but returns the **larger parent document/chunk** they belong to, giving the LLM more surrounding context than the small chunk alone.

---

### 34. How would you build a basic RAG chain in LangChain (LCEL)?
```python
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | model
    | StrOutputParser()
)
rag_chain.invoke("What is LangChain?")
```

---

### 35. What is re-ranking in a RAG pipeline, and why use it?
A second-stage step where an initial (fast, approximate) retrieval of many candidate documents is **re-scored by a more accurate but slower model** (e.g. a cross-encoder), and only the top re-ranked results are passed to the LLM — improving relevance without paying the cost of running the expensive model over the entire corpus.

---

## 🔹 Agents & Tools

### 36. What is an Agent in LangChain?
A system where an LLM **decides which tools to call, in what order, and with what inputs**, based on the user's request, iterating until it can produce a final answer — as opposed to a fixed, hard-coded chain.

---

### 37. What is a Tool in LangChain?
A callable (function/API) with a name, description, and input schema that an agent can invoke. The LLM chooses tools based on their **description**, so descriptions must clearly state what the tool does and when to use it.

---

### 38. How do you define a custom tool?
```python
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get the current weather for a given city."""
    return f"Weather in {city}: Sunny, 25°C"
```
The `@tool` decorator infers the name/schema from the function signature and uses the docstring as the tool description shown to the LLM.

---

### 39. What is the ReAct pattern, and how does it relate to LangChain agents?
**Re**asoning + **Act**ing — the LLM alternates between generating a reasoning trace ("Thought"), choosing an action/tool ("Action"), observing the tool's result ("Observation"), and repeating until it reaches a final answer. Many LangChain agent types (`create_react_agent`) implement this loop.

---

### 40. What is Function/Tool Calling, and how does LangChain use it?
A capability of modern LLMs (OpenAI, Anthropic, etc.) to output a structured JSON call (tool name + arguments) instead of free text. LangChain's tool-calling agents rely on this native capability instead of parsing free-form ReAct text, making tool selection **more reliable**.

---

### 41. What is `AgentExecutor`?
The runtime loop that drives an agent: it takes the agent's chosen action, executes the corresponding tool, feeds the observation back to the agent, and repeats until the agent returns a final answer or a stopping condition (e.g. `max_iterations`) is hit.

---

### 42. How do you prevent an agent from looping indefinitely or misbehaving?
- Set `max_iterations` / `max_execution_time` on `AgentExecutor`
- Use `handle_parsing_errors=True` to gracefully recover from malformed outputs
- Constrain the toolset to only what's necessary
- Add guardrails/validation on tool inputs and outputs
- Use human-in-the-loop approval for high-risk tool calls

---

### 43. What is LangGraph, and how does it relate to LangChain?
LangGraph is a library (built by the LangChain team) for building **stateful, multi-actor applications** as a graph of nodes and edges, giving explicit control over branching, loops, and cycles — useful for complex agentic workflows that simple `AgentExecutor` loops can't express cleanly (e.g. multi-agent systems, human-in-the-loop approval steps, cyclic reasoning).

---

### 44. When would you use LangGraph instead of a standard LangChain agent?
When you need **explicit control flow** — conditional branching, cycles, persistence/checkpointing of state, multi-agent coordination, or resumable long-running workflows — rather than a purely LLM-driven "decide next tool" loop.

---

## 🔹 Observability, Testing & Production

### 45. What is LangSmith?
LangChain's observability/tracing platform for **debugging, monitoring, evaluating, and testing** LLM applications — it captures every step of a chain/agent run (inputs, outputs, latency, token usage) for inspection.

---

### 46. What are Callbacks in LangChain?
Hooks that let you tap into events during a chain/agent run (e.g. `on_llm_start`, `on_tool_end`, `on_chain_error`) — used for logging, streaming tokens to a UI, custom metrics, or integrating with tracing tools like LangSmith.

---

### 47. How does streaming work in LangChain?
Runnables expose a `.stream()` (and `.astream()`) method that yields output incrementally (token-by-token for chat models) instead of waiting for the full response — important for responsive UIs on long generations.

---

### 48. How do you handle structured output reliably in LangChain?
- `with_structured_output(schema)` on a chat model (uses native function/tool calling under the hood when supported)
- `PydanticOutputParser` / `JsonOutputParser` with format instructions embedded in the prompt
- Adding retry/fixing parsers (`OutputFixingParser`, `RetryOutputParser`) that ask the LLM to correct malformed output

---

### 49. How do you reduce hallucination and improve reliability in LangChain apps?
- Ground responses with RAG instead of relying on parametric knowledge
- Use structured output parsing with validation/retry
- Add re-ranking to improve retrieved-context relevance
- Lower temperature for factual tasks
- Add evaluation via LangSmith datasets/evaluators before shipping changes

---

### 50. What are common failure points / pitfalls when building LangChain apps in production?
- Unbounded memory/context growth → rising cost & latency
- Poor chunking strategy → irrelevant retrieved context
- Vague tool descriptions → agent picks the wrong tool
- No tracing/observability → hard to debug multi-step chains
- No caching → repeated identical LLM calls waste cost
- Version drift — LangChain's API surface changes frequently between versions; pin versions and check migration guides
- Treating agents as fully deterministic — always add guardrails, timeouts, and fallbacks for production use

---
