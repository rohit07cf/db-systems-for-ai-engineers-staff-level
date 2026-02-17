# Billion-Scale Vector Architecture

## Staff/Principal Level — Database Systems for AI Engineers

---

## 1. The Billion-Vector Challenge

At 1 billion vectors with d=768 (float32), the raw data alone is **3 TB**. Add graph
structures (HNSW) and you need **3.4 TB of RAM** — far exceeding any single machine. This
forces architectural decisions around sharding, tiered storage, quantization, and
distributed query execution.

```
  Memory Estimation Formula:
  ─────────────────────────
  Raw vectors:     N * d * sizeof(float32) = 1B * 768 * 4 = 3,072 GB
  HNSW graph:      N * M_avg * sizeof(int64) ≈ 1B * 32 * 8 = 256 GB
  Metadata/IDs:    N * ~100 bytes = 100 GB
  ─────────────────────────────────────────────────
  Total (HNSW):    ~3,428 GB  ← impossible on one node

  With IVF-PQ (m=96):
  PQ codes:        N * m = 1B * 96 = 96 GB
  Centroids:       nlist * d * 4 ≈ negligible
  ─────────────────────────────────────────────────
  Total (IVF-PQ):  ~196 GB   ← fits on a large node
```

---

## 2. DiskANN: SSD-Optimized Vector Search

### 2.1 Architecture

```
  ┌─────────────────────────────────────────────────────────┐
  │                    DiskANN (Vamana)                      │
  │                                                         │
  │  ┌─────────────────────────────────────────────┐        │
  │  │              RAM (< 64 GB)                  │        │
  │  │                                             │        │
  │  │  Compressed PQ codes for ALL vectors        │        │
  │  │  (enables coarse distance estimation)       │        │
  │  │                                             │        │
  │  │  Navigation graph entry points              │        │
  │  └──────────────────┬──────────────────────────┘        │
  │                     │                                    │
  │                     ▼ (SSD reads for exact distances)    │
  │  ┌─────────────────────────────────────────────┐        │
  │  │              NVMe SSD (3 TB)                │        │
  │  │                                             │        │
  │  │  Full-precision vectors + graph adjacency   │        │
  │  │  Stored in sector-aligned pages             │        │
  │  │  Neighbors co-located with vector data      │        │
  │  │                                             │        │
  │  └─────────────────────────────────────────────┘        │
  │                                                         │
  │  Query: PQ beam search in RAM → SSD rerank top-C       │
  │  Typical: 1-5 SSD reads per query → < 5ms latency      │
  └─────────────────────────────────────────────────────────┘
```

### 2.2 Key Design Decisions

| Decision | DiskANN Choice | Rationale |
|---|---|---|
| Graph type | Vamana (bounded-degree) | Degree bound → predictable SSD reads |
| Max degree | R=64-128 | Each node fits in one 4KB SSD page |
| In-memory index | PQ compressed | Enables beam search without SSD I/O |
| SSD layout | Vector + neighbors co-located | Single read fetches both |
| Search strategy | Two-phase: PQ beam → SSD rerank | Minimizes random I/O |

### 2.3 Performance Profile

```
  DiskANN on 1B vectors (d=96, SIFT-1B):
  ─────────────────────────────────────────
  RAM usage:       ~40 GB (PQ codes + metadata)
  SSD usage:       ~120 GB
  Build time:      ~12 hours (single machine)
  Query latency:   ~2-5 ms at recall@10 = 0.95
  SSD reads/query: 2-8 (4KB aligned)
  Throughput:      5,000-10,000 QPS (NVMe SSD)

  Critical: NVMe SSD IOPS is the bottleneck.
  A good NVMe SSD: 500K-1M random 4KB reads/sec
  At 5 reads/query: theoretical max = 100K-200K QPS
```

---

## 3. Tiered Storage Architecture

### 3.1 Hot / Warm / Cold Tiers

```
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  HOT TIER (DRAM)              WARM TIER (NVMe SSD)          │
  │  ┌────────────────────┐       ┌────────────────────────┐    │
  │  │ HNSW index         │       │ DiskANN / Vamana index │    │
  │  │ Full-precision     │       │ PQ codes in RAM        │    │
  │  │ vectors            │       │ Full vectors on SSD    │    │
  │  │                    │       │                        │    │
  │  │ Latency: < 1ms     │       │ Latency: 2-10ms        │    │
  │  │ Cost: $$$           │       │ Cost: $$               │    │
  │  │ Capacity: ~50M/node│       │ Capacity: ~1B/node     │    │
  │  └────────────────────┘       └────────────────────────┘    │
  │                                                              │
  │  COLD TIER (Object Storage / S3)                            │
  │  ┌───────────────────────────────────────────────────┐      │
  │  │ IVF-PQ index segments                             │      │
  │  │ Loaded on-demand into warm/hot tier               │      │
  │  │                                                   │      │
  │  │ Latency: 50-500ms (first query, then cached)      │      │
  │  │ Cost: $                                           │      │
  │  │ Capacity: unlimited                               │      │
  │  └───────────────────────────────────────────────────┘      │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

### 3.2 Tier Placement Policy

```python
  # Access-frequency-based tiering
  def assign_tier(collection, access_stats):
      qps = access_stats.queries_per_second(collection)
      p99 = access_stats.latency_sla(collection)
      size = collection.vector_count * collection.dimension * 4

      if qps > 100 and p99 < 5:
          return "hot"     # DRAM — HNSW
      elif qps > 1 and p99 < 50:
          return "warm"    # SSD — DiskANN
      else:
          return "cold"    # S3 — IVF-PQ, load on demand
```

### 3.3 Cost Model

| Tier | Storage $/GB/mo | Compute | Cost for 1B vectors (768d) |
|---|---|---|---|
| Hot (DRAM) | ~$10 | High-mem instances | ~$34,000/mo |
| Warm (NVMe SSD) | ~$0.20 | Storage-optimized | ~$700/mo |
| Cold (S3) | ~$0.023 | Serverless / on-demand | ~$70/mo |

**Key insight:** A 50/40/10 hot/warm/cold split can reduce costs by 10x compared to
all-hot deployment while maintaining sub-10ms p99 for 90% of queries.

---

## 4. Distributed Vector Index Sharding

### 4.1 Sharding Strategies

```
  Strategy 1: Hash-based Sharding
  ─────────────────────────────────
  Shard assignment: shard_id = hash(vector_id) % num_shards

  Query path:  query → ALL shards (scatter) → merge top-k (gather)

  ┌──────┐     ┌──────────┐  ┌──────────┐  ┌──────────┐
  │Query │ ──→ │ Shard 0  │  │ Shard 1  │  │ Shard 2  │
  │Router│     │ top-k    │  │ top-k    │  │ top-k    │
  └──┬───┘     └────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │              │
     │         ◄────┴──────────────┴──────────────┘
     │         Merge top-k from all shards
     ▼
   Result

  Pros: Even distribution, simple scaling
  Cons: Every query hits every shard (fan-out = num_shards)


  Strategy 2: Partition-based Sharding (IVF-like)
  ─────────────────────────────────────────────────
  Cluster vectors, assign clusters to shards.
  Route queries to relevant shards only.

  ┌──────┐     ┌──────────┐
  │Query │ ──→ │ Routing  │ ──→ Shard 0 (clusters 1-100)
  │      │     │ Layer    │ ──→ Shard 2 (clusters 201-300)  ← only 2 shards
  └──────┘     └──────────┘     (based on nearest centroids)

  Pros: Reduces fan-out, lower tail latency
  Cons: Potential hotspots, imbalanced shard sizes
```

### 4.2 Replication and Consistency

```
  ┌──────────────────────────────────────────────────────┐
  │         Shard 0 Replica Group                        │
  │                                                      │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
  │  │ Primary  │  │ Replica1 │  │ Replica2 │          │
  │  │ (writes) │→ │ (reads)  │  │ (reads)  │          │
  │  └──────────┘  └──────────┘  └──────────┘          │
  │       │                                              │
  │       ▼                                              │
  │  Write path:                                         │
  │  1. Append to WAL                                    │
  │  2. Insert into mutable segment (in-memory)          │
  │  3. Async replicate to followers                     │
  │                                                      │
  │  Consistency model: EVENTUAL (typical)               │
  │  Replication lag: 100ms - 5s                         │
  │  Read-after-write: Only guaranteed on primary        │
  └──────────────────────────────────────────────────────┘
```

---

## 5. Segment-Based Architecture (Milvus Model)

### 5.1 Segment Lifecycle

```
  ┌─────────────────────────────────────────────────────────────┐
  │                  Segment Lifecycle                           │
  │                                                             │
  │  Growing Segment          Sealed Segment        Merged      │
  │  (mutable, in-memory)     (immutable, indexed)  Segment     │
  │                                                             │
  │  ┌───────────┐            ┌───────────┐        ┌────────┐  │
  │  │ Inserts   │  ──seal──→ │ Build     │ ─merge→│ Larger │  │
  │  │ (brute    │  (size or  │ ANN index │  (comp-│ indexed│  │
  │  │  force    │   time     │ (HNSW/IVF)│  action│ segment│  │
  │  │  search)  │   trigger) │           │   )    │        │  │
  │  └───────────┘            └───────────┘        └────────┘  │
  │                                                             │
  │  Search: Query ALL segments, merge results.                 │
  │  Growing segments searched via brute-force (small, fast).   │
  │  Sealed segments searched via ANN index.                    │
  └─────────────────────────────────────────────────────────────┘
```

### 5.2 Segment Parameters

| Parameter | Typical Value | Impact |
|---|---|---|
| Max segment size | 512MB - 2GB | Larger = fewer segments, better search; slower seal |
| Seal trigger | Size OR time-based | Time-based ensures freshness for streaming |
| Segments per collection | 10-1000 | More segments = more merge overhead |
| Compaction trigger | Segment count > threshold | Reduces segment sprawl |

### 5.3 Streaming Inserts vs Batch Indexing

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  STREAMING INSERT PATH                                      │
  │  ────────────────────                                       │
  │  1. Write to WAL (durable)                                  │
  │  2. Append to growing segment (in-memory, unindexed)        │
  │  3. Search growing segment via brute force                  │
  │  4. When segment fills → seal → build index (background)    │
  │                                                             │
  │  Latency to searchable: ~0ms (immediately in growing seg)   │
  │  Latency to indexed:    minutes to hours                    │
  │                                                             │
  │  BATCH INDEXING PATH                                        │
  │  ───────────────────                                        │
  │  1. Bulk load vectors into sealed segments                  │
  │  2. Build ANN index on each segment                         │
  │  3. Register segments with query nodes                      │
  │                                                             │
  │  Latency to searchable: hours (index build time)            │
  │  Index quality: BETTER (global optimization possible)       │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

**Staff-level tradeoff:** Streaming inserts provide low latency-to-searchable but fragments
the index into many small segments, each with locally-optimized ANN structures. Batch indexing
produces globally-optimized indexes (better recall) but with hours of delay. Production systems
use both: streaming for fresh data, periodic batch reindex for quality.

---

## 6. Compaction Strategies

### 6.1 Types of Compaction

```
  Minor Compaction:
  ┌──────┐ ┌──────┐ ┌──────┐        ┌──────────────┐
  │ Seg1 │ │ Seg2 │ │ Seg3 │  ───→  │  Merged Seg  │
  │ 50MB │ │ 30MB │ │ 20MB │        │    100MB     │
  └──────┘ └──────┘ └──────┘        └──────────────┘
  Merges small segments. Reduces segment count.
  Index rebuilt on merged segment.

  Major Compaction:
  ┌──────────────┐                   ┌──────────────┐
  │  Segment     │                   │  Compacted   │
  │  500MB       │       ───→        │  Segment     │
  │  30% deleted │                   │  350MB       │
  └──────────────┘                   └──────────────┘
  Removes tombstoned/deleted vectors.
  Reclaims space. Rebuilds index without gaps.

  Full Reindex:
  All segments → single new optimized index
  Most expensive, best recall, done during maintenance windows.
```

### 6.2 Compaction Triggers

| Trigger | Condition | Action |
|---|---|---|
| Segment count | > 20 small segments | Minor compaction |
| Delete ratio | > 20% deleted vectors in segment | Major compaction |
| Recall degradation | Measured recall drops below threshold | Full reindex |
| Scheduled | Nightly/weekly maintenance | Full reindex |
| Manual | Operator-triggered | Any level |

---

## 7. Memory Estimation Formulas

### 7.1 Quick Reference

```
  Per-vector memory by index type and precision:
  ──────────────────────────────────────────────────────────────

  Flat (float32):      d * 4 bytes
  Flat (float16):      d * 2 bytes
  Flat (int8/SQ8):     d * 1 byte
  PQ (m subquantizers): m bytes
  Binary:              d / 8 bytes
  HNSW overhead:       M_avg * 8 bytes (neighbor IDs)

  ──────────────────────────────────────────────────────────────
  Example: 1B vectors, d=768

  Flat float32:        1B * 768 * 4     = 3,072 GB
  Flat SQ8:            1B * 768 * 1     =   768 GB
  PQ (m=96):           1B * 96          =    96 GB
  PQ (m=48):           1B * 48          =    48 GB
  Binary (768-bit):    1B * 96          =    96 GB
  HNSW overhead:       1B * 32 * 8      =   256 GB
  ──────────────────────────────────────────────────────────────
```

### 7.2 Total System Memory

```
  Total RAM = index_memory
            + vector_storage (if not using PQ-only)
            + metadata_memory
            + query_buffers
            + OS / overhead (~20%)

  Query buffer per concurrent query:
    = efSearch * (d * 4 + 8) bytes  (candidate vectors + IDs)
    ≈ 256 * 3080 ≈ 800 KB per query

  At 1000 concurrent queries: ~800 MB query buffer
```

---

## 8. Cost Modeling at Billion Scale

### 8.1 Infrastructure Cost Comparison

```
  ┌────────────────────────────────────────────────────────────────┐
  │ 1B vectors, d=768, target: recall@10=0.95, p99 < 10ms        │
  │ Pricing: AWS us-east-1, on-demand (2025 rates)               │
  │                                                                │
  │ Option A: All in RAM (HNSW)                                   │
  │   Instances: 24x r6g.8xlarge (256GB RAM each)                 │
  │   Cost: 24 * $1.94/hr = $46.56/hr = ~$34,000/mo              │
  │   Latency: < 2ms p99                                          │
  │                                                                │
  │ Option B: DiskANN on NVMe                                     │
  │   Instances: 4x i3en.6xlarge (192GB RAM + 15TB NVMe)          │
  │   Cost: 4 * $2.71/hr = $10.84/hr = ~$7,900/mo                │
  │   Latency: < 8ms p99                                          │
  │                                                                │
  │ Option C: IVF-PQ in RAM (compressed)                          │
  │   Instances: 2x r6g.16xlarge (512GB RAM each)                 │
  │   Cost: 2 * $3.88/hr = $7.76/hr = ~$5,700/mo                 │
  │   Latency: < 5ms p99 (lower recall: ~0.90)                    │
  │                                                                │
  │ Option D: Managed service (Pinecone p2)                       │
  │   Pods: ~40 p2 pods                                           │
  │   Cost: ~$30,000/mo (estimated)                               │
  │   Latency: < 10ms p99                                         │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘
```

### 8.2 Cost Per Query

```
  Metric: $ per 1M queries (at steady-state QPS)

  Option A (HNSW):    $34,000/mo ÷ (1000 QPS * 2.6M queries/mo)
                    = ~$0.013 per 1M queries
                    Best unit economics at high QPS

  Option B (DiskANN): $7,900/mo ÷ (500 QPS * 1.3M queries/mo)
                    = ~$0.006 per 1M queries
                    Best cost per query

  Key insight: At high QPS, the cost per query for any option
  approaches zero — infrastructure cost is amortized.
  At LOW QPS (< 10), serverless/managed services win.
```

### 8.3 Build vs Buy Decision

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │   Build (FAISS/DiskANN on EC2)     Buy (Managed Service)   │
  │   ┌───────────────────────┐        ┌─────────────────────┐ │
  │   │ + Full control        │        │ + Zero ops overhead │ │
  │   │ + Lower cost at scale │        │ + Auto-scaling      │ │
  │   │ + Custom tuning       │        │ + Built-in HA       │ │
  │   │ - 2-3 engineer team   │        │ - Vendor lock-in    │ │
  │   │ - Ops burden          │        │ - Less tuning knobs │ │
  │   │ - HA is your problem  │        │ - Higher $ at scale │ │
  │   └───────────────────────┘        └─────────────────────┘ │
  │                                                             │
  │   Breakeven: ~500M+ vectors, dedicated team available       │
  └─────────────────────────────────────────────────────────────┘
```

---

## 9. Production Architecture Patterns

### 9.1 The "Lambda Architecture" for Vectors

```
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  Real-time Path              Batch Path                      │
  │  ┌──────────────────┐       ┌──────────────────────────┐    │
  │  │ Streaming inserts│       │ Nightly full reindex     │    │
  │  │ → Growing segment│       │ → Optimized sealed index │    │
  │  │ → Brute force    │       │ → Higher recall          │    │
  │  │   search         │       │ → Replace old segments   │    │
  │  └────────┬─────────┘       └──────────┬───────────────┘    │
  │           │                             │                    │
  │           ▼                             ▼                    │
  │  ┌─────────────────────────────────────────────────────┐    │
  │  │              Query Merger                            │    │
  │  │  Searches both paths, deduplicates, returns top-k   │    │
  │  └─────────────────────────────────────────────────────┘    │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

### 9.2 Reference Architecture (Billion Scale)

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                                                                 │
  │  Load Balancer                                                  │
  │       │                                                         │
  │       ▼                                                         │
  │  ┌─────────────┐                                                │
  │  │ Query Router │ ── routes by collection + tenant              │
  │  └──────┬──────┘                                                │
  │         │                                                       │
  │    ┌────┴────┬──────────┐                                       │
  │    ▼         ▼          ▼                                       │
  │  ┌──────┐ ┌──────┐ ┌──────┐   Query Nodes (stateless)         │
  │  │ QN-0 │ │ QN-1 │ │ QN-2 │   Load segments from Data Nodes   │
  │  └──┬───┘ └──┬───┘ └──┬───┘                                    │
  │     │        │        │                                         │
  │     ▼        ▼        ▼                                         │
  │  ┌──────┐ ┌──────┐ ┌──────┐   Data Nodes (stateful)           │
  │  │ DN-0 │ │ DN-1 │ │ DN-2 │   Own segments, serve vectors     │
  │  └──┬───┘ └──┬───┘ └──┬───┘                                    │
  │     │        │        │                                         │
  │     ▼        ▼        ▼                                         │
  │  ┌────────────────────────┐   Object Storage (S3)              │
  │  │ Segment files, WAL,   │   Durable storage layer             │
  │  │ index snapshots        │                                     │
  │  └────────────────────────┘                                     │
  │                                                                 │
  │  Coordination: etcd (leader election, segment assignment)       │
  │  Message bus:  Pulsar/Kafka (WAL, change stream)                │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 10. Failure Modes at Billion Scale

| Failure | Impact | Detection | Recovery |
|---|---|---|---|
| Shard OOM | Queries to that shard fail | Memory alerts, health checks | Restart + segment offload |
| Unbalanced shards | Hot shard bottlenecks cluster | QPS/latency disparity metrics | Reshard or rebalance |
| SSD failure (DiskANN) | Shard offline | SMART monitoring, health checks | Failover to replica |
| Network partition | Split-brain reads | Consensus protocol timeout | Quorum-based reads |
| Compaction storm | Latency spike during compaction | I/O and latency monitoring | Rate-limit compaction, off-peak scheduling |
| Stale embeddings | Recall degrades silently | Canary query recall monitoring | Trigger reindex pipeline |

---

## 11. Interview Narrative

### How to Present This in a Staff/Principal Interview

**Opening framing (30 seconds):**
> "Billion-scale vector search is fundamentally a systems engineering problem, not an
> algorithms problem. The ANN algorithm matters less than the storage architecture, sharding
> strategy, and cost model. My approach starts with the cost constraint and works backward
> to the architecture that meets the latency SLA within that budget."

**When asked "Design a system for 1B vector search":**

1. **Clarify requirements:** QPS target, latency SLA, recall requirement, update frequency,
   cost budget, and whether vectors are static or streaming.

2. **Size the problem:** Use the memory formulas. Show that HNSW in RAM requires ~3.4TB,
   immediately ruling out single-node. Propose IVF-PQ (96GB) or DiskANN as alternatives.

3. **Choose architecture:**
   - If latency < 2ms required: Sharded HNSW in RAM (expensive)
   - If latency < 10ms acceptable: DiskANN on NVMe (4-5x cheaper)
   - If cost is primary concern: IVF-PQ with slight recall tradeoff

4. **Address operability:** Segment-based architecture for streaming inserts. Blue-green
   deployment for reindexing. Tiered storage for cost optimization.

5. **Discuss monitoring:** Canary queries with known ground truth for recall monitoring.
   Per-shard latency tracking. Memory and SSD IOPS dashboards.

**Demonstrating E7 depth:**
- Calculate the cost per query for each option and explain which is optimal at different QPS
- Explain why DiskANN's degree bound is critical for predictable SSD I/O
- Discuss the tension between streaming inserts (freshness) and batch reindex (recall quality)
- Propose a tiered architecture with specific percentages and migration policies
- Articulate the operational cost of self-managed vs managed and where the breakeven is

**Red flags to avoid:**
- Don't propose "just add more RAM" without costing it.
- Don't ignore the network hop latency in distributed scatter-gather.
- Don't forget that index BUILD time matters — a 12-hour rebuild blocks schema changes.
- Don't say "use Pinecone" without discussing vendor lock-in and cost at scale.
