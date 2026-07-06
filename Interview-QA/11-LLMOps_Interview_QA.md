# 🛠️ LLMOps Interview Q&A

## 🔹 Fundamentals

### 1. What is LLMOps?
LLMOps (LLM Operations) is the set of practices and tooling for **developing, deploying, monitoring, and maintaining** LLM-powered applications in production — covering prompt lifecycle management, evaluation, observability, cost/latency optimization, and safety guardrails.

---

### 2. How does LLMOps differ from traditional MLOps?
| MLOps | LLMOps |
|----|----|
| Manages model **training** pipelines, feature stores, retraining | Often uses **pretrained/third-party models** via API — no training pipeline needed |
| Versions models and datasets | Versions **prompts**, chains/agents, and retrieved-context sources |
| Evaluation via fixed metrics (accuracy, F1) | Evaluation is harder — open-ended output needs LLM-as-judge/human eval |
| Deployment = serving a model artifact | Deployment = shipping a prompt/chain/agent configuration, often calling an external API |
| Cost driven by compute/infra | Cost driven largely by **token usage** per API call |

---

### 3. What are the main stages of the LLMOps lifecycle?
1. **Prompt/pipeline development** – iterating on prompts, RAG pipelines, or agent logic
2. **Evaluation** – testing against golden datasets, LLM-as-judge, human review
3. **Deployment** – rolling out a prompt/model/chain version safely
4. **Monitoring & observability** – tracing requests, tracking quality/cost/latency in production
5. **Feedback loop** – capturing user feedback and production failures to drive the next iteration

---

## 🔹 Observability & Tracing

### 4. Why is observability especially important for LLM applications?
LLM pipelines (RAG, agents, multi-step chains) have many **non-deterministic, opaque intermediate steps** — a bad final answer could stem from retrieval, prompt construction, the model call, or a tool call. Without step-by-step tracing, diagnosing *why* an output was wrong is largely guesswork.

---

### 5. What is a "trace" in the context of LLM observability tools (e.g. LangSmith, Langfuse)?
A recorded, hierarchical record of an entire request's execution — every LLM call, tool call, retrieval step, and intermediate output, along with latency, token usage, and cost for each step — letting you inspect exactly what happened for any given request.

---

### 6. What metrics should be tracked for an LLM application in production?
- **Latency** (p50/p95/p99, time-to-first-token for streaming)
- **Token usage and cost** per request/feature
- **Error rates** (API failures, timeouts, parsing failures)
- **Quality signals** (user feedback/thumbs up-down, automated eval scores, hallucination flags)
- **Throughput** (requests per second, rate-limit headroom)

---

### 7. What is a "span" vs a "trace"?
A **trace** represents one complete end-to-end request; a **span** is a single step within that trace (e.g. one LLM call, one retrieval call, one tool invocation) — traces are composed of nested spans, mirroring how distributed tracing works in traditional software observability (OpenTelemetry-style).

---

## 🔹 Evaluation

### 8. What is Offline Evaluation, and how is it done for LLM apps?
Running a fixed, curated set of inputs (a **golden dataset**) through the pipeline before deployment and scoring outputs against expected answers or grading criteria — done in CI/CD-like fashion to catch regressions before shipping a prompt/pipeline change.

---

### 9. What is Online Evaluation, and how does it complement offline evaluation?
Continuously evaluating **live production traffic** — via implicit signals (user follow-up behavior, thumbs up/down, task completion/abandonment) or automated scoring sampled from real requests — catching issues offline eval's curated dataset might miss (real-world input distribution, edge cases).

---

### 10. What is LLM-as-a-Judge, and what are its limitations?
Using a strong LLM to score/compare outputs against criteria (correctness, faithfulness, relevance) at scale, since human evaluation doesn't scale and simple string-match metrics fail on open-ended text.
**Limitations**: judge models can have their own biases (e.g. favoring longer or more confident-sounding answers), may not catch subtle factual errors, and add their own cost/latency to the eval pipeline — often mitigated by calibrating the judge against a smaller human-labeled sample.

---

### 11. What is Regression Testing for prompts, and why is it necessary?
Re-running the full golden dataset against a **new** prompt/model/pipeline version and comparing scores against the previous version — because prompt engineering is empirical, a change that fixes one case can silently break others; regression testing catches this before it reaches production.

---

### 12. How do you build a good golden/evaluation dataset for an LLM application?
- Include a **representative sample** of real production queries (not just easy cases)
- Cover known **edge cases and failure modes** discovered in production
- Include adversarial/ambiguous inputs, not just "happy path" examples
- Periodically refresh it with newly discovered failure cases (a living dataset, not a one-time artifact)

---

## 🔹 Versioning & Deployment

### 13. What needs to be versioned in an LLM application (beyond code)?
- **Prompts** (system prompts, few-shot examples)
- **Model** (provider + specific model version — e.g. pinned model IDs, since providers can silently update models)
- **Retrieval configuration** (embedding model, chunking strategy, index version)
- **Tool/agent definitions**
- **Evaluation datasets** used to validate each version

---

### 14. Why is pinning a specific model version important in LLMOps?
LLM providers periodically update "aliased" model endpoints (e.g. a `-latest` tag) with new underlying weights, which can **silently shift behavior** on existing prompts without any code change on your side — pinning to an explicit dated/versioned model ID avoids unannounced regressions, at the cost of manually opting into upgrades.

---

### 15. What deployment strategies are used to safely roll out a new prompt or model version?
- **Canary release** – route a small percentage of traffic to the new version, monitor metrics, gradually increase
- **Shadow deployment** – run the new version in parallel on real traffic without serving its output to users, comparing results offline
- **A/B testing** – split traffic between old and new versions and compare quality/business metrics statistically
- **Blue-green** – deploy the new version fully in parallel, switch traffic over, keep the old version ready for instant rollback

---

### 16. How do you roll back a bad prompt/model deployment quickly?
Keep prompt/pipeline versions in a system that supports **instant revert** (version-controlled prompts, feature-flagged model selection) rather than requiring a full code redeploy — combined with monitoring/alerting sensitive enough to catch a quality regression quickly after rollout.

---

## 🔹 Cost & Latency Optimization

### 17. What are the main levers for reducing LLM API cost in production?
- **Prompt/context minimization** – trim unnecessary context, shorter few-shot examples
- **Caching** – exact-match caching for repeated queries, semantic caching for near-duplicate queries
- **Model routing** – use a cheaper/smaller model for simple requests, escalate to a stronger model only when needed
- **Batching** – combine multiple requests where the provider supports batch discounts
- **Prompt caching** (provider-level KV-cache reuse for repeated prefixes, e.g. Anthropic/OpenAI prompt caching) to cut cost on repeated system prompts/context

---

### 18. What is Semantic Caching, and how does it differ from exact-match caching?
Exact-match caching returns a cached response only for an **identical** input. Semantic caching embeds incoming queries and checks for a **sufficiently similar** past query (via vector similarity) to reuse its cached response — catching paraphrased duplicate requests, at the risk of returning a stale/slightly-off answer if the similarity threshold is too loose.

---

### 19. What is Model Routing (or a "model cascade"), and why use it?
Directing requests to different models based on task difficulty/cost sensitivity — e.g. a lightweight/cheap model handles simple classification or short queries, escalating to a larger, more expensive model only for complex requests — balancing quality against cost/latency at scale.

---

### 20. How does streaming responses help with perceived latency, even if total generation time is unchanged?
Streaming sends tokens to the user as they're generated instead of waiting for the full response, dramatically improving **time-to-first-token** and perceived responsiveness — total end-to-end generation time is the same, but the user isn't staring at a blank loading state.

---

### 21. What infrastructure-level techniques reduce LLM serving latency (for self-hosted models)?
- **KV caching** to avoid recomputing attention for prior tokens
- **Continuous batching** (e.g. vLLM) to keep GPU utilization high across concurrent requests
- **Speculative decoding** — a small draft model proposes tokens, verified by the larger model
- **Quantization** to speed up inference at some precision cost

---

## 🔹 Safety, Guardrails & Security

### 22. What are Guardrails in LLMOps, and what do they typically check?
Validation layers wrapped around LLM input/output to enforce safety and correctness constraints — checking for **PII leakage, toxic/unsafe content, jailbreak attempts, off-topic responses, or schema/format violations** — before a request reaches the model or a response reaches the user.

---

### 23. What is PII redaction, and where does it fit in an LLM pipeline?
Detecting and masking/removing personally identifiable information (names, emails, SSNs, etc.) — either **before** sending user input to a third-party LLM API (to avoid leaking sensitive data to an external provider) or **after** generation (to prevent the model from echoing sensitive data back) — often implemented via a dedicated NER/regex-based scanning step.

---

### 24. How do you defend against prompt injection and jailbreak attempts in a production system?
- Clear separation of trusted instructions vs untrusted user/retrieved content in the prompt structure
- Guardrail models/classifiers scanning input and output for injection/jailbreak patterns
- Least-privilege tool access for agents processing untrusted content
- Rate limiting and anomaly detection to catch abusive usage patterns
- Continuous red-teaming to discover new attack patterns before they're exploited in production

---

### 25. What is Hallucination Monitoring, and how is it done at scale?
Automated checks (often via an LLM-as-judge "faithfulness" grader) that sample production responses and verify claims are actually supported by the provided context/sources — flagging responses with unsupported claims for review, since manually checking every production response isn't feasible.

---

## 🔹 Feedback Loops & Continuous Improvement

### 26. How do you collect and use user feedback to improve an LLM application?
- Explicit feedback (thumbs up/down, ratings, corrections) tied back to the specific trace/request
- Implicit feedback (user rephrasing the same question, abandoning a session, escalating to a human)
- Feeding flagged bad responses into the **evaluation dataset** so future prompt/model changes are tested against real failure cases
- Using accumulated feedback data as a candidate dataset for **fine-tuning** if patterns of systematic failure emerge

---

### 27. How does Human-in-the-Loop (HITL) fit into an LLMOps workflow?
HITL can appear at multiple points: reviewing/approving high-risk agent actions before execution, labeling production outputs for quality (feeding evaluation and potential fine-tuning), and escalation paths where a low-confidence or flagged response is routed to a human instead of being shown directly to the end user.

---

### 28. What is Data/Prompt Drift, and how do you detect it in an LLM system?
The phenomenon where the **distribution of real user inputs** shifts over time (new topics, new phrasing patterns, new edge cases) away from what the prompt/pipeline was originally validated against — detected by continuously sampling production traffic for evaluation, monitoring quality metrics over time, and watching for a rising rate of low-confidence/flagged outputs.

---

## 🔹 Practical / Tooling

### 29. What tools are commonly used in an LLMOps stack?
- **Tracing/observability**: LangSmith, Langfuse, Helicone, Arize Phoenix
- **Evaluation**: RAGAS, DeepEval, promptfoo, custom LLM-as-judge pipelines
- **Prompt management**: PromptLayer, LangSmith prompt hub, Git-based prompt version control
- **Guardrails**: Guardrails AI, NeMo Guardrails, Llama Guard
- **Serving/inference**: vLLM, TGI (Text Generation Inference), Ray Serve
- **Experiment/gateway layer**: LiteLLM (multi-provider routing), Portkey

---

### 30. What are common pitfalls in LLMOps that teams run into?
- No regression testing — shipping prompt tweaks based on a handful of manual spot-checks
- No cost visibility per feature/user, discovering a cost blowup only after the bill arrives
- Treating the first working prompt as "done" instead of continuously monitoring/iterating
- Missing tracing, making production incidents nearly impossible to root-cause
- Not pinning model versions, causing silent behavior drift when a provider updates a model
- Under-investing in guardrails until after a public incident (prompt injection, PII leak, toxic output) forces the issue

---
