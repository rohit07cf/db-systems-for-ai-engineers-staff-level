# RAG Storage Architecture at FAANG Scale

## Why This Matters at Staff/Principal Level

Retrieval-Augmented Generation (RAG) has become the dominant pattern for grounding LLM outputs
in factual, up-to-date information. At FAANG scale — with 1B+ documents, sub-200ms retrieval
SLOs, and thousands of QPS — the storage architecture becomes the critical bottleneck. A staff
engineer must design systems that balance retrieval quality, freshness, cost, and operational
complexity across multiple storage tiers.

---

## End-to-End RAG Storage Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        RAG Query Pipeline                          │
│                                                                     │
│  User Query ──► Query Encoder ──► Hybrid Search ──► Re-Ranker      │
│                      │               │    │            │            │
│                      ▼               ▼    ▼            ▼            │
│                 Query Cache     Vector  BM25       LLM Context      │
│                 (Redis/Memcached) Index  Index     Assembly         │
│                      │               │    │            │            │
│                      ▼               ▼    ▼            ▼            │
│                 Cache Hit?     Pinecone/ Elastic-  Top-K Merge      │
│                 Y: Return      Milvus/  search/    + Filter        │
│                 N: Continue    pgvector  Solr                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     Document Ingestion Pipeline                     │
│                                                                     │
│  Raw Docs ──► Chunker ──► Embedder ──► Index Writer ──► Metadata   │
│     │            │            │              │              │        │
│     ▼            ▼            ▼              ▼              ▼        │
│  S3/GCS      Chunk DB     Embedding      Vector +       PostgreSQL  │
│  (originals) (Postgres)   Queue (SQS)    BM25 Index    (metadata)   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │  Freshness Daemon: monitors source changes, triggers     │       │
│  │  re-ingestion, manages tombstones for deleted docs       │       │
│  └──────────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Document Chunking Strategies

Chunking is deceptively complex at scale. The wrong strategy degrades retrieval precision
and wastes embedding compute.

### Strategy Comparison

| Strategy | Chunk Size | Overlap | Pros | Cons | Best For |
|---|---|---|---|---|---|
| Fixed-size token | 256-512 tokens | 10-20% | Simple, predictable cost | Breaks semantic units | Homogeneous docs |
| Recursive splitter | Variable | Hierarchical | Respects structure | Tuning per doc type | Mixed content |
| Semantic chunking | Variable | None | High coherence | Expensive (requires model) | High-value corpora |
| Sentence-window | 1 sentence + context | Sliding window | Fine-grained retrieval | High chunk count | QA systems |
| Parent-child | Hierarchical | Implicit | Retrieve small, feed large | Complex indexing | Long documents |

### Parent-Child Chunking Architecture

```
┌─────────────────────────────────────┐
│         Parent Document             │
│  (stored in doc store, ~2000 tok)   │
│                                     │
│  ┌───────┐ ┌───────┐ ┌───────┐     │
│  │Child 1│ │Child 2│ │Child 3│     │
│  │256 tok│ │256 tok│ │256 tok│     │
│  │embedded│ │embedded│ │embedded│   │
│  └───┬───┘ └───┬───┘ └───┬───┘     │
│      │         │         │          │
│      ▼         ▼         ▼          │
│   Vector     Vector    Vector       │
│   Index      Index     Index        │
└─────────────────────────────────────┘

Retrieval: match child ──► fetch parent ──► feed parent to LLM
```

At 1B documents with ~10 chunks each, you manage 10B vectors — this is a capacity planning problem.

---

## Embedding Storage and Indexing

### Vector Index Selection at Scale

| Index Type | Build Time | Query Latency | Recall@10 | Memory | Mutable? |
|---|---|---|---|---|---|
| Flat (brute force) | O(1) | O(n) | 100% | Full | Yes |
| IVF-PQ | Hours (1B) | 1-5ms | 92-97% | ~10x reduction | Partial |
| HNSW | Hours (1B) | <1ms | 97-99% | 1.5-2x overhead | Difficult |
| ScaNN (Google) | Hours | <1ms | 97-99% | Moderate | Partial |
| DiskANN (Microsoft) | Hours | 2-5ms | 95-99% | Minimal (SSD) | No |

### Storage Layout for 1B+ Vectors

```
┌────────────────────────────────────────────────────┐
│              Tiered Vector Storage                  │
│                                                     │
│  Hot Tier (DRAM):     100M vectors (HNSW)          │
│  ├── Most queried documents                         │
│  ├── Recently ingested                              │
│  └── ~150GB at 1536-dim float32                     │
│                                                     │
│  Warm Tier (SSD):     900M vectors (DiskANN/IVF)   │
│  ├── Bulk of the corpus                             │
│  ├── Quantized to int8 or PQ codes                  │
│  └── ~200GB with PQ64                               │
│                                                     │
│  Cold Tier (S3):      Archive / deprecated versions │
│  ├── Previous embedding model generations           │
│  └── Compressed, not indexed                        │
└────────────────────────────────────────────────────┘
```

---

## Metadata Schema Design

```sql
-- Core chunk metadata (PostgreSQL)
CREATE TABLE rag_chunks (
    chunk_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES documents(document_id),
    chunk_index     INT NOT NULL,
    content_hash    BYTEA NOT NULL,          -- SHA-256 for dedup
    token_count     INT NOT NULL,
    embedding_model VARCHAR(64) NOT NULL,     -- e.g., 'text-embedding-3-large-v2'
    embedding_dim   INT NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT now(),
    source_uri      TEXT,
    content_type    VARCHAR(32),              -- 'text', 'table', 'code', 'image_caption'
    language        VARCHAR(8),
    acl_tags        TEXT[],                   -- access control for multi-tenant RAG
    freshness_ts    TIMESTAMPTZ,              -- when source was last verified current
    UNIQUE(document_id, chunk_index, embedding_model)
);

-- Partition by embedding_model for efficient model migrations
CREATE TABLE rag_chunks_v2 PARTITION OF rag_chunks
    FOR VALUES IN ('text-embedding-3-large-v2');

-- GIN index for ACL-based filtering
CREATE INDEX idx_acl ON rag_chunks USING GIN(acl_tags);

-- BRIN index for time-range freshness queries
CREATE INDEX idx_freshness ON rag_chunks USING BRIN(freshness_ts);
```

---

## Hybrid Search: BM25 + Vector

Neither BM25 nor vector search alone is sufficient. Staff-level designs use hybrid retrieval.

### Fusion Strategies

```
┌──────────────────────────────────────────────────┐
│            Hybrid Retrieval Pipeline              │
│                                                   │
│  Query ──┬──► BM25 (Elasticsearch)               │
│          │     top-100 results ──────┐            │
│          │                           ▼            │
│          └──► ANN (Milvus/Pinecone)  Reciprocal   │
│                top-100 results ──► Rank Fusion    │
│                                      │            │
│                                      ▼            │
│                                   Top-20 ──►      │
│                                   Cross-Encoder   │
│                                   Re-ranker       │
│                                      │            │
│                                      ▼            │
│                                   Top-5 to LLM    │
└──────────────────────────────────────────────────┘
```

### Reciprocal Rank Fusion (RRF) Formula

```
RRF_score(d) = Σ  1 / (k + rank_i(d))
               i∈{bm25, vector}

where k = 60 (standard constant)
```

| Approach | Precision@5 | Latency (p99) | Complexity |
|---|---|---|---|
| BM25 only | 0.62 | 15ms | Low |
| Vector only | 0.71 | 25ms | Medium |
| Hybrid (RRF) | 0.79 | 40ms | Medium |
| Hybrid + Re-ranker | 0.87 | 120ms | High |

---

## Cache Layers for Repeated Queries

```
┌─────────────────────────────────────────────────┐
│              Multi-Level Cache                   │
│                                                  │
│  L1: Exact Query Cache (Redis)                   │
│  ├── Key: hash(query + filters)                  │
│  ├── Value: serialized top-K results             │
│  ├── TTL: 5-15 minutes                           │
│  └── Hit rate: 15-30% (high query repetition)    │
│                                                  │
│  L2: Semantic Query Cache                        │
│  ├── Key: embedding of query                     │
│  ├── Lookup: ANN search in cache index           │
│  ├── Threshold: cosine_sim > 0.95                │
│  ├── Value: cached retrieval results             │
│  └── Hit rate: additional 10-20%                 │
│                                                  │
│  L3: Document Cache (Memcached)                  │
│  ├── Cache individual chunk content              │
│  ├── Avoids repeated doc store reads             │
│  └── TTL: 1 hour                                 │
└─────────────────────────────────────────────────┘
```

---

## Freshness Management

At FAANG scale, corpora change continuously. Stale retrieval is a correctness bug.

### Freshness Architecture

```
Source Systems ──► Change Data Capture ──► Freshness Queue (Kafka)
                                                │
                    ┌───────────────────────────┤
                    ▼                           ▼
            Re-chunk + Re-embed          Tombstone old chunks
            (async pipeline)             (mark deleted in metadata)
                    │                           │
                    ▼                           ▼
            Upsert to vector index       Lazy deletion from
            + BM25 index                 vector index (compaction)
```

**Freshness SLO tiers:**
- Tier 1 (critical): <5 min staleness (e.g., policy docs, pricing)
- Tier 2 (standard): <1 hour staleness (e.g., knowledge base articles)
- Tier 3 (relaxed): <24 hour staleness (e.g., archived research)

---

## Multi-Modal RAG Storage

```
┌─────────────────────────────────────────────────────────┐
│            Multi-Modal Document Pipeline                 │
│                                                          │
│  PDF ──► Text Extractor ──► Text chunks ──► text embed   │
│   │                                                      │
│   ├──► Table Extractor ──► Structured JSON ──► text embed│
│   │                                                      │
│   ├──► Image Extractor ──► CLIP embed + caption ──► both │
│   │                                                      │
│   └──► Chart Extractor ──► Data table ──► text embed     │
│                                                          │
│  Storage: each modality stored with type tag             │
│  Retrieval: modality-aware fusion at re-ranking          │
└─────────────────────────────────────────────────────────┘
```

---

## Cost Modeling: RAG at 1B+ Documents

| Component | Specification | Monthly Cost Estimate |
|---|---|---|
| Vector storage (hot) | 100M x 1536-dim, HNSW, DRAM | $8,000-12,000 |
| Vector storage (warm) | 900M x PQ64, SSD | $3,000-5,000 |
| BM25 index (Elasticsearch) | 1B docs, 3 replicas | $15,000-25,000 |
| Embedding compute | Re-embed 10M docs/day | $2,000-4,000 |
| Metadata DB (PostgreSQL) | 10B rows, read replicas | $5,000-8,000 |
| Cache layer (Redis cluster) | 256GB, HA | $4,000-6,000 |
| Re-ranker inference | Cross-encoder, 1000 QPS | $10,000-20,000 |
| Object storage (S3) | Raw docs, 50TB | $1,200 |
| **Total** | | **$48,200-81,200/mo** |

**Key cost levers:**
- Quantization (float32 to int8): 4x memory reduction, ~2% recall loss
- Aggressive caching: 30-40% compute reduction
- Tiered storage: 60% cost reduction vs. all-DRAM
- Re-ranker batching: 3x throughput improvement

---

## Interview Narrative

**When asked about RAG system design in a Staff/Principal interview:**

> "I'd start by clarifying the scale and freshness requirements — these drive almost every
> architectural decision. At 1B+ documents, you can't keep everything in DRAM, so I'd design
> a tiered storage architecture: hot tier with HNSW for the most-queried 10% of documents,
> warm tier with DiskANN or IVF-PQ for the bulk, and cold storage on S3 for deprecated
> embedding versions.
>
> For retrieval quality, I'd implement hybrid search — BM25 handles exact keyword matches
> that vector search misses, and reciprocal rank fusion merges the results before a
> cross-encoder re-ranker selects the final top-K. This consistently beats either approach
> alone by 10-15% in precision.
>
> Chunking strategy depends on the corpus. For heterogeneous documents, I'd use parent-child
> chunking — embed small chunks for precise retrieval but feed the parent context to the LLM.
> This avoids the common failure mode of retrieving a relevant sentence but losing the
> surrounding context the LLM needs.
>
> For freshness, I'd build a CDC pipeline from source systems into a re-embedding queue,
> with tiered freshness SLOs. Critical documents get sub-5-minute staleness via priority
> processing; the long tail gets daily re-indexing.
>
> On cost — the re-ranker and vector storage are the biggest line items. I'd invest in
> quantization (4x memory savings for ~2% recall loss), semantic query caching (15-20%
> additional hit rate beyond exact-match caching), and batched re-ranking inference. At
> FAANG scale, these optimizations save $200-400K annually."

**Key signals this demonstrates:**
- Systems thinking across ingestion, storage, and retrieval
- Cost-awareness and quantitative reasoning
- Understanding of quality/cost/latency tradeoffs
- Practical experience with failure modes (stale data, context loss, cold-start)
- Ability to articulate tiered architecture decisions
