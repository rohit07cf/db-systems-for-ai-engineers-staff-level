# HNSW vs IVF vs Flat: Deep Index Comparison

## Staff/Principal Level — Database Systems for AI Engineers

---

## 1. Index Family Overview

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    ANN Index Taxonomy                           │
  │                                                                 │
  │  Graph-based              Partition-based        Brute Force    │
  │  ┌──────────┐             ┌──────────────┐       ┌──────────┐  │
  │  │  HNSW    │             │  IVF (Flat)  │       │  Flat    │  │
  │  │  NSW     │             │  IVF-PQ      │       │  (exact) │  │
  │  │  Vamana  │             │  IVF-SQ8     │       │          │  │
  │  │ (DiskANN)│             │  ScaNN       │       │          │  │
  │  └──────────┘             └──────────────┘       └──────────┘  │
  │                                                                 │
  │  Tree-based               Hash-based                           │
  │  ┌──────────┐             ┌──────────────┐                     │
  │  │  Annoy   │             │  LSH         │                     │
  │  │  KD-tree │             │  Multi-probe │                     │
  │  │  Ball    │             │  E2LSH       │                     │
  │  │  tree    │             │              │                     │
  │  └──────────┘             └──────────────┘                     │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 2. Flat Index (Brute Force)

### 2.1 How It Works

```
  Query Processing:
  ┌────────────────────────────────────────────────────┐
  │                                                    │
  │  q ──→ Compare against ALL N vectors ──→ Top-k    │
  │        using SIMD-optimized distance               │
  │                                                    │
  │  No preprocessing. No approximation. No index.     │
  │                                                    │
  └────────────────────────────────────────────────────┘
```

### 2.2 Characteristics

| Property | Value |
|---|---|
| Build time | O(1) — just store vectors |
| Query time | O(N * d) |
| Memory | O(N * d * 4) bytes (float32) |
| Recall | 1.0 (exact) |
| Update cost | O(1) append |
| Supports delete | Yes (mark + skip) |

### 2.3 When Flat Wins

- **N < 50K:** Overhead of index construction exceeds brute-force cost
- **GPU available:** `faiss.GpuIndexFlat` handles 10M vectors in < 5ms
- **Reranking stage:** After ANN returns 1000 candidates, flat rerank is trivial
- **Ground truth generation:** Always use flat for benchmark baselines

---

## 3. HNSW (Hierarchical Navigable Small World)

### 3.1 Architecture

```
  Layer 3 (sparse):    A ──────────────── B
                       │                  │
  Layer 2:             A ── C ──── D ──── B
                       │    │      │      │
  Layer 1:             A ── C ── E ── D ── B ── F
                       │    │    │    │    │    │
  Layer 0 (dense):     A ── C ── E ── D ── B ── F ── G ── H ── I
                       │    │    │    │    │    │    │    │    │
                      (all vectors present at layer 0)

  Key insight: Upper layers act as "express lanes" for navigation.
  Search starts at top layer, greedily descends, refines at layer 0.
```

### 3.2 Construction Algorithm

```
  INSERT(vector v, max_layer l):
    1. Randomly assign layer l = floor(-ln(rand()) * m_L)
       where m_L = 1/ln(M)  — higher M → more layers
    2. From top layer down to l+1:
       - Greedy search to find nearest node (entry point refinement)
    3. From layer l down to 0:
       - Find efConstruction nearest neighbors
       - Connect v to M nearest (M_max0 at layer 0)
       - Prune connections that exceed M_max per node
```

### 3.3 Search Algorithm

```
  SEARCH(query q, k, efSearch):
    1. Start at entry point (top layer)
    2. For each layer from top to 1:
       - Greedy search: move to nearest neighbor, repeat until local min
    3. At layer 0:
       - Beam search with beam width = efSearch
       - Maintain priority queue of efSearch candidates
       - Explore neighbors of best candidates
    4. Return top-k from candidate set
```

### 3.4 Tuning Parameters

| Parameter | Controls | Typical Range | Impact |
|---|---|---|---|
| **M** | Edges per node per layer | 8-64 | Higher = better recall, more memory |
| **M_max0** | Edges at layer 0 | 2*M | Dense base layer connectivity |
| **efConstruction** | Build-time beam width | 100-500 | Higher = better graph quality, slower build |
| **efSearch** | Query-time beam width | 50-500 | Higher = better recall, slower query |
| **m_L** | Layer assignment multiplier | 1/ln(M) | Usually auto-calculated |

### 3.5 Memory Model

```
  Memory per vector =
    sizeof(vector)          → d * 4 bytes (float32)
  + sizeof(adjacency_list)  → avg_degree * 8 bytes (int64 neighbor IDs)
  + overhead                → ~40 bytes (metadata, layer info)

  Example: d=768, M=16
    Vector:      768 * 4     = 3,072 bytes
    Neighbors:   ~32 * 8     =   256 bytes  (avg across layers)
    Overhead:                     40 bytes
    Total:                     3,368 bytes per vector

  For 10M vectors: ~33.7 GB  — must fit in RAM
```

---

## 4. IVF (Inverted File Index)

### 4.1 Architecture

```
  Training Phase (k-means):
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │  Run k-means on sample of vectors → nlist centroids  │
  │                                                      │
  │   C1      C2      C3      C4    ...    C_nlist       │
  │    ●       ●       ●       ●              ●          │
  │   /|\     /|\     /|\     /|\            /|\         │
  │  / | \   / | \   / | \   / | \          / | \        │
  │ v1 v2 v3 v4 v5 v6 v7 v8 v9 ...        ...           │
  │                                                      │
  │  Each vector assigned to its nearest centroid.        │
  └──────────────────────────────────────────────────────┘

  Query Phase:
  ┌──────────────────────────────────────────────────────┐
  │ 1. Compare query to all nlist centroids              │
  │ 2. Select top nprobe closest centroids               │
  │ 3. Scan ALL vectors in those nprobe clusters         │
  │ 4. Return top-k                                      │
  └──────────────────────────────────────────────────────┘
```

### 4.2 IVF Variants

```
  ┌────────────────────────────────────────────────────────────┐
  │ IVF-Flat:  Stores full vectors in each cluster.            │
  │            Exact distances within probed clusters.          │
  │            Memory = O(N * d * 4)                           │
  │                                                            │
  │ IVF-PQ:   Product Quantization compresses residual         │
  │            vectors. Splits d dims into m subspaces,        │
  │            quantizes each to 8-bit code.                   │
  │            Memory = O(N * m) — dramatic compression        │
  │                                                            │
  │ IVF-SQ8:  Scalar quantization: float32 → int8             │
  │            4x memory reduction, minimal recall loss.       │
  │            Memory = O(N * d)                               │
  │                                                            │
  │ IVF-PQ+R: PQ with re-ranking using stored full vectors    │
  │            Best recall, but needs full vectors in memory.  │
  └────────────────────────────────────────────────────────────┘
```

### 4.3 Product Quantization Deep Dive

```
  Original vector (d=768):
  [v1, v2, ..., v768]

  Split into m=96 subvectors of d/m=8 dimensions:
  [v1..v8] [v9..v16] ... [v761..v768]
     ↓          ↓              ↓
  Quantize each subvector to nearest of 256 centroids:
    code_1     code_2    ...  code_96

  Storage: 96 bytes instead of 3072 bytes = 32x compression

  Distance computation:
  - Precompute distance table: query to all 256 centroids × 96 subspaces
  - Table size: 256 * 96 * 4 = ~98 KB (fits in L2 cache)
  - Per-vector distance: 96 table lookups + 96 additions
  - Asymmetric distance computation (ADC) for best accuracy
```

### 4.4 IVF Tuning Parameters

| Parameter | Controls | Typical Range | Impact |
|---|---|---|---|
| **nlist** | Number of clusters | sqrt(N) to 4*sqrt(N) | More = finer partition, longer training |
| **nprobe** | Clusters searched per query | 1 to nlist | Higher = better recall, linear latency |
| **m** (PQ) | Number of subquantizers | 8 to d/2 | Higher = better recall, more memory |
| **nbits** (PQ) | Bits per subquantizer | 8 (standard) | 8 = 256 centroids per subspace |
| **training_size** | Vectors for k-means | 30*nlist to 256*nlist | More = better centroids |

---

## 5. Head-to-Head Comparison

### 5.1 Performance at Different Scales

| Scale | Flat | HNSW | IVF-Flat | IVF-PQ |
|---|---|---|---|---|
| **10K vectors** | 0.1ms, R=1.0 | 0.05ms, R=0.99 | 0.3ms, R=0.95 | 0.5ms, R=0.90 |
| **1M vectors** | 50ms, R=1.0 | 0.5ms, R=0.99 | 2ms, R=0.95 | 1ms, R=0.90 |
| **100M vectors** | 5000ms, R=1.0 | 1ms, R=0.98 | 5ms, R=0.93 | 3ms, R=0.85 |
| **1B vectors** | Infeasible | OOM (>300GB) | 10ms, R=0.90 | 5ms, R=0.80 |

*(d=768, k=10, representative single-threaded latencies)*

### 5.2 Resource Comparison

| Resource | Flat | HNSW | IVF-Flat | IVF-PQ (m=96) |
|---|---|---|---|---|
| **Memory per vector** | 3072 B | ~3400 B | 3072 B + overhead | ~100 B |
| **Memory for 100M** | 307 GB | ~340 GB | 307 GB | 10 GB |
| **Build time (100M)** | 0 | 2-6 hours | 30-60 min | 1-2 hours |
| **Supports streaming insert** | Yes | Yes* | No (retrain) | No (retrain) |
| **On-disk viable** | Yes (slow) | Poor | Yes | Yes |

*HNSW supports inserts but quality degrades without global optimization.*

### 5.3 Tradeoff Matrix

```
                    ┌──────────────────────────────────────────┐
                    │          Memory Budget                    │
                    │    Low            Medium          High    │
    ┌───────────────┼──────────────────────────────────────────┤
    │ < 1M vectors  │  IVF-PQ          IVF-Flat        HNSW   │
  S │               │  (overkill       (good           (best   │
  c │               │   but cheap)      balance)        recall)│
  a ├───────────────┼──────────────────────────────────────────┤
  l │ 1M - 100M     │  IVF-PQ          HNSW            HNSW   │
  e │               │  (only option)   (if fits RAM)    (ideal)│
    ├───────────────┼──────────────────────────────────────────┤
    │ 100M - 1B     │  IVF-PQ          IVF-PQ+Disk     DiskANN│
    │               │  (compressed)    (tiered)         (SSD)  │
    ├───────────────┼──────────────────────────────────────────┤
    │ > 1B          │  IVF-PQ          Distributed      Custom │
    │               │  (sharded)       IVF-PQ           hybrid │
    └───────────────┴──────────────────────────────────────────┘
```

---

## 6. Construction Time Deep Dive

### 6.1 Build Time Comparison

```
  Build Time (hours) for 100M vectors, d=768, 32 cores:

  Flat:            ████  ~0 (just copy data)
  IVF-Flat:        ████████████████  ~1h (k-means training)
  IVF-PQ:          ████████████████████████  ~2h (k-means + PQ training)
  HNSW (M=16):     ████████████████████████████████████████  ~4h
  HNSW (M=32):     ████████████████████████████████████████████████  ~6h

  HNSW build is NOT parallelizable at the per-insertion level
  (graph modifications require locks), but can batch-parallelize
  using fine-grained locking strategies.
```

### 6.2 Build Time Formulas

```
  IVF:  T_build = T_kmeans(N, nlist, d) + T_assign(N, nlist, d)
        T_kmeans ≈ O(N * nlist * d * n_iter)
        Typically n_iter = 20

  HNSW: T_build = N * O(efConstruction * M * log(N))
        Each insertion does beam search + neighbor selection

  PQ:   T_build = O(N * m * 256 * d/m * n_iter) for PQ training
                 + O(N * m * 256 * d/m) for encoding
```

---

## 7. Query Path Analysis

### 7.1 HNSW Query Path

```
  ┌─────────────────────────────────────────────────┐
  │ HNSW Query: q, k=10, efSearch=128               │
  │                                                  │
  │ Layer 3: 1 greedy step      →  ~2 distance calcs│
  │ Layer 2: 2 greedy steps     →  ~8 distance calcs│
  │ Layer 1: 3 greedy steps     →  ~20 distance calc│
  │ Layer 0: beam search (ef=128)→ ~500-2000 calcs  │
  │                                                  │
  │ Total distance computations: ~600-2000           │
  │ For d=768: each calc = 768 multiplies + 767 adds│
  │                                                  │
  │ Cache behavior: POOR — random graph traversal    │
  │                 causes cache misses per hop       │
  └─────────────────────────────────────────────────┘
```

### 7.2 IVF Query Path

```
  ┌─────────────────────────────────────────────────┐
  │ IVF-PQ Query: q, k=10, nprobe=32, nlist=4096   │
  │                                                  │
  │ Step 1: Compare q to 4096 centroids              │
  │         → 4096 distance calcs (d=768)            │
  │                                                  │
  │ Step 2: Select top 32 clusters                   │
  │                                                  │
  │ Step 3: Build PQ distance table                  │
  │         → 256 * 96 = 24,576 distance calcs       │
  │         → 96 KB table (fits L2 cache!)           │
  │                                                  │
  │ Step 4: Scan ~(N/nlist)*nprobe vectors           │
  │         = (100M/4096)*32 = ~780K vectors         │
  │         Each: 96 lookups + 96 additions          │
  │                                                  │
  │ Cache behavior: EXCELLENT — sequential scan      │
  │                 + small lookup table in L2 cache  │
  └─────────────────────────────────────────────────┘
```

---

## 8. Failure Modes by Index Type

| Index | Failure Mode | Cause | Mitigation |
|---|---|---|---|
| **HNSW** | Memory OOM | All vectors + graph in RAM | Use SQ8 or offload to DiskANN |
| **HNSW** | Slow inserts at scale | Lock contention on graph | Batch inserts, segment-based arch |
| **HNSW** | Recall degradation after deletes | Tombstones fragment graph | Periodic rebuild |
| **IVF** | Unbalanced clusters | Skewed data distribution | Use balanced k-means, more clusters |
| **IVF** | Stale centroids | Data distribution shifts | Periodic retraining |
| **IVF-PQ** | Quantization error too high | d/m ratio too large | Increase m, use OPQ rotation |
| **Flat** | Latency wall | N too large | Graduate to ANN index |
| **All** | Recall drops with filters | Filtering reduces candidate pool | Use in-filter search |

---

## 9. Parameter Tuning Playbook

### 9.1 HNSW Tuning Guide

```
  Step 1: Start with M=16, efConstruction=200
  Step 2: Build index on representative data
  Step 3: Binary search on efSearch:
          - Start efSearch=64
          - If recall < target: double efSearch
          - If latency > budget: halve efSearch
  Step 4: If recall target unachievable:
          - Increase M (try 32, 48)
          - Increase efConstruction (400, 800)
          - Rebuild index
  Step 5: Memory check:
          - If OOM: reduce M, consider IVF-PQ
```

### 9.2 IVF Tuning Guide

```
  Step 1: Set nlist = 4 * sqrt(N)  (e.g., N=100M → nlist=40000)
  Step 2: Train on max(N, 256*nlist) vectors
  Step 3: Binary search on nprobe:
          - Start nprobe = nlist/100
          - Increase until recall target met
  Step 4: For PQ: start m = d/8, increase if recall low
  Step 5: Consider OPQ (optimized PQ with rotation matrix)
          if recall plateau hit
```

---

## 10. Real-World Selection Guide

### 10.1 Decision Framework

```
  ┌─ How many vectors?
  │
  ├─ < 100K
  │    └─ Use Flat (brute force). Done.
  │
  ├─ 100K - 5M
  │    ├─ Need streaming inserts? → HNSW
  │    ├─ Need lowest memory?     → IVF-PQ
  │    └─ Default                 → HNSW (M=16, efC=200)
  │
  ├─ 5M - 100M
  │    ├─ Fits in RAM? → HNSW (best recall)
  │    ├─ Memory constrained? → IVF-PQ
  │    └─ Need fast build? → IVF-SQ8
  │
  └─ > 100M
       ├─ SSD available? → DiskANN / Vamana
       ├─ GPU available? → FAISS GPU IVF-PQ
       └─ CPU only?     → Distributed IVF-PQ (sharded)
```

### 10.2 System-Specific Index Defaults

| Vector DB | Default Index | Notes |
|---|---|---|
| Pinecone | Proprietary (graph-based) | Auto-tuned, no user configuration |
| Weaviate | HNSW | With product quantization option |
| Qdrant | HNSW | With scalar/product quantization |
| Milvus | IVF-Flat, HNSW, DiskANN | User selects index type |
| Chroma | HNSW (via hnswlib) | Limited configuration |
| pgvector | IVFFlat, HNSW | HNSW added in 0.5.0 |

---

## 11. Interview Narrative

### How to Present This in a Staff/Principal Interview

**Opening framing (30 seconds):**
> "The three main index families — flat, graph-based (HNSW), and partition-based (IVF) — each
> occupy a distinct region in the recall-latency-memory tradeoff space. My approach is to start
> with the simplest option that meets requirements and graduate to more complex indexes only
> when forced by scale or latency constraints."

**When asked "Why would you choose HNSW over IVF?":**
> "HNSW gives best-in-class recall at low latency when the dataset fits in memory. It supports
> online inserts without retraining, which is critical for systems with streaming data. However,
> it has 10-15% memory overhead for the graph structure and poor cache locality during search.
> I'd choose IVF when memory is the binding constraint — IVF-PQ can compress vectors 30x,
> making billion-scale search feasible on commodity hardware."

**Demonstrating depth (E7 signal):**
- Explain PQ's asymmetric distance computation and why the lookup table fits in L2 cache
- Discuss HNSW's relationship to skip lists and how the exponential layer distribution works
- Analyze the cache behavior difference: HNSW's random access vs IVF-PQ's sequential scan
- Explain why IVF needs retraining when data distribution shifts but HNSW doesn't
- Discuss the "build vs buy" tension: FAISS gives you low-level control, managed services trade
  that for operational simplicity

**System design follow-up — "How would you handle 1B vectors?":**
> "At billion scale, no single machine can hold the full HNSW graph in memory. I'd use a
> sharded architecture: partition by consistent hashing, each shard holds an HNSW or IVF-PQ
> index. Query scatter-gather with latency-based routing. For cost optimization, I'd tier:
> hot data in HNSW (RAM), warm in DiskANN (SSD), cold in IVF-PQ on object storage with
> on-demand loading. This gives sub-10ms p99 for hot queries while controlling infrastructure
> cost to ~$0.10/1M vectors/month."

**Red flags to avoid:**
- Don't say "HNSW is always better" without discussing memory constraints.
- Don't forget that IVF requires a training step and can't handle distribution shift.
- Don't ignore the build time — HNSW on 100M vectors takes hours.
- Don't quote recall numbers without specifying k, dataset, and parameters used.
