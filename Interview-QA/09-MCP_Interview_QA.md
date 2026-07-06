# 🔌 MCP (Model Context Protocol) Interview Q&A

## 🔹 Fundamentals

### 1. What is MCP (Model Context Protocol)?
MCP is an **open, standardized protocol** (introduced by Anthropic) that defines how LLM applications connect to external data sources, tools, and prompts. It works like a **"USB-C port for AI"** — instead of building a custom integration for every LLM app × every data source/tool combination, both sides implement MCP once and can interoperate.

---

### 2. What problem does MCP solve?
Before MCP, every AI application had to write **bespoke integration code** for each tool/data source (M apps × N tools = M×N integrations). MCP standardizes the interface, so any MCP-compatible client can talk to any MCP-compatible server — turning it into an M+N problem.

---

### 3. What are the three core roles in the MCP architecture?
- **Host** – the AI application the user interacts with (e.g. Claude Desktop, an IDE, a custom agent app)
- **Client** – lives inside the host, maintains a 1:1 stateful connection to a single MCP server
- **Server** – an external program that exposes tools, resources, and/or prompts over MCP

A single host can run multiple clients, each connected to a different server.

---

### 4. What underlying protocol does MCP use for messages?
**JSON-RPC 2.0** — all requests, responses, and notifications between client and server are JSON-RPC messages, giving MCP a well-defined, language-agnostic message format.

---

### 5. What transport mechanisms does MCP support?
- **stdio** – client launches the server as a subprocess and communicates over stdin/stdout; simplest, used for local servers
- **Streamable HTTP** – the server runs as an HTTP endpoint (supporting both request/response and server-sent events for streaming), used for remote/hosted servers
- (Older spec versions also defined a separate **HTTP+SSE** transport, since consolidated into Streamable HTTP)

---

## 🔹 Core Primitives

### 6. What are the three main primitives an MCP server can expose?
- **Tools** – executable functions the LLM can call (model-controlled)
- **Resources** – read-only data/context the client can fetch and attach (application-controlled)
- **Prompts** – reusable, parameterized prompt templates (user-controlled, often surfaced as slash commands)

---

### 7. What is a Tool in MCP, and who decides when to call it?
A named, described function with a JSON Schema input definition that the server exposes (e.g. `search_flights(origin, destination, date)`). The **model** decides when to invoke a tool based on the conversation, similar to function calling — the client sends `tools/call` with the chosen arguments and gets a result back.

---

### 8. What is a Resource in MCP?
A piece of data (a file, a database row, an API response, application state) identified by a URI (e.g. `file:///logs/app.log`, `postgres://db/table/schema`) that the **application/user** decides to attach as context — not something the model autonomously calls like a tool.

---

### 9. What is a Prompt in MCP?
A predefined, parameterized template exposed by the server (e.g. "summarize-pr", "code-review") that a **user** explicitly selects/triggers (often shown as a slash command or menu item in the host UI) to insert a well-crafted prompt into the conversation.

---

### 10. Tools vs Resources vs Prompts — who controls each?
| Primitive | Controlled by | Purpose |
|----|----|----|
| Tools | Model | Take actions / fetch data on demand |
| Resources | Application/User | Attach known context explicitly |
| Prompts | User | Trigger a reusable, templated interaction |

---

### 11. What is Sampling in MCP?
A feature that lets an **MCP server request an LLM completion from the client/host** (i.e. the server can ask the connected AI application to run a piece of text through the model) — inverting the usual direction, useful for agentic servers that need their own LLM calls without embedding their own API key/model access.

---

### 12. What are Roots in MCP?
A way for the client to tell the server which **directories/URIs it's allowed to operate within** (e.g. a specific project folder), scoping a filesystem-style server's access to what's relevant/permitted rather than the whole system.

---

### 13. What is Elicitation in MCP?
A capability allowing a server to **request additional information from the user mid-operation** (e.g. asking for a missing parameter or a confirmation) by sending a structured request back through the client, rather than failing or guessing.

---

## 🔹 Connection Lifecycle

### 14. What happens during MCP connection initialization?
1. Client sends an `initialize` request with its protocol version and supported **capabilities**
2. Server responds with its own protocol version and capabilities (which primitives it supports: tools/resources/prompts/etc.)
3. Client sends an `initialized` notification to confirm
4. Both sides now know what features are mutually supported before any real interaction happens

---

### 15. What is capability negotiation, and why does MCP need it?
Since servers/clients may implement different subsets of the spec (e.g. a server might support tools but not sampling), capability negotiation during initialization lets both sides **discover what's actually supported** before attempting to use a feature, avoiding runtime errors from calling unsupported functionality.

---

### 16. How does a client discover what tools/resources/prompts a server offers?
Via list methods — `tools/list`, `resources/list`, `prompts/list` — which the server implements and returns metadata (name, description, schema) for each item. The client also can subscribe to **change notifications** (e.g. `notifications/tools/list_changed`) if the server's available tools change dynamically.

---

### 17. How does an MCP server actually execute a tool call end-to-end?
1. Model decides to call a tool → host asks the client to send `tools/call` with the tool name and arguments
2. Client sends the JSON-RPC request to the server over the transport
3. Server executes the underlying logic (API call, DB query, file operation, etc.)
4. Server returns a structured result (content blocks: text, image, or embedded resource)
5. Client feeds that result back to the model as a tool result message

---

## 🔹 Security & Auth

### 18. What security concerns are specific to MCP?
- **Tool poisoning / prompt injection** – malicious or compromised servers could return content designed to manipulate the model
- **Overly broad permissions** – a filesystem or shell-executing server given too much access ("confused deputy" risk)
- **Unvetted third-party servers** – installing a random community MCP server runs its code with the permissions you grant it
- **Token/credential exposure** – servers that wrap authenticated APIs need careful credential handling, not model-visible secrets

---

### 19. How does MCP handle authentication/authorization for remote servers?
The spec defines an **OAuth 2.1-based authorization framework** for HTTP-based servers, where the client performs a standard OAuth flow (discovery, authorization code + PKCE) to obtain a token used to authenticate requests to the server — local stdio servers typically don't need this since they run under the user's own OS-level permissions.

---

### 20. What best practices reduce risk when connecting to MCP servers?
- Only install/connect to servers from trusted sources
- Review what tools a server exposes and what permissions it requests (e.g. filesystem roots) before granting access
- Prefer human-in-the-loop confirmation for high-risk tool calls (file writes, payments, deletions)
- Treat resource/tool-result content as **untrusted input** that could contain injected instructions, same as any external data fed to an LLM

---

## 🔹 Ecosystem & Comparisons

### 21. How is MCP different from a plain OpenAI/Anthropic-style function/tool call?
Function calling is a **model-level capability** (the model outputs a structured call), while MCP is an **integration-level protocol** standardizing how those callable tools (plus resources and prompts) are *discovered, described, and invoked* across different applications and servers — MCP servers ultimately still get exposed to the model as tool-calling-compatible tools, but the wiring/discovery is standardized rather than app-specific.

---

### 22. How does MCP relate to LangChain tools/agents?
They operate at different layers: LangChain provides the **application-level abstractions** (chains, agents, retrievers) inside your app, while MCP defines a **standard wire protocol** for connecting to external tool/data servers. In practice, LangChain (and other frameworks) ship MCP client adapters that turn any MCP server's tools into native LangChain `Tool` objects usable inside an agent.

---

### 23. Can one MCP server be used by multiple different AI applications?
Yes — that's the core value proposition. A single MCP server (e.g. a GitHub or Slack integration) written once can be connected to by any MCP-compliant host (Claude Desktop, an IDE, a custom agent), instead of each application needing its own bespoke GitHub/Slack integration.

---

### 24. What are some example use cases for MCP servers?
- Filesystem access (read/write project files)
- Database querying (Postgres, SQLite)
- Version control / issue tracker integration (GitHub, Jira, Linear)
- Web search / browsing
- Internal company APIs and knowledge bases
- Developer tools (running tests, linters, build systems)

---

### 25. What is an MCP client SDK, and what languages are supported?
Official SDKs (Python, TypeScript, Java, Kotlin, C#, and others) that implement the client/server sides of the protocol — handling JSON-RPC message framing, capability negotiation, and transport details — so developers write server/client logic without hand-rolling the protocol layer.

---

## 🔹 Design Rationale & Practical Considerations

### 26. Why JSON-RPC instead of a plain REST API for MCP?
JSON-RPC naturally supports **bidirectional communication** (both client→server requests and server→client requests, needed for features like sampling and elicitation) and notifications (fire-and-forget messages like list-changed events) over a single persistent connection — which a traditional stateless REST API doesn't model as cleanly.

---

### 27. Why is the client-server relationship 1:1 per connection in MCP?
Each client instance maintains a dedicated, stateful session with exactly one server, keeping capability negotiation, subscriptions, and context scoped and simple. A host application achieves multi-server connectivity by running **multiple client instances**, one per server, rather than one client multiplexing many servers.

---

### 28. How should a developer decide whether something should be a Tool or a Resource in their MCP server?
If the LLM should be able to **autonomously decide** to fetch/act on it based on conversation context → Tool. If it's context that the **user/application already knows it wants** to provide (a specific file, a known dataset) → Resource. Getting this right affects whether the model has to "guess" to call something vs. it being handed to it directly.

---

### 29. What are common pitfalls when building an MCP server?
- Poorly written tool descriptions/schemas → the model calls tools incorrectly or not at all
- Exposing overly broad or dangerous capabilities (e.g. unrestricted shell execution) without safeguards
- Not handling errors gracefully — returning raw stack traces instead of structured, actionable error content
- Ignoring capability negotiation and assuming every client supports every feature (e.g. sampling)
- Not scoping filesystem/resource access via Roots, leading to unintended access outside the intended project

---

### 30. What are common pitfalls when building an MCP client/host?
- Blindly trusting tool/resource results as safe, non-adversarial input (opening the door to prompt injection from malicious servers)
- Not surfacing tool calls / permissions clearly to the user before execution
- Failing to handle server disconnects/errors gracefully mid-session
- Hardcoding assumptions about a specific server instead of relying on discovered capabilities/schemas

---
