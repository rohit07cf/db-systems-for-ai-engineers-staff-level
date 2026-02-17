# Vector Database -- Quick Reference

## Algorithm Comparison

| Property | HNSW | IVF-PQ | PQ (Product Quantization) | DiskANN | Brute Force |
|---|---|---|---|---|---|
| Index type | Graph | Inverted list + compression | Compression only | Graph on SSD | None |
| Build time | Slow (hours at 1B) | Moderate | Moderate | Moderate-slow | None |
| Query latency | **< 1 ms** (in-memory) | 1-10 ms | 5-20 ms | **1-10 ms** (SSD) | O(n) |
| Memory per vector (d=768) | ~3.1 KB + graph overhead | ~64-128 bytes (PQ) | ~64-128 bytes | ~128 bytes + graph on disk | 3.1 KB |
| Recall@10 | **95-99%** | 85-95% | 75-90% | **90-98%** | 100% |
| Scale ceiling | ~50M in-memory | Billions | Billions | **Billions on SSD** | < 100K |
| Best for | Low-latency, moderate scale | Large scale, cost-sensitive | Extreme compression | **Large scale on commodity hardware** | Small datasets, ground truth |

> **Interview Answer**: "For < 10M vectors we use HNSW in-memory for sub-millisecond latency.
> Beyond 100M, we switch to DiskANN or IVF-PQ to keep cost manageable."

## Distance Metric Selection Guide

| Metric | Formula | Use When | Normalized? |
|---|---|---|---|
| **Cosine similarity** | 1 - (A . B) / (\|A\| \|B\|) | Text embeddings, semantic search | Direction only |
| **Euclidean (L2)** | sqrt(sum((a_i - b_i)^2)) | Image embeddings, spatial data | Magnitude matters |
| **Dot product (IP)** | sum(a_i * b_i) | Pre-normalized vectors, MaxSim | Must be normalized |
| **Manhattan (L1)** | sum(\|a_i - b_i\|) | Sparse, high-dimensional | Magnitude matters |

```
Decision:
  Vectors already normalized? --> Dot Product (fastest)
  Text/NLP embeddings?        --> Cosine Similarity
  Image/spatial?              --> Euclidean (L2)
  Unsure?                     --> Cosine Similarity (safe default)
```

## Scale Thresholds -- When to Change Strategy

| Vector Count | Recommended Approach |
|---|---|
| < 100K | Brute force (pgvector, in-app) |
| 100K -- 1M | HNSW in-memory (single node) |
| 1M -- 10M | HNSW with sharding or managed service |
| 10M -- 100M | IVF-PQ or HNSW + quantization |
| 100M -- 1B | DiskANN, Milvus distributed, or Pinecone |
| > 1B | DiskANN + sharding, custom infra |

## Memory Estimation Formulas

```
Raw vector memory:
  bytes_per_vector = dimensions * 4            (float32)
  total_raw = num_vectors * bytes_per_vector

HNSW overhead:
  graph_memory = num_vectors * M * 2 * 8       (M = connections per node, bidirectional, 8 bytes per pointer)
  total_hnsw = total_raw + graph_memory
  Rule of thumb: 1.5x-2x raw vector size

PQ compressed:
  bytes_per_vector = num_subquantizers * 1      (typically 64-128 subquantizers)
  compression_ratio = (dimensions * 4) / num_subquantizers  (e.g., 768*4/96 = 32x)

Quick estimates (d=1536, float32):
  1M vectors raw:    ~5.7 GB
  1M vectors HNSW:   ~9-12 GB  (with M=16)
  1M vectors PQ-96:  ~96 MB
  10M vectors HNSW:  ~90-120 GB --> need multiple nodes or DiskANN
```

## Vendor Comparison

| Feature | Pinecone | Weaviate | Milvus | Qdrant | pgvector | Chroma |
|---|---|---|---|---|---|---|
| Deployment | Managed only | Self/Cloud | Self/Cloud | Self/Cloud | Extension | Embedded/Self |
| Primary index | Proprietary | HNSW | IVF/HNSW/DiskANN | HNSW | IVF/HNSW | HNSW |
| Max scale | Billions | ~100M | Billions | ~100M | ~5M | ~1M |
| Filtering | Metadata | Hybrid (inverted + vector) | Attribute + vector | Payload filters | SQL WHERE | Metadata |
| Multitenancy | Namespaces | Built-in | Partitions | Collections | Schemas | Collections |
| Disk-based | Yes | Partially | Yes (DiskANN) | Partially (mmap) | Yes (native) | No |
| Pricing model | Per vector | Open source / Cloud | Open source / Cloud | Open source / Cloud | Free (Postgres) | Open source |
| Best for | Zero-ops, rapid start | Hybrid search, GraphQL | Large-scale production | Performance-focused | Existing Postgres stack | Prototyping |

## Tuning Parameters Cheat Sheet

### HNSW Parameters

| Parameter | Default Range | Effect of Increasing |
|---|---|---|
| `M` (connections) | 8-64 (typical: 16) | Higher recall, more memory, slower build |
| `ef_construction` | 100-500 (typical: 200) | Better index quality, slower build |
| `ef_search` (ef) | 50-500 (typical: 100) | Higher recall, higher latency |

**Rule of thumb**: `ef_search >= k` (where k = top-k results requested)

### IVF Parameters

| Parameter | Typical Range | Guidance |
|---|---|---|
| `nlist` (clusters) | sqrt(n) to 4*sqrt(n) | More clusters = finer partitioning |
| `nprobe` | 1-nlist (typical: 5-20%) | More probes = higher recall, higher latency |

### PQ Parameters

| Parameter | Typical Range | Guidance |
|---|---|---|
| `m` (subquantizers) | 8-128 | More = better accuracy, more memory |
| `nbits` | 8 (standard) | 8 bits = 256 centroids per subspace |

## Hybrid Search Pattern

```
Query: "red running shoes under $50"

  Sparse retrieval (BM25/keyword)      Dense retrieval (vector)
           |                                    |
           v                                    v
      keyword matches                    semantic matches
           |                                    |
           +---------> Reciprocal Rank <--------+
                        Fusion (RRF)
                            |
                            v
                     Re-ranked results
                     (cross-encoder)
                            |
                            v
                       Top-K output
```

**RRF formula**: `score = sum(1 / (k + rank_i))` where k = 60 (typical constant)

## Common Interview Questions -- Concise Answers

**Q: How do you handle filtering with vector search?**
A: Pre-filtering (filter then search) is exact but slow for selective filters. Post-filtering
(search then filter) misses results. Best practice: integrated filtering where the index
supports metadata predicates during search (Weaviate, Qdrant, Milvus).

**Q: How do you update embeddings when the model changes?**
A: Dual-write during migration: keep old and new index in parallel. Backfill new index
asynchronously. Feature-flag switch reads. Drop old index. Budget 2-4 hours per 100M vectors.

**Q: HNSW vs. IVF -- when do you choose which?**
A: HNSW for latency-critical, moderate scale (< 50M in-memory). IVF-PQ for cost-sensitive,
very large scale (100M+) where slightly lower recall is acceptable.

**Q: How do you handle multi-tenancy?**
A: Namespace/partition per tenant for isolation. Shared index with metadata filter for
efficiency. Tradeoff: isolation vs. resource utilization. At FAANG scale, typically
partition-per-tenant with shared infrastructure.

**Q: What is the curse of dimensionality?**
A: In very high dimensions (>1000), all distances converge, making nearest-neighbor less
meaningful. Mitigation: dimensionality reduction (PCA, Matryoshka embeddings), domain-specific
fine-tuning, or reducing to 256-768 dims via projection.

## Key Numbers to Remember

| Metric | Value |
|---|---|
| OpenAI text-embedding-3-large dimensions | 3,072 (or 256-1536 via Matryoshka) |
| Cohere embed-v3 dimensions | 1,024 |
| float32 per dimension | 4 bytes |
| HNSW sweet spot (M) | 16-32 |
| HNSW search ef typical | 100-200 |
| IVF nprobe sweet spot | 5-10% of nlist |
| PQ compression ratio | 20-50x typical |
| Cosine similarity range | [-1, 1] (1 = identical) |
| ANN benchmark 95% recall latency (1M, HNSW) | < 1 ms |
| DiskANN 95% recall latency (1B, SSD) | < 5 ms |
| Reindexing throughput (HNSW) | ~5K-20K vectors/sec/core |
