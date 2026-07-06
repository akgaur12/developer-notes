# 🧠 LLM (Large Language Model) Interview Q&A

## 🔹 Fundamentals

### 1. What is an LLM?
A Large Language Model is a **transformer-based neural network** trained on massive amounts of text to predict the next token in a sequence, enabling it to understand and generate human-like language.

---

### 2. What architecture do most LLMs use?
The **Transformer** architecture (Vaswani et al., 2017), specifically the **decoder-only** variant for most modern LLMs (GPT, LLaMA, Mistral), which uses masked self-attention to predict the next token.

---

### 3. Encoder-only vs Decoder-only vs Encoder-Decoder models?
| Type | Examples | Use Case |
|----|----|----|
| Encoder-only | BERT | Classification, embeddings, understanding |
| Decoder-only | GPT, LLaMA, Mistral | Text generation |
| Encoder-Decoder | T5, BART | Translation, summarization (seq2seq) |

---

### 4. What is Self-Attention?
A mechanism that lets each token in a sequence **weigh the relevance of every other token** when building its representation, using Query (Q), Key (K), and Value (V) projections:
`Attention(Q,K,V) = softmax(QK^T / √d_k) V`

---

### 5. What is Multi-Head Attention?
Running several attention mechanisms ("heads") in parallel, each learning to focus on **different types of relationships** (syntax, coreference, position), then concatenating and projecting their outputs.

---

### 6. Why is attention scaled by √d_k?
To prevent the dot products `QK^T` from growing too large in magnitude as dimensionality increases, which would push the softmax into regions with extremely small gradients.

---

### 7. What is Positional Encoding and why is it needed?
Transformers process tokens in parallel with no inherent sense of order, so positional information (sinusoidal, learned, or rotary — RoPE) is added/injected to give the model a sense of **token sequence/order**.

---

### 8. What is Tokenization?
The process of splitting raw text into smaller units (tokens) — words, subwords, or characters — that the model can map to numeric IDs. Most modern LLMs use **subword tokenization** (BPE, WordPiece, SentencePiece) to balance vocabulary size and coverage of rare words.

---

### 9. What is an Embedding?
A dense vector representation of a token (or piece of text) in a continuous space, where semantically similar tokens end up **closer together**, learned during training.

---

### 10. What is the Context Window?
The maximum number of tokens (input + output combined) an LLM can process/attend to at once. Exceeding it means older tokens get truncated or dropped.

---

## 🔹 Training

### 11. What are the stages of training an LLM?
1. **Pretraining** – self-supervised next-token prediction on massive unlabeled text corpora
2. **Supervised Fine-Tuning (SFT)** – trained on curated instruction/response pairs
3. **RLHF / DPO** – aligned with human preferences using reward models or preference optimization
4. **(Optional) Domain fine-tuning** – adapted to a specific task/domain

---

### 12. What is RLHF (Reinforcement Learning from Human Feedback)?
A technique where a **reward model** is trained on human preference rankings of model outputs, and the LLM is then fine-tuned via reinforcement learning (typically PPO) to maximize that reward — aligning outputs with human preferences for helpfulness, safety, and honesty.

---

### 13. What is DPO (Direct Preference Optimization)?
A simpler alternative to RLHF that directly optimizes the model on preference pairs (chosen vs rejected response) using a closed-form loss, **without needing a separate reward model or RL loop**.

---

### 14. What are Scaling Laws?
Empirical relationships (e.g. Chinchilla, Kaplan et al.) showing that LLM performance improves predictably as a power-law function of **model size, dataset size, and compute**, and that these three should be scaled together for compute-optimal training.

---

### 15. What is Catastrophic Forgetting?
When fine-tuning a model on new data causes it to **lose previously learned capabilities**, because gradient updates overwrite weights important for earlier tasks.

---

## 🔹 Efficient Fine-Tuning & Optimization

### 16. What is PEFT (Parameter-Efficient Fine-Tuning)?
A family of techniques that fine-tune only a **small subset of parameters** (or small added modules) instead of the full model, drastically reducing compute/memory cost while retaining most of full fine-tuning's benefit.

---

### 17. What is LoRA (Low-Rank Adaptation)?
A PEFT method that freezes the original model weights and injects small trainable **low-rank matrices** (A, B) into attention/linear layers. Only these low-rank matrices are trained, reducing trainable parameters by orders of magnitude.

---

### 18. What is QLoRA?
LoRA combined with **4-bit quantization** of the base model, allowing fine-tuning of very large models on a single consumer GPU by drastically reducing memory footprint while training LoRA adapters in higher precision.

---

### 19. What is Quantization?
Reducing the numerical precision of model weights (e.g. FP32 → INT8/INT4) to **shrink model size and speed up inference**, at a small cost to accuracy.

---

### 20. What is Knowledge Distillation?
Training a smaller "student" model to mimic the outputs (logits/behavior) of a larger "teacher" model, producing a **compact model** with comparable performance for cheaper inference.

---

## 🔹 Inference & Generation

### 21. What is Greedy Decoding vs Sampling?
- **Greedy**: always picks the highest-probability next token (deterministic, can be repetitive)
- **Sampling**: draws from the probability distribution (temperature, top-k, top-p control randomness)

*(See [llm_generation_parameters.md](../llm_generation_parameters.md) for details on temperature, top-k, top-p, repeat_penalty, etc.)*

---

### 22. What is Beam Search?
A decoding strategy that keeps track of the **top-k most probable sequences** ("beams") at each step instead of just one, exploring multiple candidate continuations before picking the best overall sequence.

---

### 23. What is KV Caching?
Caching the Key/Value projections of previously generated tokens during autoregressive decoding so they don't need to be recomputed at every step, **significantly speeding up inference**.

---

### 24. What causes LLM inference to be slow, and how is it addressed?
- Sequential token-by-token generation → **speculative decoding** (small draft model proposes tokens, verified by the large model)
- Large KV cache memory → **PagedAttention** (vLLM), **multi-query/grouped-query attention**
- Compute-heavy matmuls → **quantization**, batching, optimized kernels (FlashAttention)

---

### 25. What is FlashAttention?
An IO-aware exact attention algorithm that restructures computation to minimize reads/writes to GPU HBM memory, making attention **faster and more memory-efficient** without approximation.

---

## 🔹 Applications & System Design

### 26. What is RAG (Retrieval-Augmented Generation) and why is it used with LLMs?
Combines an LLM with a retrieval step over an external knowledge base (via vector search) so the model's context includes **relevant, up-to-date, or private information** it wasn't trained on — reducing hallucination and enabling domain-specific answers without retraining.

---

### 27. What is a Vector Database and why does it matter for LLM apps?
A database optimized to store and query high-dimensional embeddings using **approximate nearest neighbor (ANN)** search (e.g. Pinecone, Weaviate, Milvus, FAISS, pgvector). It's the retrieval backbone for RAG pipelines, semantic search, and recommendation systems.

---

### 28. What is Chunking in a RAG pipeline?
Splitting large documents into smaller, semantically coherent pieces before embedding and indexing them, so retrieval returns **focused, relevant context** that fits within the LLM's context window.

---

### 29. What is an AI Agent (in the LLM context)?
A system where an LLM **plans, calls tools/functions, observes results, and iterates** to accomplish a multi-step task autonomously, rather than just producing a single response (e.g. ReAct pattern, function calling, agentic loops).

---

### 30. What is Function/Tool Calling?
A capability where the LLM outputs a structured request (function name + arguments) instead of free text, which the application executes against a real API/tool and feeds the result back into the model's context.

---

### 31. What is Prompt Injection?
A security vulnerability where malicious input (in the prompt or retrieved content) manipulates the LLM into **ignoring its original instructions** or leaking sensitive data/system prompts.

---

### 32. How do you evaluate an LLM's output quality?
- **Perplexity** – model's confidence in predicting held-out text
- **Task-specific metrics** – BLEU/ROUGE (summarization/translation), exact match/F1 (QA)
- **LLM-as-a-judge** – using a strong LLM to score/compare responses
- **Human evaluation** – preference ranking, red-teaming for safety

---

### 33. Why do different LLMs have different context window sizes, and what's the trade-off?
Larger context windows require **quadratically more compute/memory** for standard self-attention (O(n²)), so extending context involves architectural tricks (sparse attention, sliding window, RoPE scaling) and trades off inference cost/latency against how much context the model can use.

---

### 34. What is Mixture of Experts (MoE)?
An architecture where only a subset of specialized sub-networks ("experts") are activated per input token (via a learned router), allowing models to have a **much larger total parameter count** while keeping per-token compute cost low (e.g. Mixtral).

---
