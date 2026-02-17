# Vector Database vs pgvector: Deep Tradeoff Analysis

## Staff/Principal Level — Database Systems for AI Engineers

---

## 1. The Central Question

Every AI engineering team eventually faces this decision: use a dedicated vector database
(Pinecone, Weaviate, Qdrant, Milvus) or add the pgvector extension to their existing
PostgreSQL? The answer depends on scale, operational maturity, query patterns, and
whether vector search is a core differentiator or a supporting feature.

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  The Two Philosophies:                                      │
  │                                                             │
  │  "Use the database you already have"                        │
  │  → pgvector: Less infrastructure, simpler operations,       │
  │    ACID guarantees, joins with relational data              │
  │                                                             │
  │  "Use the right tool for the job"                           │
  │  → Dedicated vector DB: Purpose-built for vector ops,       │
  │    better performance at scale, richer vector features      │
  │                                                             │
  │  Reality: Both are correct — at different scales and        │
  │  for different organizational contexts.                     │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

---

## 2. Architecture Comparison

### 2.1 pgvector Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                    PostgreSQL + pgvector                     │
  │                                                             │
  │  ┌───────────────────────────────────────────────────────┐  │
  │  │                PostgreSQL Process                     │  │
  │  │                                                       │  │
  │  │  ┌─────────┐  ┌─────────┐  ┌───────────────────┐    │  │
  │  │  │ Planner │  │Executor │  │ Buffer Manager    │    │  │
  │  │  └────┬────┘  └────┬────┘  │ (shared_buffers)  │    │  │
  │  │       │            │       └────────┬──────────┘    │  │
  │  │       ▼            ▼                │               │  │
  │  │  ┌──────────────────────────────────┴──────────┐    │  │
  │  │  │              pgvector Extension              │    │  │
  │  │  │                                              │    │  │
  │  │  │  Index types:                                │    │  │
  │  │  │  - ivfflat (IVF with flat quantization)     │    │  │
  │  │  │  - hnsw (since pgvector 0.5.0)              │    │  │
  │  │  │                                              │    │  │
  │  │  │  Operators: <-> (L2), <=> (cosine),         │    │  │
  │  │  │             <#> (inner product)              │    │  │
  │  │  │                                              │    │  │
  │  │  │  Data type: vector(dim) — stored as float32 │    │  │
  │  │  └──────────────────────────────────────────────┘    │  │
  │  │                                                       │  │
  │  │  ACID transactions ✓  WAL replication ✓              │  │
  │  │  Joins ✓  Constraints ✓  Extensions ✓                │  │
  │  └───────────────────────────────────────────────────────┘  │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### 2.2 Dedicated Vector DB Architecture (e.g., Qdrant)

```
  ┌─────────────────────────────────────────────────────────────┐
  │              Dedicated Vector Database (Qdrant)              │
  │                                                             │
  │  ┌───────────────────────────────────────────────────────┐  │
  │  │                  gRPC / REST API                      │  │
  │  └────────────────────────┬──────────────────────────────┘  │
  │                           │                                  │
  │  ┌────────────────────────▼──────────────────────────────┐  │
  │  │              Query Engine                              │  │
  │  │  - Vector search (HNSW)                               │  │
  │  │  - Metadata filtering (integrated with HNSW traversal)│  │
  │  │  - Multi-vector search, grouping, recommendation      │  │
  │  └────────────────────────┬──────────────────────────────┘  │
  │                           │                                  │
  │  ┌────────────────────────▼──────────────────────────────┐  │
  │  │              Storage Engine                            │  │
  │  │  - Segment-based (growing + sealed)                   │  │
  │  │  - WAL for durability                                 │  │
  │  │  - Quantization: scalar, product, binary              │  │
  │  │  - Memory-mapped files for disk index                 │  │
  │  └────────────────────────┬──────────────────────────────┘  │
  │                           │                                  │
  │  ┌────────────────────────▼──────────────────────────────┐  │
  │  │              Distributed Layer                          │  │
  │  │  - Sharding (automatic)                               │  │
  │  │  - Replication (Raft consensus)                       │  │
  │  │  - Cluster coordination                               │  │
  │  └───────────────────────────────────────────────────────┘  │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

---

## 3. Performance Benchmarks

### 3.1 Query Performance Comparison

```
  Single-node, d=768, k=10, HNSW index, recall@10 ≈ 0.95

  ┌────────────────────────────────────────────────────────────┐
  │               Query Latency (p50, ms)                      │
  │                                                            │
  │  Vector Count    pgvector (HNSW)    Qdrant     Weaviate   │
  │  ──────────────  ────────────────   ──────     ────────   │
  │  100K            1.5                0.5        0.8        │
  │  1M              4.0                1.2        2.0        │
  │  5M              12.0               2.5        4.0        │
  │  10M             25.0               4.0        6.5        │
  │  50M             OOM*               8.0        15.0       │
  │                                                            │
  │  * pgvector HNSW on 50M x 768d requires ~200GB RAM        │
  │    PostgreSQL shared_buffers + OS cache must hold index    │
  │                                                            │
  └────────────────────────────────────────────────────────────┘

  Notes:
  - pgvector latency includes PostgreSQL query planning overhead
  - Dedicated DBs have zero planning overhead (purpose-built API)
  - pgvector HNSW builds are MUCH slower (single-threaded until 0.8.0)
  - Dedicated DBs use parallel index construction
```

### 3.2 Throughput Comparison

```
  Queries Per Second (QPS), single node, 1M vectors, d=768:

  pgvector:     ████████████████████  ~500 QPS
  Qdrant:       ████████████████████████████████████████████████  ~5,000 QPS
  Weaviate:     ████████████████████████████████████████  ~3,000 QPS
  Milvus:       ████████████████████████████████████████████  ~4,000 QPS

  Why pgvector is slower:
  1. PostgreSQL connection model (process per connection)
  2. Query planner overhead per query (~0.5ms)
  3. MVCC overhead (tuple visibility checks)
  4. Shared buffer management not optimized for vector access patterns
  5. No SIMD-optimized distance functions until recent versions
```

### 3.3 Index Build Time

```
  Build time for 1M vectors, d=768, HNSW (M=16, efConstruction=200):

  pgvector 0.7.x:    ████████████████████████████████████████  ~45 min
                      (single-threaded build)

  pgvector 0.8.0+:   ████████████████████  ~15 min
                      (parallel build added)

  Qdrant:             ████████  ~5 min
                      (parallel, optimized from scratch)

  FAISS (in-memory):  ████  ~2 min
                      (pure C++, SIMD optimized)
```

---

## 4. Feature Comparison

### 4.1 Comprehensive Feature Matrix

| Feature | pgvector | Pinecone | Weaviate | Qdrant | Milvus |
|---|---|---|---|---|---|
| **HNSW index** | Yes (0.5+) | Yes (proprietary) | Yes | Yes | Yes |
| **IVF index** | Yes (ivfflat) | N/A | N/A | N/A | Yes |
| **DiskANN** | No | No | No | No | Yes |
| **Product Quantization** | No | Yes | Yes | Yes | Yes |
| **Scalar Quantization** | halfvec (0.7+) | Yes | N/A | Yes | Yes |
| **Binary Quantization** | bit type (0.7+) | N/A | Yes | Yes | Yes |
| **Filtered search** | WHERE clause | Metadata filter | Filter | Payload filter | Expr filter |
| **Hybrid (sparse+dense)** | BM25 + vector* | Sparse vectors | BM25 built-in | Sparse vectors | Sparse vectors |
| **Multi-vector** | Multiple columns | Namespaces | Cross-references | Named vectors | Multiple fields |
| **ACID transactions** | Yes | No | No | No | No |
| **Joins** | Yes (full SQL) | No | GraphQL refs | No | No |
| **Replication** | Streaming repl | Managed | Raft | Raft | Custom |
| **Sharding** | Citus/partitioning | Managed | Auto | Auto | Auto |
| **Max tested scale** | ~10M per node | Billions | ~100M | ~100M | Billions |

*pgvector hybrid search requires pg_trgm or tsvector + RRF in application code.*

### 4.2 What pgvector Cannot Do (Well)

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  1. IN-FILTER SEARCH                                        │
  │     pgvector applies WHERE after HNSW traversal             │
  │     → Post-filtering → recall degrades with selective       │
  │       filters. No integrated filtering during graph walk.   │
  │                                                             │
  │  2. PRODUCT QUANTIZATION                                    │
  │     No PQ support. Vectors stored as full float32.          │
  │     Memory usage: N * d * 4 bytes minimum.                  │
  │     No 10-30x compression available.                        │
  │                                                             │
  │  3. DISTRIBUTED VECTOR SEARCH                               │
  │     No native scatter-gather across shards.                 │
  │     Citus can partition, but each shard is independent.     │
  │     Application must merge results from multiple shards.    │
  │                                                             │
  │  4. STREAMING INDEX UPDATES                                 │
  │     HNSW inserts are slow under concurrent load.            │
  │     Index maintenance blocks vacuum and vice versa.         │
  │     No segment-based architecture for isolating writes.     │
  │                                                             │
  │  5. PURPOSE-BUILT OBSERVABILITY                             │
  │     No built-in recall monitoring.                          │
  │     No vector-specific dashboards or metrics.               │
  │     Must build custom monitoring on pg_stat.                │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### 4.3 What pgvector Does Better

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  1. ACID TRANSACTIONS                                       │
  │     INSERT INTO items (id, data, embedding)                 │
  │     VALUES (...) — atomic, consistent, durable.             │
  │     Vector + metadata always consistent.                    │
  │     Dedicated vector DBs: eventual consistency risk.        │
  │                                                             │
  │  2. SQL JOINS                                               │
  │     SELECT i.name, i.price                                  │
  │     FROM items i                                            │
  │     JOIN categories c ON i.category_id = c.id               │
  │     ORDER BY i.embedding <=> query_vec                      │
  │     LIMIT 10;                                               │
  │     One query. One round trip. Impossible in vector DBs.    │
  │                                                             │
  │  3. EXISTING OPERATIONAL KNOWLEDGE                          │
  │     Your team already knows PostgreSQL.                     │
  │     Backup: pg_dump. Replication: streaming.                │
  │     Monitoring: pg_stat_statements. Tuning: explain.        │
  │     No new system to learn, deploy, monitor, secure.        │
  │                                                             │
  │  4. SCHEMA ENFORCEMENT                                      │
  │     vector(768) — dimension enforced at type level.         │
  │     No accidental insertion of wrong-dimension vectors.     │
  │     Dedicated DBs may accept any dimension silently.        │
  │                                                             │
  │  5. COST AT SMALL SCALE                                     │
  │     No additional infrastructure. No separate service.      │
  │     Just `CREATE EXTENSION vector;` on existing Postgres.   │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

---

## 5. Scale Limits

### 5.1 pgvector Scale Boundaries

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  PRACTICAL LIMITS OF pgvector (single node):                │
  │                                                             │
  │  Vectors    Memory      Latency (p50)    Verdict            │
  │  ────────   ────────    ─────────────    ────────           │
  │  < 100K     < 1 GB      < 2ms            EASY               │
  │  100K-1M    1-4 GB      2-5ms            COMFORTABLE        │
  │  1M-5M      4-20 GB     5-15ms           MANAGEABLE         │
  │  5M-10M     20-40 GB    15-30ms          CHALLENGING        │
  │  10M-50M    40-200 GB   30-100ms         PUSHING LIMITS     │
  │  > 50M      > 200 GB    > 100ms          USE DEDICATED DB   │
  │                                                             │
  │  Key constraint: PostgreSQL's shared_buffers + OS page      │
  │  cache must hold the HNSW index. If index exceeds RAM,      │
  │  disk seeks during graph traversal → 100x latency spike.    │
  │                                                             │
  │  HNSW index size ≈ N * (d * 4 + M * 2 * 8 + 64) bytes     │
  │  For N=10M, d=768, M=16:                                    │
  │  = 10M * (3072 + 256 + 64) = ~33.9 GB index                │
  │  + 10M * 3072 = ~30.7 GB heap (tuple storage)              │
  │  = ~64.6 GB total                                           │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### 5.2 Scaling Strategies

```
  pgvector Scaling Path:
  ──────────────────────

  Phase 1: Vertical (< 10M vectors)
    - Bigger instance (more RAM)
    - Tune maintenance_work_mem for faster index builds
    - Tune shared_buffers (25% of RAM)
    - Use effective_cache_size for planner hints

  Phase 2: Read replicas (< 10M, high QPS)
    - Streaming replication to read replicas
    - Route vector queries to replicas
    - Primary handles writes

  Phase 3: Partitioning (10M-100M)
    - Partition by tenant_id or date range
    - Each partition has its own HNSW index
    - Application-level scatter-gather
    - OR use Citus for distributed queries

  Phase 4: Graduate to dedicated vector DB (> 100M)
    - pgvector's architecture can't compete
    - Need PQ, DiskANN, distributed scatter-gather
    - Migration: dual-write, validate, switch


  Dedicated Vector DB Scaling Path:
  ──────────────────────────────────

  Phase 1: Single node (< 50M)
  Phase 2: Replication (high QPS)
  Phase 3: Sharding (50M-1B)
  Phase 4: Tiered storage (> 1B, cost optimization)

  Note: Dedicated DBs handle Phase 2-4 natively.
  pgvector requires significant application-level work.
```

---

## 6. Operational Complexity

### 6.1 Operations Comparison

| Operation | pgvector | Dedicated Vector DB |
|---|---|---|
| **Installation** | `CREATE EXTENSION vector;` | Deploy new service + infra |
| **Backup** | pg_dump / pg_basebackup | Custom backup, vendor-specific |
| **Monitoring** | pg_stat + existing PG tooling | New dashboards, new alerts |
| **Upgrades** | Standard PG upgrade path | Vendor-specific, may need reindex |
| **Security** | PostgreSQL RBAC, SSL, pgaudit | New auth system, new certs |
| **Team expertise** | Existing DBA knowledge | New specialty required |
| **Incident response** | Known PG runbooks | New runbooks, less experience |
| **Index tuning** | EXPLAIN ANALYZE + standard PG | Vendor-specific parameters |
| **HA/DR** | PG streaming replication | Vendor-specific clustering |

### 6.2 Total Cost of Ownership

```
  ┌─────────────────────────────────────────────────────────────┐
  │          TCO Comparison (1M vectors, 100 QPS, 1 year)       │
  │                                                             │
  │  pgvector on existing PostgreSQL:                           │
  │  ┌─────────────────────────────────────────────────┐       │
  │  │ Incremental compute cost:  ~$0/mo (already running)│    │
  │  │ RAM upgrade (if needed):   ~$100/mo                │    │
  │  │ Engineering time:          ~$0/mo (known tech)     │    │
  │  │ ─────────────────────────────────────────────────  │    │
  │  │ Total:                     ~$100/mo = $1,200/yr    │    │
  │  └─────────────────────────────────────────────────────┘    │
  │                                                             │
  │  Managed vector DB (Pinecone):                              │
  │  ┌─────────────────────────────────────────────────┐       │
  │  │ Service cost:              ~$70/mo (s1 pod)        │    │
  │  │ Engineering time (new):    ~$2,000 (one-time)      │    │
  │  │ Ongoing learning curve:    ~$500/yr                │    │
  │  │ ─────────────────────────────────────────────────  │    │
  │  │ Total:                     ~$70/mo = $840/yr + $2.5K│   │
  │  └─────────────────────────────────────────────────────┘    │
  │                                                             │
  │  Self-hosted vector DB (Qdrant):                            │
  │  ┌─────────────────────────────────────────────────┐       │
  │  │ Compute:                   ~$200/mo (dedicated node)│   │
  │  │ Engineering (setup):       ~$5,000 (one-time)      │    │
  │  │ Engineering (ops):         ~$2,000/yr              │    │
  │  │ ─────────────────────────────────────────────────  │    │
  │  │ Total:                     ~$200/mo = $2,400/yr+$7K│   │
  │  └─────────────────────────────────────────────────────┘    │
  │                                                             │
  │  At 1M vectors: pgvector wins on cost.                      │
  │  At 100M vectors: dedicated DB wins on performance/dollar.  │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

---

## 7. Hybrid Search Capabilities

### 7.1 Hybrid Search Comparison

```
  pgvector Hybrid Search (manual RRF):
  ─────────────────────────────────────

  -- Step 1: Full-text search
  WITH text_results AS (
    SELECT id, ts_rank(tsv, to_tsquery('quantum physics')) as text_score
    FROM documents
    WHERE tsv @@ to_tsquery('quantum physics')
    ORDER BY text_score DESC LIMIT 100
  ),
  -- Step 2: Vector search
  vector_results AS (
    SELECT id, 1 - (embedding <=> $query_vec) as vec_score
    FROM documents
    ORDER BY embedding <=> $query_vec LIMIT 100
  ),
  -- Step 3: Reciprocal Rank Fusion
  combined AS (
    SELECT COALESCE(t.id, v.id) as id,
           COALESCE(1.0/(60 + t.rank), 0) + COALESCE(1.0/(60 + v.rank), 0) as rrf
    FROM (SELECT id, ROW_NUMBER() OVER (ORDER BY text_score DESC) as rank
          FROM text_results) t
    FULL OUTER JOIN
         (SELECT id, ROW_NUMBER() OVER (ORDER BY vec_score DESC) as rank
          FROM vector_results) v ON t.id = v.id
  )
  SELECT * FROM combined ORDER BY rrf DESC LIMIT 10;

  Pros: Full SQL power, transaction-consistent, custom fusion logic
  Cons: Two index scans, manual RRF, complex query, no built-in relevance tuning


  Weaviate Hybrid Search (built-in):
  ───────────────────────────────────

  client.query.get("Document", ["title", "content"])
    .with_hybrid(query="quantum physics", alpha=0.5)
    .with_limit(10)

  Pros: One API call, built-in BM25+vector fusion, tunable alpha
  Cons: Less flexible fusion, can't join with relational data
```

---

## 8. Decision Framework

### 8.1 The Decision Matrix

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  USE pgvector WHEN:                                         │
  │                                                             │
  │  ✓ < 5M vectors (comfortably) or < 10M (with tuning)      │
  │  ✓ You already run PostgreSQL                               │
  │  ✓ You need ACID transactions across vector + relational   │
  │  ✓ You need SQL joins between vectors and business data    │
  │  ✓ Your team has PostgreSQL expertise but not vector DB     │
  │  ✓ Vector search is a feature, not the product             │
  │  ✓ You're building an MVP or prototype                     │
  │  ✓ Operational simplicity is the top priority               │
  │  ✓ QPS requirements < 500                                   │
  │                                                             │
  │  USE DEDICATED VECTOR DB WHEN:                              │
  │                                                             │
  │  ✓ > 10M vectors (or growing toward that)                  │
  │  ✓ QPS > 1000 for vector queries                           │
  │  ✓ Latency SLA < 5ms at p99                                │
  │  ✓ You need PQ/SQ compression (memory-constrained)         │
  │  ✓ You need distributed vector search (sharding)           │
  │  ✓ Vector search IS the product                            │
  │  ✓ You need advanced features (multi-vector, sparse, etc.) │
  │  ✓ You have budget for new infrastructure + learning curve │
  │  ✓ Filtered search with low-selectivity filters is common  │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### 8.2 Decision Flowchart

```
  ┌─ How many vectors?
  │
  ├─ < 1M ──→ pgvector (no debate)
  │
  ├─ 1M-10M
  │   ├─ Need ACID + joins? ──→ pgvector
  │   ├─ Need < 5ms p99?    ──→ Dedicated DB
  │   ├─ Need > 500 QPS?    ──→ Dedicated DB
  │   └─ Otherwise           ──→ pgvector (simpler)
  │
  ├─ 10M-100M
  │   ├─ Need ACID + joins?  ──→ pgvector + Citus (complex)
  │   └─ Otherwise            ──→ Dedicated DB (recommended)
  │
  └─ > 100M ──→ Dedicated DB (only option)
```

### 8.3 The Migration Path

```
  Recommended approach: Start with pgvector, migrate when needed.

  Phase 1: pgvector (Month 1-6)
  ┌─────────────────────────────────────────────────────┐
  │ - Quick to set up, familiar technology               │
  │ - Learn vector search patterns with real data        │
  │ - Establish recall baselines and latency benchmarks  │
  │ - Determine actual scale requirements                │
  └─────────────────────────────────────────────────────┘
           │
           │ When: latency > SLA, or scale > 10M, or QPS > 500
           ▼
  Phase 2: Dual-write + Shadow (Month 6-7)
  ┌─────────────────────────────────────────────────────┐
  │ - Deploy dedicated vector DB alongside PostgreSQL    │
  │ - Dual-write: all vectors go to both systems         │
  │ - Shadow queries: query both, compare results        │
  │ - Validate recall, latency, correctness              │
  └─────────────────────────────────────────────────────┘
           │
           ▼
  Phase 3: Cutover (Month 7-8)
  ┌─────────────────────────────────────────────────────┐
  │ - Route vector queries to dedicated DB               │
  │ - Keep relational data in PostgreSQL                 │
  │ - Application joins results from both systems        │
  │ - Decommission pgvector index (keep data in PG)      │
  └─────────────────────────────────────────────────────┘
```

---

## 9. Ecosystem Maturity

### 9.1 Maturity Assessment

| Dimension | pgvector | Pinecone | Weaviate | Qdrant | Milvus |
|---|---|---|---|---|---|
| **Age** | 2021 (ext), PG since 1996 | 2019 | 2019 | 2021 | 2019 |
| **Production deployments** | Thousands | Thousands | Hundreds | Hundreds | Hundreds |
| **Community size** | Huge (PostgreSQL) | Large (SaaS) | Medium | Medium | Large |
| **Enterprise support** | Many PG vendors | Yes (managed) | Yes | Yes | Zilliz (managed) |
| **Cloud-native** | RDS, Cloud SQL, etc. | SaaS-only | Cloud + self-host | Cloud + self-host | Zilliz Cloud |
| **Documentation quality** | Good (PG standard) | Excellent | Good | Excellent | Good |
| **Breaking changes risk** | Very Low (PG stability) | Low (managed) | Medium | Medium | Medium |
| **Tooling ecosystem** | pg_stat, pgBouncer, etc. | Built-in dashboard | Built-in | Built-in | Built-in |

### 9.2 Risk Assessment

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  pgvector risks:                                            │
  │  - Extension maintenance lag (community-driven)             │
  │  - Performance ceiling at scale                             │
  │  - Limited vector-specific optimizations                    │
  │  - No PQ = higher memory costs                              │
  │  Hedge: PostgreSQL isn't going anywhere.                    │
  │                                                             │
  │  Dedicated vector DB risks:                                 │
  │  - Vendor viability (startups, funding-dependent)           │
  │  - API stability (breaking changes in young software)       │
  │  - Operational knowledge gap                                │
  │  - Migration cost if vendor pivots or shuts down            │
  │  Hedge: Open-source options (Qdrant, Milvus) reduce lock-in│
  │                                                             │
  │  Managed vector DB risks (Pinecone):                        │
  │  - Vendor lock-in (proprietary, no self-host)               │
  │  - Cost scaling (opaque pricing at high volume)             │
  │  - Region availability constraints                          │
  │  - Data residency compliance                                │
  │  Hedge: Abstract vector operations behind interface.        │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

---

## 10. Performance Tuning Guide

### 10.1 pgvector Tuning

```sql
  -- Essential pgvector HNSW tuning
  -- Index creation:
  CREATE INDEX ON items
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 200);

  -- Query-time tuning:
  SET hnsw.ef_search = 100;           -- Higher = better recall, slower
  SET work_mem = '256MB';             -- Memory for sorting
  SET maintenance_work_mem = '2GB';   -- Memory for index build

  -- PostgreSQL-level tuning for vector workloads:
  SET shared_buffers = '8GB';           -- 25% of RAM
  SET effective_cache_size = '24GB';    -- 75% of RAM
  SET random_page_cost = 1.1;          -- SSD storage
  SET max_parallel_workers_per_gather = 4;  -- Parallel scans

  -- CRITICAL: Prevent planner from choosing seq scan
  SET enable_seqscan = off;  -- For vector queries only (session-level)

  -- Monitor index usage:
  SELECT * FROM pg_stat_user_indexes
  WHERE relname = 'items' AND indexrelname LIKE '%hnsw%';
```

### 10.2 Dedicated Vector DB Tuning

```yaml
  # Qdrant example configuration
  storage:
    performance:
      max_search_threads: 0  # auto (= num_cpus)
    optimizers:
      default_segment_number: 4
      max_segment_size_kb: 200000  # 200MB per segment
      indexing_threshold_kb: 20000  # Build HNSW after 20K vectors
    hnsw_index:
      m: 16
      ef_construct: 200
      full_scan_threshold: 10000  # Brute force below this
    quantization:
      scalar:
        type: int8
        always_ram: true  # Keep quantized in RAM, full on disk
```

---

## 11. Interview Narrative

### How to Present This in a Staff/Principal Interview

**Opening framing (30 seconds):**
> "The pgvector vs dedicated vector DB question is fundamentally about organizational
> efficiency, not technology superiority. pgvector wins when vector search is a feature
> within a relational application and scale is under 10M vectors. A dedicated vector DB
> wins when vector search is the core workload and you need performance at scale. My
> default recommendation is to start with pgvector and migrate only when you have
> concrete evidence that you've outgrown it."

**When asked "Why not just use pgvector for everything?":**
> "pgvector inherits PostgreSQL's process-per-connection model, MVCC overhead, and
> general-purpose query planner — all of which add latency for pure vector operations.
> At 1M vectors, that's maybe 4ms vs 1ms — not a big deal. At 10M vectors with 1000
> QPS and a 5ms SLA, pgvector simply can't keep up. It also lacks product quantization,
> so your memory cost is 4-30x higher than a dedicated DB with PQ. The key question is:
> at your scale, does that performance and cost difference justify the operational
> complexity of a new system?"

**Demonstrating E7 depth:**
- Quantify the TCO crossover point where dedicated DB becomes cheaper (typically ~10M vectors)
- Explain how pgvector's post-filtering hurts recall for selective metadata queries
- Discuss the migration path: pgvector -> dual-write -> shadow -> cutover
- Articulate the ACID advantage: vector + metadata consistency without 2PC
- Propose the abstraction layer that makes the underlying vector store swappable

**System design follow-up — "Design a RAG system for a startup":**
> "For a startup, I'd start with pgvector on their existing Postgres instance. Store
> document chunks as rows with a vector column. Use HNSW index. Total setup: one
> afternoon. When they hit 5M chunks and need < 5ms latency, migrate to Qdrant behind
> an abstraction layer. Keep the document metadata and access control in Postgres. The
> application joins vector results (IDs) with relational data — one extra round trip,
> but full ACID on the relational side and fast vector search on the dedicated side."

**Red flags to avoid:**
- Don't dismiss pgvector as "just a toy" — it handles millions of vectors in production.
- Don't recommend a dedicated vector DB for a < 1M vector prototype.
- Don't ignore the operational cost of a new system (monitoring, alerting, on-call).
- Don't forget that pgvector gives you transactions — consistency bugs in eventual
  consistency systems are expensive to debug.
