# ANN Search Fundamentals

## Staff/Principal Level — Database Systems for AI Engineers

---

## 1. The Core Problem: Finding Similar Vectors at Scale

At the heart of every vector database lies a single operation: given a query vector **q** in
d-dimensional space, find the **k** vectors from a corpus of **N** vectors that are closest to **q**
under some distance metric. This is the **k-Nearest Neighbor (k-NN)** problem.

```
Exact k-NN:   O(N * d)  per query   — scans every vector
Approximate:  O(log N)  to O(N^α)   — trades recall for speed, α << 1
```

For a corpus of 1 billion 768-dimensional vectors, exact search requires ~3 trillion floating
point operations per query. At 1 TFLOP/s, that is ~3 seconds — utterly unacceptable for
real-time serving.

---

## 2. Distance Metrics Deep Dive

### 2.1 Metric Comparison Table

| Metric | Formula | Range | Normalized? | Common Use Cases |
|---|---|---|---|---|
| **L2 (Euclidean)** | `sqrt(sum((a_i - b_i)^2))` | [0, inf) | No | Image embeddings, face recognition |
| **Cosine Similarity** | `dot(a,b) / (||a|| * ||b||)` | [-1, 1] | Yes | Text embeddings, semantic search |
| **Inner Product (Dot)** | `sum(a_i * b_i)` | (-inf, inf) | No | Maximum inner product search (MIPS), recommendation |
| **Hamming** | `popcount(a XOR b)` | [0, d] | N/A (binary) | Binary hash codes, locality-sensitive hashing |
| **Manhattan (L1)** | `sum(|a_i - b_i|)` | [0, inf) | No | Sparse feature spaces |

### 2.2 Critical Equivalences

```
When vectors are L2-normalized (||v|| = 1):

  cosine_similarity(a, b) = dot(a, b)
  L2_distance(a, b)^2     = 2 - 2 * dot(a, b)

  => Ranking under cosine ≡ ranking under L2 ≡ ranking under dot product
     for unit-norm vectors.

Implication: normalize once at ingestion time, then use the fastest metric
             (usually dot product — a single SIMD instruction sequence).
```

### 2.3 When to Use Which Metric

```
Decision Tree:

  Are your vectors already L2-normalized?
  ├── YES → Use Inner Product (fastest, SIMD-friendly)
  ├── NO  → Is magnitude meaningful?
  │         ├── YES → Use L2 (e.g., pixel distances, spatial coords)
  │         └── NO  → Normalize, then use Inner Product
  └── Binary codes? → Hamming (popcount is single CPU instruction)
```

**Staff-level insight:** Many production systems silently normalize at ingestion and use IP
internally regardless of the user-facing API. Pinecone's "cosine" metric, for example, stores
normalized vectors and executes dot product — saving a division per comparison.

---

## 3. The Curse of Dimensionality

### 3.1 Why High Dimensions Break Intuition

In high-dimensional spaces (d > 20), the ratio of the distance to the nearest neighbor vs. the
farthest neighbor converges to 1:

```
                    dist(q, farthest)
  lim (d → ∞)   ─────────────────────  →  1
                    dist(q, nearest)

  "All points become equidistant."
```

This has devastating consequences:
- **Space partitioning** (KD-trees) degenerates to linear scan above ~20 dimensions
- **Ball trees** similarly fail — pruning becomes ineffective
- **LSH** hash collisions lose discriminative power without more hash tables

### 3.2 Concentration of Distances

```
  ┌──────────────────────────────────────────────────────┐
  │  d=2                    d=128                        │
  │                                                      │
  │  ****                      ***********               │
  │ **    **                 **           **              │
  │*        *               *               *            │
  │          *             *                 *            │
  │           **        ***                   *           │
  │  ───────────────  ────────────────────────────────   │
  │  Distance →       Distance →                         │
  │                                                      │
  │  Wide spread       Tight concentration               │
  │  (easy to          (hard to distinguish               │
  │   distinguish)      neighbors from non-neighbors)    │
  └──────────────────────────────────────────────────────┘
```

### 3.3 Practical Mitigations

| Strategy | How It Helps | Tradeoff |
|---|---|---|
| Dimensionality reduction (PCA, UMAP) | Reduces d, spreads distances | Loses information, 5-15% recall drop |
| Product Quantization (PQ) | Compresses vectors, works in subspaces | Approximation error, tuning complexity |
| Scalar Quantization (SQ8) | Reduces memory 4x (float32 → int8) | Small recall loss (~1-2%) |
| Binary Quantization | 32x compression | Significant recall loss, needs reranking |

---

## 4. Exact vs Approximate Nearest Neighbor

### 4.1 Architecture Comparison

```
  ┌─────────────────────────────────────────────────────────┐
  │                  EXACT (Brute Force)                     │
  │                                                         │
  │  Query → [Scan ALL N vectors] → Sort top-k → Return    │
  │                                                         │
  │  Guarantees: 100% recall, deterministic                 │
  │  Cost:       O(N*d) compute, O(N*d) memory              │
  │  Latency:    Linear in N — fails above ~1M vectors      │
  └─────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────┐
  │                  APPROXIMATE (ANN)                       │
  │                                                         │
  │  Offline: Build index structure (graph/tree/hash)       │
  │  Query:   Traverse index → Candidate set → Rerank      │
  │                                                         │
  │  Guarantees: Recall@k ∈ [0.9, 0.999] (tunable)         │
  │  Cost:       O(log N) to O(N^0.5) compute               │
  │  Latency:    Sub-millisecond at billion scale           │
  └─────────────────────────────────────────────────────────┘
```

### 4.2 When Exact Search Is Actually Fine

| Scenario | Typical N | Why Exact Works |
|---|---|---|
| Internal tooling / batch pipelines | < 100K | Latency doesn't matter |
| Prototype / MVP | < 500K | Correctness > speed |
| Reranking stage (after ANN) | 100-1000 | Tiny candidate set |
| GPU-accelerated brute force | < 10M | FAISS `GpuIndexFlat` saturates bandwidth |

**Key formula for brute-force feasibility:**

```
  Latency ≈ (N * d * 2) / memory_bandwidth

  Example: N=1M, d=768, bandwidth=50 GB/s
  = (1M * 768 * 4 bytes * 2) / 50 GB/s
  ≈ 120 ms  ← borderline for real-time
```

---

## 5. Recall vs Latency Tradeoffs

### 5.1 The Fundamental Tradeoff Curve

```
  Recall@10
  1.00 ┤                                          ●  (brute force)
       │                                     ●
  0.99 ┤                                ●
       │                           ●
  0.98 ┤                      ●
       │                 ●
  0.95 ┤            ●
       │       ●
  0.90 ┤  ●
       │
  0.80 ┤●
       └──┬────┬────┬────┬────┬────┬────┬────┬───
         0.1  0.5   1    2    5   10   50  100  ms
                      Query Latency (log scale)

  Every ANN algorithm traces a curve in this space.
  The "best" algorithm has the highest recall at a given latency.
```

### 5.2 Recall Definition and Variants

```
  Recall@k = |ANN_result ∩ Exact_result| / k

  Variants:
  - Recall@10:  Standard benchmark metric
  - Recall@100: Used when reranking stage follows
  - Recall@1:   Critical for deduplication use cases
  - Recall@k with filtering: Much harder — metadata filters reduce candidate pool
```

### 5.3 Recall Requirements by Use Case

| Use Case | Target Recall@10 | Latency Budget | Rationale |
|---|---|---|---|
| E-commerce search | 0.90-0.95 | < 50ms | Diversity matters more than perfection |
| RAG retrieval | 0.95-0.99 | < 100ms | Missing a relevant chunk degrades LLM output |
| Duplicate detection | 0.999+ | < 200ms | False negatives = duplicates slip through |
| Recommendation | 0.85-0.95 | < 20ms | Approximate is fine, latency critical |
| Anomaly detection | 0.99+ | < 1s | Missing anomalies has safety implications |

---

## 6. Benchmarking Methodology

### 6.1 ANN-Benchmarks Framework

The gold standard: [ann-benchmarks.com](http://ann-benchmarks.com)

```
  Benchmark Protocol:
  ┌───────────────────────────────────────────────────┐
  │ 1. Index build phase   — timed separately         │
  │ 2. Query phase         — batch of 10K queries     │
  │ 3. Compute recall      — against brute-force GT   │
  │ 4. Plot recall vs QPS  — the Pareto frontier      │
  └───────────────────────────────────────────────────┘

  Standard Datasets:
  - SIFT-1M    (128d, 1M vectors)   — baseline
  - GloVe-200  (200d, 1.2M vectors) — text embeddings
  - GIST-1M    (960d, 1M vectors)   — high dimensional
  - Deep-1B    (96d, 1B vectors)    — billion scale
```

### 6.2 What Benchmarks Miss (Staff-Level Critique)

| Real-World Factor | Benchmark Behavior | Production Reality |
|---|---|---|
| Concurrent queries | Single-threaded | 1000+ QPS with contention |
| Insert during query | Static index | Streaming inserts cause lock contention |
| Metadata filtering | No filters | Filters can reduce recall 10-30% |
| Memory pressure | Unlimited RAM | Shared with OS, other services |
| Tail latency (p99) | Reports median/mean | p99 matters for SLAs |
| Vector distribution shift | Fixed dataset | Embeddings change with model updates |

### 6.3 Building Your Own Benchmark

```python
# Staff-level benchmark template
def benchmark_vector_db(db, dataset, queries, ground_truth, k=10):
    """Measures what ANN-benchmarks does NOT."""

    results = {
        "recall": [],
        "latency_p50": [],
        "latency_p99": [],
        "latency_p999": [],
        "qps_at_recall_95": None,
        "memory_rss_gb": None,
        "recall_with_filters": [],
        "recall_during_ingest": [],
    }

    # Phase 1: Measure recall + latency at various search params
    for ef_search in [16, 32, 64, 128, 256, 512]:
        latencies = []
        recalls = []
        for q, gt in zip(queries, ground_truth):
            t0 = time.perf_counter_ns()
            res = db.search(q, k=k, ef_search=ef_search)
            latencies.append(time.perf_counter_ns() - t0)
            recalls.append(len(set(res) & set(gt[:k])) / k)

        # Report percentiles, not averages
        results["latency_p99"].append(np.percentile(latencies, 99))

    # Phase 2: Measure recall degradation during concurrent inserts
    # Phase 3: Measure recall with metadata filters
    # Phase 4: Measure memory under production load
    return results
```

---

## 7. Advanced Topics

### 7.1 Filtered Search and the "Filter Paradox"

```
  The Filter Paradox:
  ───────────────────
  If your filter matches 1% of vectors, ANN must explore 100x more
  candidates to find k results. This can degrade recall OR latency.

  Strategies:
  ┌──────────────────┬───────────────────────────────────────────┐
  │ Pre-filtering     │ Filter first, then ANN on subset.        │
  │                   │ Fast if filter is selective.              │
  │                   │ Fails if subset too small for ANN index. │
  ├──────────────────┼───────────────────────────────────────────┤
  │ Post-filtering    │ ANN first, then filter results.          │
  │                   │ May return < k results.                  │
  │                   │ Wastes compute on filtered-out vectors.  │
  ├──────────────────┼───────────────────────────────────────────┤
  │ In-filter search  │ Integrated filtering during traversal.   │
  │ (Weaviate,Qdrant) │ Best recall, most complex to implement.  │
  └──────────────────┴───────────────────────────────────────────┘
```

### 7.2 Multi-Vector and Late Interaction

```
  Traditional:  doc → single vector (768d)
  ColBERT:      doc → N token vectors (128d each)

  Scoring: MaxSim — for each query token, find max similarity
           across all document token vectors, then sum.

  Storage implication: 100x more vectors per document.
  Index implication:  Need document-level aggregation after ANN.
```

### 7.3 Hybrid Search (Sparse + Dense)

```
  ┌──────────────┐     ┌──────────────┐
  │  BM25 / SPLADE│     │  Dense ANN   │
  │  (sparse)     │     │  (vectors)   │
  └──────┬───────┘     └──────┬───────┘
         │                     │
         ▼                     ▼
    ┌─────────────────────────────┐
    │   Reciprocal Rank Fusion    │
    │   or Linear Combination     │
    │   score = α*dense + β*sparse│
    └─────────────┬───────────────┘
                  ▼
           Final top-k results
```

---

## 8. Failure Modes in ANN Search

| Failure Mode | Symptoms | Root Cause | Mitigation |
|---|---|---|---|
| Recall cliff | Sudden recall drop at scale | Index parameters tuned for small data | Re-tune at target scale |
| Hubness | Same vectors appear in many results | High-dimensional phenomenon | Use CSLS (cross-domain similarity local scaling) |
| Distribution mismatch | Low recall on new data | Training data != query distribution | Periodic reindexing |
| Quantization collapse | Distinct vectors map to same code | Too few PQ centroids | Increase PQ segments (m) |
| Filter starvation | < k results returned | Overly selective filters + post-filtering | Switch to pre-filter or in-filter |

---

## 9. Interview Narrative

### How to Present This in a Staff/Principal Interview

**Opening framing (30 seconds):**
> "Vector search is fundamentally a tradeoff between recall, latency, and memory. The exact
> algorithm choice matters less than understanding where you are on that Pareto frontier and
> whether your system's recall requirements match the operational constraints."

**When asked "How would you design a vector search system?":**
1. **Start with requirements:** What's the recall target? Latency SLA? Corpus size? Update frequency?
2. **Choose the metric:** Clarify whether magnitude matters. Default to cosine (normalized IP).
3. **Size the problem:** < 1M vectors? Brute force on GPU. 1M-100M? HNSW in-memory. 100M+? Distributed with IVF-PQ or DiskANN.
4. **Address the hard parts:** Filtered search, streaming updates, reindexing strategy.
5. **Discuss observability:** How do you detect recall degradation? Canary queries with known ground truth.

**Differentiating signal for E7:**
- Discuss the tension between benchmarks and production (concurrent writes, filter selectivity).
- Explain why recall@10 on ANN-benchmarks is misleading when you have metadata filters.
- Propose a system for monitoring embedding quality drift and triggering reindex.
- Articulate cost modeling: memory cost per vector, QPS per dollar, recall per dollar.

**Red flags to avoid:**
- Don't say "just use cosine similarity" without discussing normalization.
- Don't ignore the curse of dimensionality when proposing KD-trees.
- Don't propose billion-scale brute force without GPU cost justification.
- Don't conflate recall@k with precision@k.
