# Chapter 15: Architecture & Internals

> "Composition is the art of building complexity from a small number of things that all speak the same language." — paraphrased from countless systems-design retrospectives

## Learning Objectives

By the end of this chapter, you will be able to:

- State the full method surface of the `Runnable` abstract base class and explain why every LCEL component (prompts, models, parsers, retrievers, your own functions) implements it
- Explain precisely what `__or__` does when you write `prompt | model | parser` — and why nothing executes at that line
- Trace, step by step, what happens inside `RunnableSequence.invoke(input)`
- Explain why `.batch()` is not a for-loop of `.invoke()` calls, and how parallelism composes across a sequence
- Explain why `.stream()` produces "streaming for free" across a composed chain, by reasoning about generators chained together
- Describe the role of `RunnableConfig` and why silently dropping it in a custom component is a correctness bug, not a style nitpick
- Explain the `RunnableBinding` decorator pattern behind `.bind()`, `.with_retry()`, `.with_fallbacks()`, and `.with_config()`
- Explain how `RunnableParallel` fans a single input out to multiple branches and reassembles a dict of results

---

## Prerequisites for This Chapter

This chapter assumes you are comfortable with the LCEL basics from **Chapter 6**: you've written `prompt | model | parser` chains, called `.invoke()`, `.stream()`, and `.batch()` on them, and you've used `RunnableLambda`, `RunnableParallel`, and `RunnablePassthrough` as building blocks without necessarily knowing what happens underneath. You should also have just finished **[Chapter 14: Error Handling & Resilience](./14-error-handling-and-resilience.md)**, where you used `.with_retry()` and `.with_fallbacks()` to make chains resilient — this chapter explains the mechanism that makes those wrapper methods possible.

Nothing in this chapter requires new packages or new imports. Every code block is illustrative pseudocode or simplified real LangChain Core source, reasoned through on paper — there is nothing to run.

By design, this is the chapter promised back in Chapter 6: "we'll explain how this actually works later." This is later.

---

## 1. The Problem: Why Does `prompt | model | parser` Even Work?

### 1.1 The question you've been deferring since Chapter 6

Since Chapter 6 you've written code like this dozens of times:

```python
chain = prompt | model | parser
result = chain.invoke({"topic": "black holes"})
```

Three completely different kinds of object — a `ChatPromptTemplate`, a chat model, an output parser — get connected with the same `|` operator, and the result behaves as one thing: it has `.invoke()`, `.batch()`, `.stream()`, all working correctly, all the way through. There is no special-casing anywhere in your code for "the middle step is an LLM call and might be slow" or "the first step is deterministic template filling and the last step is pure Python." Every step is treated identically by the machinery.

That uniformity isn't an accident of naming. It's the direct consequence of a single design decision: **everything in LangChain Core that can appear in a chain implements the exact same abstract interface, called `Runnable`.** Once you understand that interface and the small number of `Runnable` subclasses built from it, the entire behavior of LCEL — composition, batching, streaming, config propagation, retries — stops being "framework magic" and becomes a straightforward, occasionally surprising, but fully predictable consequence of a few classes calling each other.

### 1.2 The core insight in one sentence

> **LCEL is not a DSL that gets parsed or compiled. It's ordinary Python objects, of a common shape, wired together at construction time by `__or__`, that know how to call each other at execution time.**

There is no LCEL "compiler." There is no intermediate representation you can't inspect. `prompt | model | parser` produces a plain Python object — a `RunnableSequence` — sitting in memory, holding a list of three references. Everything downstream in this chapter is the story of what that object does with those three references when you call `.invoke()`, `.batch()`, or `.stream()` on it.

---

## 2. The `Runnable` Abstract Base Class

### 2.1 The method surface

Every component you have piped together since Chapter 6 — `ChatPromptTemplate`, every chat model class, `StrOutputParser` and friends, `RunnableLambda`, `RunnableParallel`, retrievers, tools — inherits from `Runnable`. Simplified to the methods that matter for this chapter, the contract looks like this:

```python
from abc import ABC, abstractmethod
from typing import Any, AsyncIterator, Iterator, Optional

class Runnable(ABC):
    # --- the one method every subclass MUST implement ---
    @abstractmethod
    def invoke(self, input: Any, config: Optional[RunnableConfig] = None) -> Any:
        """Transform a single input into a single output, synchronously."""
        ...

    # --- everything below has a DEFAULT implementation in terms of invoke(),
    #     which subclasses are free to override for efficiency ---

    async def ainvoke(self, input: Any, config: Optional[RunnableConfig] = None) -> Any:
        """Async version of invoke(). Default: runs invoke() in a thread pool."""
        ...

    def batch(self, inputs: list[Any], config: Optional[RunnableConfig] = None) -> list[Any]:
        """Invoke on a list of inputs. Default: parallelizes invoke() across a thread pool."""
        ...

    async def abatch(self, inputs: list[Any], config: Optional[RunnableConfig] = None) -> list[Any]:
        """Async version of batch(). Default: gathers ainvoke() calls concurrently."""
        ...

    def stream(self, input: Any, config: Optional[RunnableConfig] = None) -> Iterator[Any]:
        """Stream output incrementally. Default: yields the single invoke() result once."""
        ...

    async def astream(self, input: Any, config: Optional[RunnableConfig] = None) -> AsyncIterator[Any]:
        """Async streaming. Default: yields the single ainvoke() result once."""
        ...

    async def astream_events(self, input: Any, config: Optional[RunnableConfig] = None):
        """Streams fine-grained lifecycle events (chain start, llm token, chain end, ...)
        for observability — the mechanism behind tracing dashboards."""
        ...

    def __or__(self, other: "Runnable") -> "RunnableSequence":
        """The pipe operator: compose self and other into a RunnableSequence."""
        ...

    def __ror__(self, other: "Runnable") -> "RunnableSequence":
        """Supports piping when self is on the right: other | self."""
        ...
```

Nothing here is exotic. It's the same shape you already know from any well-designed sync/async dual API (think of it as similar in spirit to how `requests` and `httpx` both expose a synchronous and an async client with matching methods) — except here the *same class* exposes both, plus batch and streaming variants, plus a composition operator.

### 2.2 Why uniformity is the whole point

Think about what would happen *without* this shared interface. If `ChatPromptTemplate` had a method called `.format()`, the chat model had a method called `.generate()`, and the output parser had a method called `.parse()`, you could still glue them together — but only with custom code specific to that exact sequence of three types. Add a fourth step, or swap the parser for a retriever, and you'd need to write new glue code again.

Because every one of them instead exposes `.invoke(input) -> output`, `.batch(inputs) -> outputs`, and `.stream(input) -> Iterator[output]`, a generic composition object — `RunnableSequence` — can be written *once*, entirely ignorant of what the individual steps actually do, and it works for any chain of any length made of any mixture of components. This is the classic "program to an interface, not an implementation" principle, applied so consistently that it becomes invisible. You stopped noticing it around Chapter 7, which is exactly the point of good abstraction: it disappears into the background until a chapter like this one asks you to look at it directly.

### 2.3 What kinds of things implement `Runnable`

A non-exhaustive map of the `Runnable` family you've already been using:

| Class | What `.invoke(input)` does |
|---|---|
| `ChatPromptTemplate` | Takes a dict of variables, returns a formatted list of messages |
| `ChatOpenAI` / any chat model | Takes messages, returns an `AIMessage` (making the network call) |
| `StrOutputParser` | Takes an `AIMessage`, returns its `.content` as a plain string |
| `PydanticOutputParser` | Takes text, parses and validates it into a Pydantic model instance |
| `RunnableLambda` | Wraps an arbitrary Python function as a `Runnable` |
| `RunnableParallel` | Takes one input, runs several named branches on it, returns a dict |
| `RunnablePassthrough` | Returns its input unchanged (optionally merging extra keys) |
| A retriever (e.g., a vector store's `.as_retriever()`) | Takes a query string, returns a list of documents |
| `RunnableSequence` | Takes one input, threads it through a list of steps in order |

Every row in that table is a different job. Every row shares the same four verbs: invoke, batch, stream, pipe. That's the architecture in one table.

---

## 3. `__or__` Builds a Data Structure, It Does Not Execute Anything

### 3.1 The line everyone misreads

```python
chain = prompt | model | parser
```

The single most common misconception for engineers new to LangChain Core — understandably, since `|` looks like an operator that "does something" — is imagining this line runs the prompt template, calls the model, and parses a result, all in sequence, right there. **It does none of that.** No network call happens on this line. No template is even filled in yet — you haven't supplied a `topic` or any other input.

What actually happens is pure object construction:

```python
class Runnable(ABC):
    def __or__(self, other):
        return RunnableSequence(first=self, last=other)

    def __ror__(self, other):
        return RunnableSequence(first=other, last=self)
```

`prompt | model` evaluates first (Python's left-to-right operator evaluation), producing `RunnableSequence(steps=[prompt, model])`. Then `(prompt | model) | parser` calls `__or__` again on that sequence, and a well-behaved `RunnableSequence.__or__` implementation *flattens* rather than nests, producing a single sequence of three steps instead of a sequence-of-a-sequence-and-a-step:

```python
class RunnableSequence(Runnable):
    def __init__(self, steps: list[Runnable]):
        self.steps = steps

    def __or__(self, other: Runnable) -> "RunnableSequence":
        if isinstance(other, RunnableSequence):
            return RunnableSequence(steps=self.steps + other.steps)
        return RunnableSequence(steps=self.steps + [other])
```

The end result of `chain = prompt | model | parser` is one object:

```python
chain = RunnableSequence(steps=[prompt, model, parser])
```

Sitting quietly in memory. Nothing has been called. `chain` is inert data — a list of three object references — until something calls `.invoke()`, `.batch()`, or `.stream()` on it.

### 3.2 The object graph, as a diagram

This is worth seeing as a picture, because the mental model "the pipe operator builds a graph, then something later walks the graph" is the single idea this whole chapter hangs on:

```mermaid
flowchart LR
    subgraph BUILD["Construction time: prompt | model | parser"]
        direction LR
        RS["RunnableSequence"]
        RS -->|steps[0]| P["ChatPromptTemplate\n(prompt)"]
        RS -->|steps[1]| M["ChatOpenAI\n(model)"]
        RS -->|steps[2]| PA["StrOutputParser\n(parser)"]
    end
    NOTE["No network calls.\nNo template filled.\nJust three references\nheld in a list."]
    RS -.-> NOTE
```

Nothing in that diagram has an arrow labeled "then" or "returns" between the steps — because at construction time there is no data flowing between them yet. It is purely: "here is a sequence object, and here are its three children." Compare that to the very different diagram in Section 4, which shows the same three objects *during* `.invoke()` — that second diagram has data flowing left to right. The distinction between these two diagrams — a static object graph versus a dynamic execution trace — is exactly the distinction this section exists to teach.

### 3.3 Why this design choice matters practically

Because `chain` is just a plain object holding a list, you can:

- Build it once at module import time (e.g., as a module-level constant) and reuse it for every request in a FastAPI app, since building it does no I/O and has no per-request state.
- Introspect it — `chain.steps` genuinely is a Python list you can iterate, log, or unit-test against, though LangChain also exposes higher-level introspection like `.get_graph()` for visualization.
- Wrap it further (`chain.with_retry()`, `chain | another_step`) without re-executing or duplicating any work, because wrapping, like the original `|`, is just more object construction (Section 7).

This is also why building a chain is cheap and safe to do at request-handling time if you need input-dependent structure (say, choosing which prompt to pipe in based on a request header) — you're just allocating a small object graph, not doing any actual work.

---

## 4. `RunnableSequence.invoke()`: Threading Input Through Steps

### 4.1 The mental model

Once something calls `chain.invoke({"topic": "black holes"})`, the inert object graph from Section 3 springs into action. The logic is almost embarrassingly simple — this is the entire reason LCEL feels predictable once you've seen it once:

```python
class RunnableSequence(Runnable):
    def invoke(self, input, config=None):
        current_input = input
        for step in self.steps:
            current_input = step.invoke(current_input, config)
        return current_input
```

That's it. That's the whole algorithm. Step 1's output becomes step 2's input. Step 2's output becomes step 3's input. The final step's output is returned as the sequence's own output. `RunnableSequence.invoke()` doesn't know or care whether `step` is a prompt template, a network call to an LLM, or a pure Python function — it only knows every `step` has an `.invoke()` method that takes one input and returns one output, which is precisely the contract Section 2 established.

### 4.2 Walking it through by hand

Trace `chain.invoke({"topic": "black holes"})` for `chain = prompt | model | parser`:

```
current_input = {"topic": "black holes"}

# Step 1: prompt.invoke(current_input, config)
#   ChatPromptTemplate fills its template variables.
current_input = ChatPromptValue(messages=[
    SystemMessage("You are a helpful astronomy tutor."),
    HumanMessage("Explain: black holes")
])

# Step 2: model.invoke(current_input, config)
#   ChatOpenAI sends the messages to the API, waits for a response.
current_input = AIMessage(content="A black hole is a region of spacetime...")

# Step 3: parser.invoke(current_input, config)
#   StrOutputParser extracts .content as a plain string.
current_input = "A black hole is a region of spacetime..."

return current_input   # the final string is what chain.invoke() returns to you
```

Every `prompt | model | parser` chain you wrote in Chapters 6 through 14 executed exactly this three-line loop underneath, whether the chain had 3 steps or 12, whether a step was a `RunnableLambda` wrapping five lines of your own Python or a full agent sub-chain. `RunnableSequence` is deliberately the dumbest possible piece of orchestration logic — a for-loop passing output to input — and that dumbness is the feature: it's easy to reason about, easy to debug, and imposes essentially zero framework overhead beyond the loop itself.

### 4.3 Where the real work happens

Note carefully what `RunnableSequence.invoke()` does *not* do: it does not talk to any LLM, format any template, or parse any output itself. All of the actual work lives inside each step's own `.invoke()` implementation — `RunnableSequence` is pure plumbing. This separation is why you can unit-test `prompt.invoke(...)` or `parser.invoke(...)` completely independently of the chain they're used in (a technique worth remembering from Chapter 13's testing material).

---

## 5. `.batch()`: Not a For-Loop of `.invoke()` Calls

### 5.1 The naive (wrong) mental model

It would be reasonable to guess that `chain.batch([input1, input2, input3])` is implemented as:

```python
# NOT how it actually works - shown only to contrast with the real implementation
def batch_naive(self, inputs):
    return [self.invoke(inp) for inp in inputs]
```

This would be *correct* in terms of output, but catastrophic for latency: if each `.invoke()` involves a network round-trip to an LLM API, processing three inputs sequentially takes roughly three times as long as processing one. For a batch of 100 prompts, that's 100x the latency of a single call — completely defeating the purpose of having a `.batch()` method at all.

### 5.2 What actually happens: concurrency by default

The base `Runnable.batch()` implementation instead dispatches invocations across a thread pool, so the *wall-clock* time for a batch is close to the time of the *slowest single call*, not the *sum* of all calls:

```python
from concurrent.futures import ThreadPoolExecutor

class Runnable(ABC):
    def batch(self, inputs, config=None, *, max_concurrency=None):
        with ThreadPoolExecutor(max_workers=max_concurrency) as executor:
            futures = [executor.submit(self.invoke, inp, config) for inp in inputs]
            return [f.result() for f in futures]
```

(The real implementation in LangChain Core adds bookkeeping — preserving input order in the output list, propagating exceptions per-item or in aggregate depending on `return_exceptions`, and honoring `max_concurrency` from `RunnableConfig` — but the concurrency-via-thread-pool idea above is the load-bearing mechanism.) Chat model classes that talk to providers with native batch/bulk endpoints can additionally override `batch()` entirely to issue a single provider-level batched request instead of N thread-pooled ones — the interface guarantees "you get a list of outputs for a list of inputs," not "it must be implemented as N calls," which is exactly the kind of implementation freedom a good abstract base class is supposed to leave open.

### 5.3 How batching composes through a `RunnableSequence`

This is the part that surprises people the first time they think about it carefully. `RunnableSequence` does **not** implement `.batch()` as "call `self.invoke()` N times via a thread pool" (which is what the generic default from Section 5.2 would give you if left unoverridden). Instead, `RunnableSequence.batch()` is overridden to batch **stage by stage**:

```python
class RunnableSequence(Runnable):
    def batch(self, inputs: list, config=None):
        current_inputs = inputs   # a LIST of inputs, not a single input
        for step in self.steps:
            current_inputs = step.batch(current_inputs, config)
        return current_inputs
```

Read that loop carefully against Section 4.1's `invoke()` loop — it's the same shape, one level up. Instead of threading a single value through steps, it threads a *list* of values through steps, and it's each step's own `.batch()` that decides how to parallelize its slice of the work. Concretely, for `chain.batch([input1, input2, input3])`:

1. `prompt.batch([input1, input2, input3])` — formats all three inputs (cheap, CPU-only, effectively instant) and returns a list of three `ChatPromptValue` objects.
2. `model.batch([...three ChatPromptValues...])` — issues **all three LLM calls concurrently** (thread pool, or a provider batch endpoint if the model class overrides `.batch()` for it) and returns a list of three `AIMessage` objects only once all three have completed.
3. `parser.batch([...three AIMessages...])` — parses all three (again cheap, CPU-only) and returns the final list of three strings.

The key consequence: **the expensive step (the model call) is the only one where concurrency actually matters, and it gets to parallelize across the entire batch of 3 in one shot**, rather than the sequence looping "invoke input1 fully through all three steps, then invoke input2 fully through all three steps." Batching flows *horizontally* across the whole input list at each stage, not *vertically* through the whole chain for each input one at a time. This is why `.batch()` on a well-built chain scales far better than looping `.invoke()` yourself in application code — a mistake worth flagging explicitly, since it's an easy one to make coming from a non-LCEL background:

```python
# Common anti-pattern: manually looping invoke() defeats the built-in concurrency
results = [chain.invoke(inp) for inp in inputs]     # sequential, slow

# Correct: let the framework's batch() parallelize internally
results = chain.batch(inputs)                        # concurrent, fast
```

---

## 6. `.stream()`: Chained Generators, Not Chained Buffers

### 6.1 The problem streaming solves

An LLM call is the slowest step in almost every chain, often taking several seconds to fully complete. If `RunnableSequence` streaming worked the same way `.invoke()` does — wait for step 1 to fully finish, then run step 2 to full completion, then run step 3 to full completion — you would see *nothing* on screen until the parser had finished processing the model's *entire* response, at which point you'd get the whole answer at once. That's not streaming at all; it's `.invoke()` with extra steps.

What you actually observed in Chapter 6 onward is genuine token-by-token streaming through a multi-step chain: text starts appearing while the model is still generating it, even though there's a parser step *after* the model in the chain. The mechanism behind that is generators, chained together.

### 6.2 Each step, as a generator

Recall (or imagine, if it's new to you) that a Python generator produces values lazily, one at a time, only as its consumer asks for the next one — it does not compute its entire output up front. `Runnable.stream()` leans on exactly this property:

```python
class Runnable(ABC):
    def stream(self, input, config=None):
        # Default implementation for a Runnable with NO special streaming
        # support: just yield the one invoke() result as a single chunk.
        yield self.invoke(input, config)
```

That default is a graceful fallback — any `Runnable` automatically "supports" `.stream()` even if it can only produce output in one lump. But components built for streaming override this. A chat model, for example, wraps its underlying token-by-token API response as a generator that yields `AIMessageChunk` objects as they arrive from the network, instead of buffering the entire completion before returning anything.

### 6.3 `RunnableSequence.stream()`: piping generators into generators

Here is the piece that makes the whole thing click. `RunnableSequence.stream()` doesn't call each step's `.invoke()` — it calls each step's `.stream()`, and it does so by feeding one step's *output generator* directly in as the next step's *input generator*, rather than fully draining a generator into a list before moving to the next step:

```python
class RunnableSequence(Runnable):
    def stream(self, input, config=None):
        # Conceptually: build a pipeline of generators, one per step,
        # each pulling from the previous one lazily.
        iterator = iter([input])           # step 0's "generator" is just the raw input
        for step in self.steps:
            iterator = step.transform(iterator, config)   # transform() consumes a generator, yields a generator
        yield from iterator
```

`transform()` here is the generalization of `.stream()` to "consume a stream, produce a stream" rather than "consume one value, produce a stream" — it's what lets a middle-of-chain step (like a parser) start emitting partial output as soon as *enough* of the previous step's output has arrived, without waiting for that previous step to fully finish. For a parser like `StrOutputParser`, this typically means: as each `AIMessageChunk` arrives from the model's generator, immediately extract and yield its `.content` piece — no buffering required, because extracting a chunk's text doesn't depend on having seen any of the chunks that come after it.

The net effect: output flows through the whole chain incrementally, chunk by chunk, and no step waits for a previous step to be **completely** done — only for enough of it to be done to produce its own next chunk. Contrast this precisely with `RunnableSequence.invoke()` from Section 4, which *does* wait for each step to fully complete (`step.invoke()` returns one finished value, not a partial one) before starting the next step. Streaming and non-streaming aren't different chains — they're the same chain, executed through two different entry points (`.invoke()` versus `.stream()`) that each step implements differently.

### 6.4 The diagram: streaming as chained generators

```mermaid
sequenceDiagram
    participant M as model.stream()
    participant P as parser.transform()
    participant U as You (consumer)

    Note over M,U: chain = prompt | model | parser ; for chunk in chain.stream(input)

    M->>P: yields AIMessageChunk("A")
    P->>U: yields "A"
    M->>P: yields AIMessageChunk(" black")
    P->>U: yields " black"
    M->>P: yields AIMessageChunk(" hole")
    P->>U: yields " hole"
    Note over M,U: model is STILL generating while<br/>earlier chunks are already on your screen
    M->>P: yields AIMessageChunk(" is...")
    P->>U: yields " is..."
    M-->>P: generation complete, generator exhausted
    P-->>U: generator exhausted
```

Notice there is no step where `P` (the parser) waits for `M` (the model) to finish before starting — every arrow from `M` to `P` is immediately followed by an arrow from `P` to `U`. That's "streaming for free": the parser never had to be taught anything about streaming semantics beyond "process whatever chunk you're handed right now, don't wait for more" — the *chaining* of generators is what turns individually-streamable steps into an end-to-end streamable chain. This is precisely why a chain like `prompt | model | JsonOutputParser()` can stream partially-valid, incrementally-growing JSON to a UI (a pattern you likely used in earlier chapters) — `JsonOutputParser` is written specifically to tolerate and incrementally re-parse a growing, temporarily-incomplete string, chunk by chunk, rather than requiring the full JSON document upfront.

---

## 7. `RunnableConfig`: The Thread That Runs Through Everything

### 7.1 What's actually inside it

Every single method surfaced in Section 2 — `invoke`, `batch`, `stream`, and their async twins — accepts an optional second argument: `config: RunnableConfig`. It's a `TypedDict` (conceptually) carrying cross-cutting concerns that have nothing to do with any individual step's business logic but need to reach *every* step, no matter how deep in a chain:

```python
class RunnableConfig(TypedDict, total=False):
    tags: list[str]                  # labels attached to this run, for filtering in tracing UIs
    metadata: dict[str, Any]         # arbitrary key/value context attached to the run
    callbacks: Callbacks             # handlers that receive lifecycle events (on_llm_start, on_chain_end, ...)
    run_name: str                    # a human-readable name for this specific invocation, shown in traces
    max_concurrency: int             # caps thread-pool/async concurrency for batch() calls
    recursion_limit: int             # guards against runaway recursive chains (e.g., agents calling themselves)
    configurable: dict[str, Any]     # values for fields marked as runtime-configurable (Chapter 9 territory)
    run_id: UUID                     # a unique identifier for this specific run, for correlating trace events
```

If this looks similar to the correlation-ID / distributed-tracing context you'd thread through a microservices call chain in a non-LLM system, that similarity is intentional — it's solving the same problem: how do you propagate cross-cutting observability and control-flow metadata through a deep call stack without every single function in that stack needing custom-written plumbing for it?

### 7.2 Why "threaded through every level" is not optional

Consider what breaks if a step in the middle of a chain receives `config` but doesn't pass it along to whatever it calls internally:

- **Tracing breaks.** If you're using LangSmith or any callback-based tracer, the `callbacks` inside `config` are how a nested LLM call inside step 2 gets associated with the same overall trace as steps 1 and 3. Drop `config`, and that nested call either shows up as an orphaned, disconnected trace or doesn't get traced at all — you lose the ability to see the full waterfall of what happened inside a request.
- **Cancellation and timeouts break.** Async cancellation in Python propagates through the `asyncio` task tree; if a step spins up its own detached async work instead of using the config-aware execution path, a cancellation signal sent to the outer call may never reach that inner work, and it keeps running (and keeps costing you money in API calls) after the caller has given up on it.
- **`max_concurrency` and `recursion_limit` stop being enforced** for anything nested inside the step that dropped the config, because those steps never received the value that was supposed to cap them.
- **Tags and metadata silently vanish** from every downstream event, making a production trace dashboard show a confusing gap exactly where the custom step ran — as if a black box were inserted into an otherwise fully-observable chain.

None of this raises an exception. There is no `MissingConfigError`. The chain still *works*, in the sense that it still produces correct output — which is exactly why this bug class is so easy to ship and so hard to notice until someone goes looking for a trace that isn't there, or a cancellation that didn't take effect. This is what makes config propagation a **hidden contract**: the type signature of `invoke(self, input, config=None)` doesn't force you to use `config` for anything, so nothing in the language stops you from accepting it and ignoring it. Section 9's Real-World Scenario walks through exactly this failure mode end to end.

### 7.3 The correct pattern for a custom `RunnableLambda`

`RunnableLambda` — the wrapper you've used since Chapter 6 to drop a plain Python function into a chain — automatically forwards `config` for you when your function is written to accept it. The rule: if your function needs to call another `Runnable` internally, or you want it to participate correctly in tracing, declare and forward `config` explicitly:

```python
from langchain_core.runnables import RunnableLambda, RunnableConfig

def enrich_with_metadata(input: dict, config: RunnableConfig) -> dict:
    # This inner call receives the SAME config the outer call was given -
    # same callbacks, same tags, same run correlation.
    summary = summarizer_chain.invoke(input["document"], config=config)
    return {**input, "summary": summary}

step = RunnableLambda(enrich_with_metadata)
```

If `enrich_with_metadata` had instead been written as `def enrich_with_metadata(input): ...` (no `config` parameter) and had called `summarizer_chain.invoke(input["document"])` with no config argument at all, the inner `summarizer_chain` call would run as an entirely disconnected, untraceable, uncancellable operation relative to the outer chain — functionally correct, silently broken for everything except "does it return the right value."

---

## 8. `RunnableBinding`: How `.bind()`, `.with_retry()`, `.with_fallbacks()`, and `.with_config()` Actually Work

### 8.1 The pattern: wrap, don't mutate

You've called all four of these methods since around Chapter 8:

```python
model_with_stop = model.bind(stop=["\n\n"])
resilient_model = model.with_retry(stop_after_attempt=3)
safe_chain = primary_chain.with_fallbacks([backup_chain])
configured = chain.with_config({"run_name": "summarize-v2", "tags": ["prod"]})
```

Every one of these returns a **new** `Runnable` object; none of them mutate `model`, `primary_chain`, or `chain` in place. The original object is left completely untouched and can still be used independently elsewhere. This is the same discipline as immutable value objects in other domains — and it exists for the same reason: two different parts of a codebase might hold a reference to the same base chain and want to configure it differently without stepping on each other.

The mechanism behind all four calls is the same class: `RunnableBinding`. Simplified:

```python
class RunnableBinding(Runnable):
    def __init__(self, bound: Runnable, kwargs: dict = None, config: RunnableConfig = None):
        self.bound = bound        # the ORIGINAL runnable, held by reference, never mutated
        self.kwargs = kwargs or {}
        self.config = config or {}

    def invoke(self, input, config=None):
        merged_config = {**self.config, **(config or {})}
        return self.bound.invoke(input, merged_config, **self.kwargs)

    def bind(self, **new_kwargs):
        return RunnableBinding(self.bound, {**self.kwargs, **new_kwargs}, self.config)

    def with_config(self, new_config):
        return RunnableBinding(self.bound, self.kwargs, {**self.config, **new_config})
```

`.bind(stop=["\n\n"])` produces a `RunnableBinding` whose `.invoke()` calls the *original* model's `.invoke()`, but always with `stop=["\n\n"]` injected as an extra keyword argument. `.with_config(...)` produces a `RunnableBinding` that merges in extra config on every call. `.with_retry()` and `.with_fallbacks()` follow the identical wrapping idea, just with more elaborate `invoke()` bodies — `.with_retry()`'s wrapper catches exceptions from `self.bound.invoke(...)` and retries according to a backoff policy before giving up; `.with_fallbacks()`'s wrapper catches an exception from the primary bound `Runnable` and, on failure, calls `.invoke()` on the next fallback in its list instead.

### 8.2 Why this is a decorator, in the classic sense

This is the Decorator design pattern, applied consistently: each of `.bind()`, `.with_retry()`, `.with_fallbacks()`, `.with_config()` takes a `Runnable` and returns a *different* `Runnable` that implements the exact same interface (still has `.invoke()`, `.batch()`, `.stream()`, still composable with `|`) but adds one specific piece of behavior around a call to the original. Crucially, because the wrapper is itself a full `Runnable`, wrappers **stack** — you can bind, then retry, then pipe into a sequence, then wrap the whole sequence with a fallback, and every layer still exposes the same four verbs to whatever wraps it next or calls it directly:

```python
model_final = (
    model
    .bind(stop=["\n\n"])                 # RunnableBinding wrapping model
    .with_retry(stop_after_attempt=3)    # RunnableBinding (or similar) wrapping the previous binding
)
chain = prompt | model_final | parser    # still a completely ordinary RunnableSequence
resilient_chain = chain.with_fallbacks([backup_chain])  # one more wrapping layer on top
```

At no point does any layer need special-case code for "what if the thing I'm wrapping is itself a wrapper" — it just calls `.invoke()` on whatever `Runnable` it's holding, exactly like `RunnableSequence` calling `step.invoke()` without caring what `step` actually is. It's the same uniform-interface principle from Section 2, applied recursively.

---

## 9. `RunnableParallel`: Fan-Out, Then Fan-In

### 9.1 What it does

`RunnableParallel` (which you've also written implicitly any time you passed a plain `dict` of runnables inside a chain, since LangChain auto-converts a dict literal into a `RunnableParallel`) takes **one input** and sends **the same input** to every named branch, then assembles a dict keyed by branch name once every branch has produced its output:

```python
from langchain_core.runnables import RunnableParallel

analysis = RunnableParallel(
    summary=summarize_chain,
    sentiment=sentiment_chain,
    entities=extract_entities_chain,
)

result = analysis.invoke(document_text)
# result == {
#     "summary": "...",
#     "sentiment": "positive",
#     "entities": [...],
# }
```

### 9.2 How branches actually run

The synchronous `.invoke()` path dispatches the branches across a thread pool — the same tool used by `.batch()` in Section 5 — since each branch is typically I/O-bound (an LLM call), not CPU-bound, so Python's Global Interpreter Lock doesn't prevent real concurrency here:

```python
class RunnableParallel(Runnable):
    def invoke(self, input, config=None):
        with ThreadPoolExecutor() as executor:
            futures = {
                name: executor.submit(branch.invoke, input, config)
                for name, branch in self.branches.items()
            }
            return {name: future.result() for name, future in futures.items()}
```

The async path (`.ainvoke()`) is arguably the more natural fit and dispatches branches via `asyncio.gather()` instead of a thread pool — genuinely concurrent I/O-bound waiting, without needing OS threads at all. Either way, the dict is only returned once **every** branch has finished — `RunnableParallel` fans the single input out, then blocks until the slowest branch completes, then fans the results back in as one dict. If `entities=extract_entities_chain` happens to be the slowest of the three branches, the overall `.invoke()` call takes as long as that slowest branch, not the sum of all three — the same "wall-clock ≈ slowest branch" principle you saw for `.batch()` in Section 5, just fanning out across *branches* for one input instead of across *inputs* for one chain.

### 9.3 Where you've already used this without naming it

Every time you wrote a dict literal directly inside a chain — a pattern common for combining a retriever's output with the original question before a prompt template — you were building a `RunnableParallel` without calling the class by name:

```python
rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | model
    | parser
)
```

That `{"context": retriever, "question": RunnablePassthrough()}` dict is auto-wrapped into a `RunnableParallel` with two branches, both receiving the same incoming question string: the `retriever` branch turns it into retrieved documents, and the `RunnablePassthrough()` branch returns the question unchanged. Both run (potentially concurrently), and the resulting `{"context": [...], "question": "..."}` dict becomes the input to `prompt`, exactly as described in Section 9.1 — this is the mechanism, named explicitly, behind a pattern you likely typed from muscle memory since your first RAG-style chain.

---

## 10. Real-World Scenario

**Setup.** A team building an internal document-Q&A assistant has a chain that looks roughly like this:

```python
def add_request_context(input: dict) -> dict:
    """Attaches a user-facing request ID and strips internal-only fields
    before the document goes to the LLM."""
    cleaned = {k: v for k, v in input.items() if not k.startswith("_")}
    enriched_doc = context_enricher_chain.invoke(cleaned["document"])
    return {**cleaned, "document": enriched_doc}

context_step = RunnableLambda(add_request_context)

qa_chain = context_step | prompt | model | parser
```

`add_request_context` is a small helper written early in the project, before anyone on the team had thought carefully about `RunnableConfig`. It takes `input`, does some cleanup, calls `context_enricher_chain.invoke(cleaned["document"])` — another small internal chain that runs a cheap classification LLM call to tag the document with a topic label — and returns the result. It works. Tests pass. It ships.

**The symptom.** Weeks later, during a production incident, the on-call engineer opens the LangSmith trace for a slow request to figure out where time is being spent. The trace shows `qa_chain` starting, then `prompt` and `model` and `parser` each showing up as expected nested spans — but there's a mysterious multi-second gap between "`qa_chain` started" and "`prompt` started," with nothing in the trace to explain it. Separately, a different engineer is trying to add a hard request-level timeout (canceling in-flight chain calls if a user disconnects) and discovers that requests hitting this particular chain keep running in the background even after the client cancels — burning API quota on abandoned requests.

**Root cause.** `add_request_context(input: dict) -> dict` never declared a `config` parameter, and its internal call `context_enricher_chain.invoke(cleaned["document"])` passes **no config at all** — not `None` as a placeholder, just nothing, which defaults to a brand-new, disconnected config inside `context_enricher_chain`. Two separate consequences follow directly from that one omission:

1. **Broken tracing:** `context_enricher_chain`'s LLM call runs with its own fresh callback set instead of inheriting the parent request's callbacks, so it never gets attached to the parent trace as a child span. It still runs — its latency is real and part of the gap the on-call engineer saw — but it's invisible in the trace, making the gap look like unexplained overhead inside `RunnableLambda` itself rather than what it actually is: an untraced nested chain call.
2. **Broken cancellation:** because `context_enricher_chain.invoke()` was called with no config, it has no way to observe a cancellation signal that was meant to propagate from the outer request. Even after the outer request is canceled, this inner call runs to completion on its own.

**The fix.** Declare `config` in the function signature and forward it explicitly to every nested `Runnable` call inside the function body:

```python
from langchain_core.runnables import RunnableConfig

def add_request_context(input: dict, config: RunnableConfig) -> dict:
    cleaned = {k: v for k, v in input.items() if not k.startswith("_")}
    enriched_doc = context_enricher_chain.invoke(cleaned["document"], config=config)
    return {**cleaned, "document": enriched_doc}

context_step = RunnableLambda(add_request_context)
```

`RunnableLambda` inspects the function signature and, seeing a `config` parameter, passes the actual config it received through to `add_request_context` automatically — no extra wiring needed on the caller's side. With `config=config` now threaded into the inner `.invoke()` call, the nested LLM call inherits the same callbacks (fixing the trace gap) and becomes properly cancellable through the same mechanism as everything else in the chain.

**Lesson.** A `Runnable` that accepts `config` but never forwards it downstream is not a style issue — it's a correctness bug that happens to produce the right *return value* while silently breaking every cross-cutting concern (tracing, cancellation, concurrency limits, tags) for everything nested inside it. Because nothing enforces this at the type level, it has to be enforced by habit: **any custom `RunnableLambda` (or hand-written `Runnable` subclass) that calls another `Runnable` internally must accept `config` and pass it along.**

---

## 11. Best Practices

- **Always accept and forward `config` in custom Runnables.** Any `RunnableLambda` or hand-written `Runnable` subclass that calls another `Runnable` internally should declare `config: RunnableConfig` and pass it through explicitly, per Section 10.
- **Build chains once, reuse them across requests.** Since `|` only constructs an object graph and does no I/O, define module-level chains in a FastAPI app rather than rebuilding them per request — it's both cheaper and clearer.
- **Prefer `.batch()` over a manual `for` loop calling `.invoke()`** whenever you have a known list of independent inputs — you get thread-pool (or provider-level) concurrency for free, per Section 5.
- **Use `.with_config()` to tag and name runs**, not ad-hoc logging, so that tracing tools can group and filter runs meaningfully in production.
- **Treat `.bind()`, `.with_retry()`, `.with_fallbacks()`, and `.with_config()` as non-mutating** — always capture their return value in a new variable; never assume they change the original object, and never rely on the original still behaving unwrapped after you've "wrapped" it under the same name.
- **When writing a custom streaming-aware step, only claim streaming support if you can genuinely produce partial output incrementally** — otherwise, rely on the default `.stream()` fallback (Section 6.2), which correctly degrades to "yield the one complete result," rather than faking partial output that isn't really partial.
- **Reach for `RunnableParallel` (or the dict-literal shorthand) whenever a step needs to fan the same input out to multiple independent branches** — it's both clearer and measurably faster than sequential branch calls.

---

## 12. Common Mistakes

- **Assuming `|` executes something.** Writing `chain = prompt | model | parser` and expecting a network call or a syntax error if `prompt` is misconfigured — you won't see any failure until `.invoke()`, `.batch()`, or `.stream()` actually runs, since construction is pure object-building (Section 3).
- **Dropping `RunnableConfig` in a custom `RunnableLambda`.** The single most common and hardest-to-detect internals bug in production LangChain Core code — it doesn't raise, it just silently disables tracing and cancellation downstream (Section 7, Section 10).
- **Manually looping `.invoke()` instead of using `.batch()`** for a list of independent inputs, leaving significant, free concurrency on the table.
- **Assuming `.stream()` guarantees token-by-token output for every chain.** A step that doesn't override the default `Runnable.stream()` (e.g., a plain Python function wrapped in `RunnableLambda` that does a single blocking computation) will emit its output as one chunk, which can make an otherwise-streaming chain feel like it "pauses" at that step before resuming — the fallback isn't broken, it's just not incremental.
- **Believing `.bind()` / `.with_config()` mutate the original object.** Reassigning `model = model.with_retry(...)` works fine, but code elsewhere holding an earlier reference to the pre-wrapped `model` will not see the retry behavior — a subtle source of "why isn't my retry working" bugs when a chain is built up across multiple functions or modules.
- **Confusing the object-graph diagram with an execution-trace diagram.** They look similar (boxes and arrows) but describe entirely different moments in time — Section 3.2's diagram exists purely to keep these visually distinct in your head.
- **Writing a custom `Runnable` subclass that only implements `invoke()`** and assumes `batch()`/`stream()` will magically be efficient — the *default* implementations (Section 2.1) will still work correctly, but only override `batch()`/`stream()` yourself if you have a genuine efficiency gain to offer (e.g., a provider-native batch endpoint); otherwise the defaults are perfectly fine and you should not reinvent them.

---

## Summary

- Every LCEL-composable object — prompts, models, parsers, retrievers, `RunnableLambda`, `RunnableParallel` — implements the same `Runnable` interface: `invoke`, `ainvoke`, `batch`, `abatch`, `stream`, `astream`, `astream_events`, and `__or__`. This uniformity is what makes generic composition possible.
- `prompt | model | parser` calls `__or__` at **construction time** and produces a `RunnableSequence` object holding a plain list of steps — no execution happens on that line.
- `RunnableSequence.invoke(input)` is a simple for-loop: each step's output becomes the next step's input, ending with the final step's output.
- `.batch()` is not a for-loop of `.invoke()` calls — the base implementation parallelizes via a thread pool (or provider-level batch endpoints where available), and `RunnableSequence.batch()` batches **stage by stage**, so the whole list of inputs flows through each step together.
- `.stream()` propagates through a chain as **chained generators**: each step yields chunks as they become available, and the next step consumes those chunks incrementally rather than waiting for the previous step to fully finish — this is the real mechanism behind "streaming for free."
- `RunnableConfig` carries callbacks, tags, metadata, run naming, concurrency limits, and recursion limits through every level of execution. A custom `Runnable` that accepts but doesn't forward `config` silently breaks tracing and cancellation for everything nested inside it — a hidden contract violation that produces correct output but broken observability.
- `.bind()`, `.with_retry()`, `.with_fallbacks()`, and `.with_config()` all follow the `RunnableBinding` decorator pattern: they wrap a `Runnable` in a new `Runnable` that adds behavior, leaving the original untouched, and these wrappers stack cleanly because a wrapper is itself a full `Runnable`.
- `RunnableParallel` dispatches one input to several named branches concurrently (threads for sync, `asyncio.gather` for async) and assembles a dict of results only once every branch completes — the mechanism behind the dict-literal shorthand you've used in retrieval chains since earlier chapters.
- Everything you did in Chapters 6 through 14 — piping, batching, streaming, retries, fallbacks, parallel branches — was this machinery running underneath the surface the whole time.

---

## Knowledge Check

1. Explain in your own words why `chain = prompt | model | parser` does not make a network call. What Python mechanism (special method) is responsible for what actually happens on that line?
2. Write out, step by step, what `RunnableSequence.invoke({"topic": "black holes"})` does for a three-step chain, naming what each step receives and returns.
3. Why is `chain.batch(inputs)` not implemented as a for-loop calling `chain.invoke()` on each input? Describe what would go wrong (in terms of latency) if it were.
4. For a `RunnableSequence` of `prompt | model | parser`, explain how `.batch()` handles the middle (`model`) step differently from a naive "invoke each input fully through the whole chain, one at a time" approach.
5. In your own words, explain why a `RunnableSequence`'s `.stream()` implementation can start emitting output before the `model` step has finished generating its full response. What Python language feature makes this possible?
6. A colleague writes a `RunnableLambda`-wrapped function that calls another chain internally but never accepts a `config` parameter. What two categories of production behavior does this silently break, and why doesn't the bug ever raise an exception?
7. Explain why `chain.with_retry(...)` returns a new object instead of modifying `chain` in place, and describe one bug this design choice could cause if a developer isn't aware of it.

---

## Hands-On Exercise

Implement a minimal custom `Runnable` subclass **by hand** — not via `RunnableLambda` — that correctly forwards `RunnableConfig`, then compose it into a sequence with a built-in `Runnable`.

**Tasks:**

1. Write a class `UppercaseAndLog(Runnable)` with an `invoke(self, input: str, config: Optional[RunnableConfig] = None) -> str` method that:
   - Reads `config.get("metadata", {})` (guarding for `config` being `None`) and, if a `"request_id"` key is present, includes it in a printed log line (e.g., `print(f"[{request_id}] processing input")`) — this simulates a real cross-cutting concern that depends on config actually being received.
   - Returns `input.upper()`.
   - Reason through, on paper: what would happen to this logging behavior if you wrote `def invoke(self, input):` instead (no `config` parameter at all)? Would the class still satisfy the abstract `Runnable` interface? Would anything downstream notice the difference, and under what circumstances would someone *actually* notice?

2. Compose `UppercaseAndLog()` into a sequence with a built-in `Runnable`, for example:
   ```python
   pipeline = RunnablePassthrough() | UppercaseAndLog()
   ```
   Reason through by hand what `pipeline.invoke("hello", config={"metadata": {"request_id": "req-42"}})` does at each step, and what gets printed.

3. Now imagine `UppercaseAndLog` needs to call *another* `Runnable` internally (e.g., a hypothetical `translation_chain.invoke(input)`). Rewrite the method so that internal call correctly forwards the `config` it received, matching the fix pattern from Section 10.

4. **Bonus:** Sketch (in comments or pseudocode, no need to run anything) what a minimal `batch()` override for `UppercaseAndLog` could look like if you wanted it to log once per item with each item's own metadata, while still processing the list concurrently via a thread pool — reasoning through the shape of the code is the goal, not producing a runnable implementation.

---

## Further Reading

- [LangChain Core — Runnable Interface Reference](https://python.langchain.com/docs/concepts/runnables/) — the official conceptual guide to the `Runnable` protocol and its method surface
- [LangChain Core — LCEL: Streaming](https://python.langchain.com/docs/concepts/streaming/) — how streaming composes across chain steps in production LCEL chains
- [LangChain Core — RunnableConfig Reference](https://python.langchain.com/docs/concepts/runnables/#runnableconfig) — the full field list and propagation rules for `RunnableConfig`
- [LangChain Core — RunnableParallel and RunnablePassthrough](https://python.langchain.com/docs/how_to/parallel/) — fan-out/fan-in composition patterns
- [LangChain Core source: `langchain_core/runnables/base.py`](https://github.com/langchain-ai/langchain) — the actual `Runnable`, `RunnableSequence`, `RunnableParallel`, and `RunnableBinding` implementations this chapter simplified
- Gamma, Helm, Johnson, Vlissides, *Design Patterns: Elements of Reusable Object-Oriented Software* (1994) — the original Decorator pattern reference underlying Section 8's explanation of `RunnableBinding`

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./14-error-handling-and-resilience.md">← Previous: Error Handling & Resilience</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./16-best-practices.md">Next: Best Practices →</a>
</div>
