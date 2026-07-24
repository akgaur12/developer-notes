# LangGraph Mastery — From Orchestration Novice to Production Expert

## Course Overview

You've already mastered the **building blocks** — LangChain Core's Runnables, messages, chat models, tools, retrievers, and streaming. What's missing is the **framework for assembling them reliably at scale**: **LangGraph** — the orchestration engine for building stateful, long-running AI workflows that survive crashes, support human review, and coordinate multiple agents.

This course teaches LangGraph not as "just another framework" but as a **state machine with superpowers** — graphs, immutable state management, durable checkpointing, human-in-the-loop interrupts, and production-grade persistence that let you build systems simple chains can't match.

By the end, you'll be able to:
- Design and implement complex, multi-agent workflows with state you can reason about
- Build systems that survive outages and resume gracefully (durable execution)
- Integrate human review and approval into autonomous AI systems
- Stream live updates to UIs while the graph is still executing
- Deploy production-grade AI platforms with LangSmith observability and monitoring
- Refactor real systems (like a MongoDB/FastAPI/Bedrock AI assistant) to use LangGraph's composable patterns

## Learning Roadmap

| Phase | Focus | Chapters |
| ----- | ------ | --------- |
| 1 | Fundamentals: graphs as state machines | 01–07 |
| 2 | Execution engine: durability, streaming, human review | 08–12 |
| 3 | Advanced patterns: parallelism, multi-agent systems | 13–16 |
| 4 | Production: testing, error handling, deployment, capstones, interviews | 17–22 |

## Prerequisites

- **Python**: comfortable with async/await, type hints, dataclasses, TypedDict, Pydantic
- **LangChain Core**: you understand Runnables, messages, chat models, tools, and streaming (see the companion [LangChain Core course](../langchain-core-course/00-index.md) if not)
- **FastAPI**: basic familiarity with routes, dependency injection, and async request handling
- **Databases**: exposure to at least one of MongoDB, PostgreSQL, or SQLite (helpful for state-persistence examples)
- **LLM APIs**: comfortable calling OpenAI, Anthropic, or similar providers directly
- **MCP**: familiarity with the Model Context Protocol is a plus for the multi-agent chapter, not required

## Estimated Learning Timeline (4–6 Weeks)

| Week | Chapters | Theme |
| ---- | --------- | ------ |
| 1 | 01–07 | Fundamentals: state, nodes, edges, commands, reducers, compilation |
| 2 | 08–10 | Execution foundations: tools, checkpointing, memory |
| 3 | 11–13 | Streaming, human-in-the-loop, parallel execution |
| 4 | 14–16 | Multi-agent systems, subgraphs, advanced routing |
| 5 | 17–20 | Testing, error handling, deployment, observability |
| 6 | 21–22 | Capstone projects and interview preparation |

## Complete Chapter Index

| # | Chapter | Description |
| --- | ------ | ----------- |
| 00 | [Index](./00-index.md) | This page |
| 01 | [Introduction & Prerequisites](./01-introduction-and-prerequisites.md) | Why LangGraph exists, the state-machine mental model, when (not) to use it |
| 02 | [StateGraph & State Management](./02-stategraph-and-state-management.md) | TypedDict, dataclass, Pydantic state; reducers; immutable thinking |
| 03 | [Nodes](./03-nodes.md) | Node types, execution contract, state updates, LLM/tool/DB/API nodes |
| 04 | [Edges & Routing](./04-edges-and-routing.md) | Normal edges, conditional edges, dynamic routing, branching |
| 05 | [Commands & Dynamic Control](./05-commands-and-dynamic-control.md) | `Command`, `goto`, `update`, dynamic transitions within a node |
| 06 | [Reducers](./06-reducers.md) | Merge strategies for concurrent/repeated state updates |
| 07 | [Compilation & Execution](./07-compilation-and-execution.md) | `.compile()`, `.invoke()`, the super-step execution loop, recursion limits |
| 08 | [Tool Calling Patterns](./08-tool-calling-patterns.md) | `ToolNode`, `bind_tools`, the ReAct loop, structured tool execution |
| 09 | [Checkpointing & Durable Execution](./09-checkpointing-and-durable-execution.md) | `MemorySaver`, SQLite, PostgreSQL checkpointers, resume after crashes |
| 10 | [Memory Management](./10-memory-management.md) | Short-term (thread-scoped) state vs. long-term (cross-session) memory stores |
| 11 | [Streaming](./11-streaming.md) | `stream_mode` values, token streaming, state streaming, event streaming |
| 12 | [Human-in-the-Loop](./12-human-in-the-loop.md) | `interrupt()`, review/approval workflows, resuming with `Command(resume=...)` |
| 13 | [Parallel Execution](./13-parallel-execution.md) | Fan-out/fan-in, concurrent nodes, reducer-based merging |
| 14 | [Multi-Agent Systems](./14-multi-agent-systems.md) | Coordinator/supervisor pattern, specialized agents, handoffs |
| 15 | [Subgraphs & Composition](./15-subgraphs-and-composition.md) | Composing graphs, invoking subgraphs, shared vs. private state |
| 16 | [Advanced Routing Patterns](./16-advanced-routing-patterns.md) | Complex/nested conditional dispatch, map-reduce style graphs |
| 17 | [Testing LangGraph Applications](./17-testing-langgraph-applications.md) | Unit-testing nodes, integration-testing graphs, mocking LLMs |
| 18 | [Error Handling & Resilience](./18-error-handling-and-resilience.md) | Retries, fallbacks, timeouts, exception handling, recovery |
| 19 | [Production Deployment](./19-production-deployment.md) | `langgraph.json`, FastAPI integration, Docker, environment config |
| 20 | [Observability & Monitoring](./20-observability-and-monitoring.md) | LangSmith tracing, metrics, debugging production graphs |
| 21 | [Capstone Projects](./21-capstone-projects.md) | Four tailored projects: analytics agent → multi-agent → code-review → enterprise platform |
| 22 | [Interview Preparation](./22-interview-preparation.md) | FAQ, scenario questions, system design, troubleshooting, production case studies |

## Milestones

- [ ] **Milestone 1 (Week 1)**: Build a chat workflow with state management and conditional routing
- [ ] **Milestone 2 (Week 2)**: Add checkpointing so your graph survives a simulated crash and resumes from the last checkpoint
- [ ] **Milestone 3 (Week 3)**: Add streaming and a human-approval interrupt to that graph
- [ ] **Milestone 4 (Week 4)**: Implement a multi-agent system with a coordinator dispatching to specialized sub-agents
- [ ] **Milestone 5 (Week 5)**: Ship a tested, error-resilient LangGraph service behind FastAPI
- [ ] **Milestone 6 (Week 6)**: Deploy with LangSmith observability and complete all four capstone tiers

## Recommended Resources

- [LangGraph Documentation](https://docs.langchain.com/oss/python/langgraph/overview)
- [LangGraph Application Structure Guide](https://docs.langchain.com/oss/python/langgraph/application-structure)
- [LangGraph GitHub Repository](https://github.com/langchain-ai/langgraph)
- [LangSmith Documentation](https://docs.smith.langchain.com/)
- Related course in this repo: [LangChain Core — From LLM/FastAPI Engineer to Production LCEL Practitioner](../langchain-core-course/00-index.md)
- Related course in this repo: [LLM Fundamentals](../llm-fundamentals-course/00-index.md)

## Learning Priority (80/20)

If time is short, focus on these in order (then circle back for the rest):

1. **StateGraph & State Management** (Ch. 2) ⭐⭐⭐⭐⭐ — the mental model that unlocks everything else
2. **Nodes & Conditional Edges** (Ch. 3–4) ⭐⭐⭐⭐ — how computation actually flows
3. **Commands & Dynamic Routing** (Ch. 5) ⭐⭐⭐ — collapse routing logic into the node itself
4. **Checkpointing & Persistence** (Ch. 9) ⭐⭐⭐⭐⭐ — what makes LangGraph production-ready
5. **Human-in-the-Loop** (Ch. 12) ⭐⭐⭐⭐ — a genuine differentiator vs. other frameworks
6. **Multi-Agent Systems** (Ch. 14) ⭐⭐⭐ — apply everything above to coordinate specialized agents
7. **Production Deployment** (Ch. 19) ⭐⭐⭐ — ship it to real users
8. Streaming, Parallel Execution, Subgraphs, Testing (Ch. 11, 13, 15, 17) — round out production fluency
9. Capstones + Interview Prep (Ch. 21–22) — solidify and interview confidently

Once you've mastered State, Nodes, Edges, and Checkpointing, you can build real production systems — everything else is refinement, patterns, and scale.

<div style="display:flex;justify-content:space-between;align-items:center;">
  <span></span>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./01-introduction-and-prerequisites.md">Next: Introduction & Prerequisites →</a>
</div>
