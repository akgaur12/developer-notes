# 🧭 Vector Databases Interview Q&A

## 🔹 Fundamentals

### 1. What is a Vector Database?
A database purpose-built to store high-dimensional **embedding vectors** and perform fast **similarity search** over them — finding the vectors "closest" to a query vector — instead of exact-match lookups like traditional relational/NoSQL databases.

---

### 2. Why can't a traditional relational database efficiently handle vector similarity search?
Relational databases are optimized for exact-match and range queries using B-tree indexes, which don't help for **"find the nearest neighbors in N-dimensional space"** queries. A brute-force scan comparing a query vector against every stored vector is `O(n)` per query and becomes too slow at scale (millions/billions of vectors) — vector databases use specialized **Approximate Nearest Neighbor (ANN)** indexes to make this fast.

---

### 3. What is an Embedding, in the context of vector databases?
A dense numerical vector (typically hundreds to a few thousand dimensions) produced by an embedding model, representing the semantic meaning of a piece of data (text, image, audio) — vectors for semantically similar inputs end up close together in the vector space.

---

### 4. What are the main components/operations of a vector database?
- **Upsert** – insert or update a vector + its metadata + an ID
- **Query/Search** – find the top-k nearest vectors to a query vector
- **Delete** – remove a vector by ID
- **Filtering** – restrict search to vectors matching metadata conditions
- **Indexing** – building/maintaining the ANN index structure over stored vectors

---

## 🔹 Similarity Metrics

### 5. What similarity/distance metrics are commonly used?
- **Cosine similarity** – measures the angle between two vectors, ignoring magnitude; most common for text embeddings
- **Dot product (inner product)** – equivalent to cosine similarity if vectors are normalized to unit length; used when the embedding model was trained with a dot-product objective
- **Euclidean (L2) distance** – straight-line distance between points; less common for high-dimensional text embeddings but used in some domains (e.g. image embeddings)

---

### 6. Why does cosine similarity dominate for text embeddings specifically?
Text embedding models are typically trained so that **direction** in vector space encodes meaning, while magnitude can vary with factors like text length — cosine similarity normalizes out magnitude and focuses purely on directional (semantic) alignment.

---

### 7. What does it mean to "normalize" vectors, and why does it matter for search?
Normalizing scales a vector to unit length (magnitude = 1). Once vectors are normalized, cosine similarity and dot product become **mathematically equivalent**, and many ANN indexes can use the cheaper dot-product computation while still effectively performing cosine similarity search.

---

## 🔹 Approximate Nearest Neighbor (ANN) Indexing

### 8. What is Approximate Nearest Neighbor (ANN) search, and why "approximate"?
Exact nearest-neighbor search over millions of high-dimensional vectors is computationally expensive. ANN algorithms trade a small amount of **accuracy (recall)** for large gains in **speed**, returning neighbors that are very likely — but not mathematically guaranteed — to be the true closest matches.

---

### 9. What is HNSW (Hierarchical Navigable Small World)?
A graph-based ANN algorithm that builds a **multi-layer graph** where each vector is a node connected to its approximate nearest neighbors; search starts at a sparse top layer and greedily navigates down through denser layers toward the query's neighborhood. It's the most widely used default in modern vector databases due to its strong speed/recall trade-off.

---

### 10. What is IVF (Inverted File Index)?
An ANN method that first **clusters** all vectors (e.g. via k-means) into buckets ("cells"), then at query time only searches the vectors within the **nearest few clusters** to the query, instead of the entire dataset — trading some recall for a large reduction in comparisons.

---

### 11. What is Product Quantization (PQ), and why combine it with IVF (IVF-PQ)?
PQ **compresses** each vector by splitting it into sub-vectors and quantizing each to a small codebook index, dramatically reducing memory footprint at the cost of some precision. Combined as **IVF-PQ**, IVF narrows the search to relevant clusters while PQ keeps the compressed vectors small enough to fit in memory/be compared cheaply — common for very large-scale deployments.

---

### 12. What is Scalar Quantization, and how does it compare to Product Quantization?
Scalar quantization reduces the precision of each vector dimension independently (e.g. float32 → int8), which is simpler and faster than PQ but generally compresses less aggressively — a middle ground between full-precision vectors and PQ's higher compression ratio.

---

### 13. What is the recall/latency/memory trade-off in ANN index tuning?
Nearly every ANN index exposes tunable parameters (e.g. HNSW's `ef_search`, IVF's `nprobe`) that trade **search accuracy (recall)** against **query latency** and **memory usage** — increasing these parameters searches more of the index (higher recall, slower), decreasing them searches less (faster, lower recall).

---

### 14. When would you use exact (brute-force/flat) search instead of an ANN index?
For small datasets (a few thousand to low tens of thousands of vectors) where a full linear scan is still fast enough, or when 100% recall is required and dataset size makes brute-force computation feasible — ANN indexes add complexity/approximation that isn't worth it below a certain scale.

---

## 🔹 Retrieval Features

### 15. What is Metadata Filtering, and why is it essential in production vector search?
The ability to combine vector similarity search with **structured filters** on associated metadata (e.g. `date > 2024`, `tenant_id = X`, `category = "finance"`) — critical for multi-tenancy, access control, and narrowing results by known constraints rather than relying on similarity alone.

---

### 16. What is Pre-filtering vs Post-filtering in filtered vector search, and why does it matter?
- **Pre-filtering** – restrict the candidate set to metadata-matching vectors *before* running ANN search
- **Post-filtering** – run ANN search first, then discard results that don't match the filter *afterward*
Post-filtering can return **fewer than k results** (or none) if most of the true top-k matches get filtered out — a common gotcha when filters are highly selective. Good vector databases support efficient pre-filtering integrated into the ANN traversal itself.

---

### 17. What is Hybrid Search, and how is it implemented in a vector database?
Combining **dense vector similarity** (semantic) with **sparse keyword-based search** (e.g. BM25) — often via **Reciprocal Rank Fusion (RRF)** to merge the two ranked result lists — to catch both semantically related content and exact keyword/entity matches that pure embedding similarity can miss.

---

### 18. What is namespace/collection partitioning, and why use it?
Logically separating vectors into distinct namespaces/collections (e.g. per tenant, per document type) so queries and index maintenance scope naturally to a subset of data — improving both query performance and data isolation without needing per-query metadata filters for basic separation.

---

## 🔹 Comparisons & Ecosystem

### 19. Dedicated Vector Database vs `pgvector` (Postgres extension) — how do you choose?
| Dedicated (Pinecone, Weaviate, Milvus, Qdrant) | pgvector |
|----|----|
| Purpose-built for scale, ANN-optimized | Good for small/medium scale, leverages existing Postgres |
| Separate system to operate/scale | No new infra — reuse existing Postgres deployment |
| Richer native filtering/hybrid search features | Filtering benefits from mature SQL joins/transactions |
| Better at very large scale (100M+ vectors) | Simpler ops when vector search is a secondary feature of an app already on Postgres |

---

### 20. What are the main managed/self-hosted vector database options, and how do they differ at a high level?
- **Pinecone** – fully managed, serverless, simple API, strong at scale
- **Weaviate** – open-source, built-in hybrid search and modules (e.g. auto-embedding)
- **Milvus** – open-source, highly scalable, supports multiple index types, popular for large deployments
- **Qdrant** – open-source (Rust), strong filtering performance, easy self-hosting
- **Chroma** – lightweight, developer-friendly, popular for local/prototype RAG apps
- **pgvector** – Postgres extension, best when you already run Postgres and don't need extreme scale
- **Elasticsearch/OpenSearch (vector search)** – good when you already need full-text + vector search combined

---

### 21. What criteria should drive choosing a vector database for a production system?
- Expected **scale** (number of vectors, QPS)
- Need for **hybrid search** / advanced metadata filtering
- **Managed vs self-hosted** operational preference and cost model
- Existing infrastructure (e.g. already on Postgres → pgvector may be simplest)
- **Latency requirements** and index-tuning flexibility
- Multi-tenancy/isolation requirements

---

## 🔹 Operations & Production Concerns

### 22. What challenges arise from updating/deleting vectors in an ANN index?
Many ANN structures (especially graph-based ones like HNSW) are not naturally designed for frequent deletes — deletions are often handled via **tombstoning** (marking as deleted, filtering at query time) with periodic **compaction/rebuild** of the index, since true in-place removal can be costly or degrade index quality over time.

---

### 23. What is re-indexing, and when is it necessary?
Rebuilding the ANN index from scratch — necessary when you change the embedding model (all vectors need to move to the new embedding space), significantly change index parameters, or after heavy delete/update churn has degraded index quality/recall.

---

### 24. How do you handle scaling a vector database horizontally?
- **Sharding** – partitioning vectors across multiple nodes (e.g. by namespace or hash of ID), each handling a subset of the index
- **Replication** – duplicating shards across nodes for read throughput and fault tolerance
- Query fan-out across shards with a final merge/re-rank step to produce the overall top-k results

---

### 25. What is "eventual consistency" in the context of vector databases, and why does it happen?
Many vector databases (especially distributed/managed ones) don't guarantee a freshly upserted vector is **immediately searchable** — there can be a short indexing delay before it appears in ANN search results, since building/updating the ANN structure isn't instantaneous. Applications needing immediate read-after-write consistency should account for this (e.g. via a short delay or a fallback exact-match check).

---

### 26. How much memory does a large vector index typically require, and how is that managed?
Raw float32 embeddings at scale can require huge amounts of RAM (e.g. a 1B-vector, 768-dim, float32 index is roughly 3TB uncompressed) — managed via **quantization** (PQ, scalar quantization, or newer binary quantization), reducing per-vector memory footprint by 4–32x at some accuracy cost, or via disk-based/hybrid memory-disk indexes for datasets too large to fit fully in RAM.

---

### 27. How would you monitor a vector database in production?
- **Recall/accuracy** – periodically validate ANN search quality against exact search on a sample
- **Query latency** (p50/p95/p99) and throughput (QPS)
- **Index build/update lag** – time between upsert and searchability
- **Memory/disk usage** growth as the dataset scales
- **Error rates** on upserts/queries

---

## 🔹 Application Design

### 28. How does chunk size choice for a RAG pipeline interact with vector database design?
Smaller chunks mean **more vectors** to store/index (higher memory/index cost, but more precise individual matches); larger chunks mean **fewer, more diluted** vectors. This directly affects index size, query latency, and the pre-filtering/metadata design (e.g. tracking parent-document IDs to reconstruct full context from small indexed chunks).

---

### 29. Why might you store multiple embeddings per document (e.g. one per chunk, one per summary)?
Different embeddings capture different granularities — a chunk-level embedding is good for precise fact retrieval, while a document/summary-level embedding is better for broad topical matching or routing a query to the right document before drilling into chunks (a form of **hierarchical/multi-vector retrieval**).

---

### 30. What are common pitfalls when using a vector database in production?
- Not normalizing vectors when the chosen metric assumes it (mismatched cosine/dot-product usage)
- Relying on post-filtering with highly selective metadata filters, silently returning fewer results than expected
- Ignoring index-tuning parameters (e.g. `ef_search`/`nprobe`), leaving recall or latency far from optimal defaults
- Not planning for re-indexing when swapping embedding models — old and new vectors are **not comparable**
- Underestimating memory requirements at scale, discovering it only after hitting OOM in production
- Treating "eventual consistency" indexing delay as a bug instead of designing around it

---
