# 📚 RAG (Retrieval-Augmented Generation) Interview Q&A

## 🔹 Fundamentals

### 1. What is RAG?
Retrieval-Augmented Generation is an architecture that **retrieves relevant external documents/context** (via a search step, usually vector similarity search) and injects them into an LLM's prompt, so the model generates answers **grounded in retrieved information** instead of relying solely on what it memorized during pretraining.

---

### 2. Why is RAG needed if LLMs already know a lot from pretraining?
- LLMs have a **knowledge cutoff** and can't know about recent or private/internal data
- Parametric knowledge can be **imprecise or hallucinated**, especially for long-tail facts
- RAG lets you ground answers in **verifiable, up-to-date, domain-specific** sources without retraining the model
- It's far cheaper and faster to update a document index than to retrain/fine-tune a model

---

### 3. What are the two main stages of a RAG pipeline?
1. **Indexing (offline)** – load documents, chunk them, embed the chunks, store them in a vector store
2. **Retrieval + Generation (online/query time)** – embed the user query, retrieve the most relevant chunks, insert them into the prompt, and generate the final answer conditioned on that context

---

### 4. RAG vs Fine-tuning — when do you use which?
| RAG | Fine-tuning |
|----|----|
| Injects fresh/external **knowledge** at query time | Teaches the model new **behavior/style/format** |
| Cheap and fast to update (just re-index) | Requires retraining, slower/costlier to update |
| Naturally supports citing sources | No inherent traceability of "why" an answer was given |
| Doesn't change model weights | Changes model weights |
| Best for: factual grounding, private data, frequently changing info | Best for: tone, task-specific formatting, domain-specific reasoning patterns |

They are complementary — many production systems fine-tune a model to better use retrieved context *and* use RAG for the knowledge itself.

---

### 5. RAG vs simply using a very long context window — is RAG still needed?
Even with large context windows, RAG remains valuable because:
- Stuffing an entire corpus into context is expensive and slow (cost/latency scale with tokens)
- Models suffer from the **"lost in the middle"** problem — degraded attention to information buried in the middle of very long contexts
- RAG scales to corpora far larger than any context window could hold (millions of documents)
- Retrieval also gives you explicit control over **what** context is included, useful for filtering, permissions, and citations

---

## 🔹 Indexing Pipeline

### 6. What are Document Loaders and why are they the first step?
Components that ingest raw data from various sources (PDFs, HTML, Notion, databases, S3, APIs) into a standard document format (text + metadata) — the starting point of the indexing pipeline before chunking/embedding can happen.

---

### 7. Why is Chunking necessary, and what happens if chunks are too big or too small?
Embedding models and LLM context windows have size limits, and retrieval works best on **focused, semantically coherent** pieces of text.
- **Too large** → chunk contains multiple unrelated topics, diluting the embedding and hurting retrieval precision
- **Too small** → chunk loses surrounding context needed to answer a question, hurting recall/coherence

---

### 8. What are common chunking strategies?
- **Fixed-size chunking** – split every N characters/tokens (simple, can cut sentences awkwardly)
- **Recursive character splitting** – tries paragraph → sentence → word boundaries recursively (most common default)
- **Semantic chunking** – splits based on embedding-similarity shifts between sentences, keeping topically coherent chunks together
- **Document-structure-aware chunking** – splits by markdown headers, HTML tags, or code function/class boundaries
- **Sliding window with overlap** – overlapping chunks so context isn't lost at boundaries

---

### 9. What is chunk overlap, and why use it?
Including a portion of the previous chunk's text at the start of the next chunk, so information/context spanning a chunk boundary isn't lost entirely in either chunk — a trade-off between retrieval quality and index size/redundancy.

---

### 10. What is an Embedding, and why does RAG rely on it?
A dense vector representation of text where semantically similar text maps to nearby points in vector space. RAG uses embeddings to convert both documents and queries into the same space so **similarity search** (rather than exact keyword match) can find conceptually relevant content.

---

### 11. What is a Vector Store/Database, and what does it need to do well?
A database optimized to store embeddings and perform fast **approximate nearest neighbor (ANN)** search over millions/billions of vectors (e.g. FAISS, Pinecone, Weaviate, Milvus, Qdrant, pgvector). Key requirements: low-latency ANN search, metadata filtering, and horizontal scalability.

---

### 12. What are common similarity metrics used in vector search?
- **Cosine similarity** – angle between vectors, most common for text embeddings (normalized magnitude-independent)
- **Dot product** – similar to cosine if vectors are normalized; used when embeddings are trained with dot-product objectives
- **Euclidean (L2) distance** – straight-line distance; less common for high-dimensional text embeddings

---

### 13. What ANN algorithms/index types are commonly used in vector databases?
- **HNSW** (Hierarchical Navigable Small World) – graph-based, very common default, good speed/accuracy trade-off
- **IVF** (Inverted File Index) – clusters vectors, searches only relevant clusters
- **PQ** (Product Quantization) – compresses vectors for memory efficiency, often combined with IVF
- **Flat/brute-force** – exact search, only feasible for small datasets

---

## 🔹 Retrieval Strategies

### 14. What is Dense Retrieval vs Sparse Retrieval?
- **Dense retrieval** – uses learned embeddings + vector similarity (captures semantic meaning, handles synonyms/paraphrasing)
- **Sparse retrieval** – uses term-frequency-based methods like **BM25/TF-IDF** (captures exact keyword/entity matches well, no training needed)

---

### 15. What is Hybrid Search, and why combine dense + sparse retrieval?
Combining dense (semantic) and sparse (keyword, e.g. BM25) retrieval results — often via **Reciprocal Rank Fusion (RRF)** — to get the best of both: semantic understanding for paraphrased queries and precise matching for exact terms, IDs, or rare keywords that embeddings can miss.

---

### 16. What is Re-ranking, and why add it after initial retrieval?
A second-stage step where an initial set of candidate documents (retrieved cheaply/approximately, e.g. top-50) is **re-scored by a more accurate but slower model** (often a cross-encoder that jointly encodes query+document), and only the top re-ranked results (e.g. top-5) are passed to the LLM — improving precision without the cost of running the expensive model over the whole corpus.

---

### 17. Bi-encoder vs Cross-encoder — what's the difference and where is each used?
| Bi-encoder | Cross-encoder |
|----|----|
| Encodes query and document **separately** into vectors, compares via similarity | Encodes query+document **together**, outputs a relevance score directly |
| Fast, scalable to millions of docs (precomputed embeddings) | Slow, must run per query-document pair at query time |
| Used for initial retrieval | Used for re-ranking a small candidate set |

---

### 18. What is Query Transformation, and why is it used?
Techniques that rewrite/expand the user's raw query before retrieval to improve recall/precision:
- **Query expansion/rewriting** – reformulate ambiguous or underspecified queries
- **Multi-query retrieval** – generate several paraphrased versions of the query with an LLM, retrieve for each, and union the results
- **HyDE (Hypothetical Document Embeddings)** – ask an LLM to generate a hypothetical answer to the query, then embed *that* (rather than the raw query) for retrieval, since a hypothetical answer often resembles the actual target document more closely than the question does

---

### 19. What is Metadata Filtering in RAG retrieval?
Restricting vector search to a subset of documents matching structured filters (e.g. `date > 2024`, `department = "finance"`, `access_level <= user_role`) alongside the semantic similarity search — essential for multi-tenant systems, access control, and narrowing results by known constraints.

---

### 20. What is a Self-Query Retriever?
A retriever that uses an LLM to translate a natural-language query into a structured query — separating the **semantic search term** from **metadata filters** — e.g. turning "cheap laptops released after 2023" into a vector search for "laptop" plus a filter `price < X AND year > 2023`.

---

### 21. What is Parent-Document Retrieval?
A strategy that indexes **small chunks** for precise similarity matching but returns the **larger parent chunk/document** they belong to when a match is found, giving the LLM more surrounding context than the tiny matched chunk alone would provide.

---

### 22. What is Multi-hop Retrieval, and when is it needed?
Retrieval that requires **multiple sequential retrieval steps**, where the answer to one sub-question is needed to formulate the next query (e.g. "What company did the founder of X work at before?" requires first finding who founded X, then searching for their employment history) — single-shot retrieval fails on these compositional questions.

---

## 🔹 Advanced RAG Patterns

### 23. What is Agentic RAG?
A RAG pipeline where an **LLM agent decides** whether/when to retrieve, reformulates queries, chooses among multiple retrieval tools/sources, and can iterate (retrieve → evaluate → retrieve again) — as opposed to a fixed single retrieve-then-generate pipeline.

---

### 24. What is Corrective RAG (CRAG)?
A pattern where retrieved documents are first **evaluated for relevance** (often by a lightweight grader model); if retrieved context is judged insufficient/irrelevant, the system falls back to an alternative action (e.g. web search, query rewriting) instead of generating an answer from poor context.

---

### 25. What is Self-RAG?
An approach where the model is trained/prompted to **decide for itself when retrieval is needed**, and to critique its own generated output against the retrieved evidence (reflection tokens indicating relevance, support, and usefulness) — improving factual grounding by making retrieval and self-assessment part of the generation process itself.

---

### 26. What is GraphRAG?
A RAG variant that builds/uses a **knowledge graph** (entities and relationships extracted from documents) instead of (or alongside) flat vector chunks, enabling retrieval that follows relationships between entities — useful for multi-hop reasoning and questions requiring understanding of how concepts connect, not just similarity to isolated text.

---

### 27. What is the "lost in the middle" problem, and how does it affect RAG design?
LLMs tend to attend well to information at the **start and end** of a long context but under-utilize information placed in the **middle**. This affects how many/which retrieved chunks to include and their ordering — e.g. placing the most relevant chunk first or last, and keeping the total retrieved context reasonably small rather than stuffing in everything possible.

---

### 28. What document-combination strategies exist when the retrieved content doesn't fit in one prompt?
- **Stuff** – concatenate everything into one prompt (simplest, limited by context window)
- **Map-Reduce** – process/answer from each document chunk independently, then combine those partial answers
- **Refine** – iteratively update a running answer, one chunk at a time
- **Map-Rerank** – answer independently per chunk, score confidence, return the highest-scoring answer

---

## 🔹 Evaluation

### 29. How do you evaluate a RAG system, and why is it different from evaluating a plain LLM?
RAG has two components to evaluate independently and jointly:
- **Retrieval quality** – are the right documents being found? (precision/recall, hit rate, MRR, NDCG)
- **Generation quality** – given good context, is the answer accurate/well-formed?
- **End-to-end quality** – is the final answer correct and grounded, regardless of where in the pipeline an error occurred

---

### 30. What are the key RAG-specific evaluation metrics (e.g. as used by RAGAS)?
- **Faithfulness** – does the generated answer only contain claims supported by the retrieved context (no hallucination)?
- **Answer relevance** – does the answer actually address the question asked?
- **Context precision** – of the retrieved chunks, how many are actually relevant?
- **Context recall** – did retrieval find all the information needed to fully answer the question?

---

### 31. What is "LLM-as-a-judge" in the context of RAG evaluation?
Using a strong LLM to score/compare RAG outputs against criteria (faithfulness, relevance, correctness vs. a reference answer) at scale, since human evaluation doesn't scale but simple string-overlap metrics (BLEU/ROUGE) poorly capture semantic correctness for open-ended answers.

---

## 🔹 Production & Failure Modes

### 32. What causes hallucination in a RAG system even though context was retrieved?
- Retrieval returned **irrelevant or insufficient** context, and the model fills gaps from its parametric memory
- The model **ignores or contradicts** the provided context (weak grounding/instruction-following)
- The prompt doesn't clearly instruct the model to answer **only** from the given context
- Context was truncated or lost in the middle of a long prompt

---

### 33. How do you reduce hallucination in a RAG pipeline?
- Improve retrieval quality first (better chunking, hybrid search, re-ranking) — most hallucinations stem from bad retrieval
- Explicitly instruct the model to say "I don't know" if the context doesn't contain the answer
- Add a faithfulness-checking step (e.g. a grader LLM call) before returning the answer
- Lower temperature for factual QA tasks
- Show citations/sources so incorrect claims are easier to catch

---

### 34. What security risks exist specifically because RAG injects external content into the prompt?
**Indirect prompt injection** — if retrieved documents come from untrusted or user-editable sources (web pages, uploaded files, emails), an attacker can embed instructions inside that content designed to hijack the model's behavior once it's pulled into context. Mitigations include treating retrieved content as data (not instructions), sanitizing/inspecting sources, and restricting what actions the model can take based on retrieved text alone.

---

### 35. What are common pitfalls when building RAG systems in production?
- One-size-fits-all chunk size instead of tuning per document type
- No re-ranking step, so the LLM sees noisy/irrelevant top-k results
- Not handling the "no relevant document found" case gracefully (model hallucinates instead of saying so)
- Ignoring metadata filtering/access control, leaking documents across tenants/users
- Not re-indexing when the underlying knowledge base changes (stale embeddings)
- No evaluation pipeline — shipping changes to chunking/retrieval without measuring regression in faithfulness or recall

---
