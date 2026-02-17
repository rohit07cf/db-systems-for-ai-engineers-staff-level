# Design a Billion-Scale Vector Search System

## Staff/Principal-Level System Design Exercise

---

## 1. Problem Statement

Design a vector search system that serves semantic search and recommendation queries
over 10 billion high-dimensional vectors with a p99 latency SLO of < 50ms. The system
must support real-time inserts (millions of new vectors per hour), multi-tenant
isolation, tiered storage to optimize cost, and a compaction strategy that maintains
search quality without impacting serving latency.

This is not "deploy Milvus and call it done." At staff level, you must reason about
the fundamental tradeoffs of ANN algorithms (recall vs latency vs memory), the shard
topology that determines whether your system can meet SLOs under correlated query
patterns, and the cost model that makes 10B vectors economically feasible.

---

## 2. Requirements

### 2.1 Functional Requirements

- **Vector Search**: k-NN and range queries over float32 vectors (128-2048 dimensions).
- **Filtered Search**: Combine vector similarity with metadata predicates (e.g.,
  "find similar items WHERE category = 'electronics' AND price < 100").
- **Real-Time Insert**: New vectors searchable within 5 seconds of ingestion.
- **Multi-Tenant Isolation**: Thousands of tenants sharing infrastructure with
  performance isolation guarantees.
- **CRUD Operations**: Insert, update (re-embed), delete with immediate consistency.
- **Batch Import**: Bulk load millions of vectors from offline pipelines.
- **Index Management**: Create, rebuild, and tune indices per collection.

### 2.2 Non-Functional Requirements

- **Scale**: 10B vectors across all tenants.
- **Latency**: p50 < 10ms, p99 < 50ms for top-100 queries.
- **Throughput**: 100K queries/sec globally.
- **Insert Rate**: 5M vectors/hour sustained.
- **Recall**: > 95% recall@100 (vs exact brute-force search).
- **Availability**: 99.95% per tenant SLO.
- **Cost**: < $0.10/million vectors/month for warm tier.

---

## 3. Scale Numbers

| Metric | Value |
|---|---|
| Total vectors | 10B |
| Vector dimensions | 128-2048 (varies by collection) |
| Avg dimension | 768 (typical embedding model) |
| Bytes per vector (768d float32) | 3,072 bytes |
| Raw vector data (10B x 768d) | ~30 TB |
| HNSW index overhead (~1.5x) | ~45 TB total with index |
| Tenants | ~10K |
| Largest tenant | ~500M vectors |
| Median tenant | ~100K vectors |
| Queries/sec (total) | 100K |
| Queries/sec (largest tenant) | 10K |
| Insert rate | ~1,400 vectors/sec sustained |
| Peak insert rate (batch load) | ~50K vectors/sec |
| Metadata per vector | ~200 bytes avg |
| Total metadata | ~2 TB |

---

## 4. High-Level Architecture

```
                    ┌─────────────────────────┐
                    │    Query Router / LB     │
                    │  (Tenant-aware routing)  │
                    └────────┬────────────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
     ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
     │  Query Node  │ │  Query Node  │ │  Query Node  │
     │  (Stateless) │ │  (Stateless) │ │  (Stateless) │
     │              │ │              │ │              │
     │  - Parse     │ │  - Filter    │ │  - Merge     │
     │  - Scatter   │ │  - Rerank    │ │  - Return    │
     └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
            │                │                │
            └────────────────┼────────────────┘
                             │  Scatter/Gather
            ┌────────────────┼────────────────┐
            │                │                │
     ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
     │ Shard Node 1 │ │ Shard Node 2 │ │ Shard Node N │
     │              │ │              │ │              │
     │ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐ │
     │ │ HNSW     │ │ │ │ HNSW     │ │ │ │ HNSW     │ │
     │ │ Index    │ │ │ │ Index    │ │ │ │ Index    │ │
     │ │ (mmap)   │ │ │ │ (mmap)   │ │ │ │ (mmap)   │ │
     │ └──────────┘ │ │ └──────────┘ │ │ └──────────┘ │
     │ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐ │
     │ │ Metadata │ │ │ │ Metadata │ │ │ │ Metadata │ │
     │ │ (RocksDB)│ │ │ │ (RocksDB)│ │ │ │ (RocksDB)│ │
     │ └──────────┘ │ │ └──────────┘ │ │ └──────────┘ │
     └──────────────┘ └──────────────┘ └──────────────┘
            │                │                │
     ┌──────▼────────────────▼────────────────▼──────┐
     │               Object Storage (S3)              │
     │        Segment files, snapshots, WAL           │
     └────────────────────────────────────────────────┘
```

---

## 5. Storage Layer Design

### 5.1 Index Sharding Strategy

**Sharding dimensions:**
1. **By tenant**: Each tenant's vectors are co-located (no cross-tenant index).
2. **By hash within tenant**: Large tenants are further sharded by consistent hash
   of vector ID across N shards (N = ceil(tenant_vectors / shard_target_size)).

**Shard target size**: 5M vectors per shard.
- At 768d, a 5M-vector HNSW index is ~15 GB in memory (vectors) + ~5 GB (graph).
- Fits comfortably on a 32 GB node with room for OS cache and metadata.
- Search over 5M vectors with HNSW: ~2ms per shard (ef_search=128, M=32).

**For 10B vectors**: 10B / 5M = 2,000 shards.
- With RF=2, that is 4,000 shard replicas.
- On 64 GB nodes hosting 2 shards each, need ~2,000 nodes.

**Shard placement:**
- Use a shard placement service (like Helix or custom) that balances:
  - Memory utilization across nodes.
  - Query load (move hot shards to less loaded nodes).
  - Co-location (keep same-tenant shards on nearby nodes for scatter/gather efficiency).

### 5.2 Tiered Storage Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Storage Tiers                         │
│                                                          │
│  HOT    (DRAM + NVMe)    Vectors in memory, HNSW graph  │
│  Tier    < 2ms search    For active/high-QPS tenants     │
│         ─────────────────────────────────────────────    │
│  WARM   (NVMe + mmap)   Vectors on SSD, mmap'd to RAM   │
│  Tier    < 10ms search   For moderate-QPS tenants        │
│         ─────────────────────────────────────────────    │
│  COLD   (S3 + local     DiskANN or IVF-PQ on disk       │
│  Tier    cache)          < 50ms search (with cache hit)  │
│          For long-tail tenants with sporadic queries      │
│         ─────────────────────────────────────────────    │
│  FROZEN (S3 only)        No local index. Load on demand. │
│  Tier    < 5s search     For archival / rare access      │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Tier placement policy:**
- Based on query rate over trailing 24h.
- Hot: > 10 QPS to this shard.
- Warm: 0.1 - 10 QPS.
- Cold: < 0.1 QPS but queried in last 30 days.
- Frozen: No queries in 30+ days.

**Cost impact:**
- Hot: ~$0.50/M vectors/month (DRAM-dominated).
- Warm: ~$0.10/M vectors/month (NVMe-dominated).
- Cold: ~$0.02/M vectors/month (S3 + minimal compute).
- Frozen: ~$0.005/M vectors/month (S3 storage only).

At 10B vectors with power-law distribution (10% hot, 20% warm, 40% cold, 30% frozen):
- Hot: 1B * $0.50/M = $500K/month
- Warm: 2B * $0.10/M = $200K/month
- Cold: 4B * $0.02/M = $80K/month
- Frozen: 3B * $0.005/M = $15K/month
- **Total: ~$795K/month** vs $5M/month if all hot.

### 5.3 HNSW Index Design (Primary Index)

**Parameters:**
```
M = 32              -- Max connections per node per layer
ef_construction = 200  -- Build-time search width (higher = better recall, slower build)
ef_search = 128     -- Query-time search width (tunable per query)
distance_metric = cosine  -- (or L2, inner_product)
```

**Why HNSW over IVF-PQ for hot tier?**
- HNSW provides 98%+ recall@100 at < 5ms on a 5M shard.
- IVF-PQ is more memory-efficient (PQ compression) but recall degrades to ~90%
  without re-ranking, and re-ranking adds latency.
- At hot tier, memory is the resource we trade for latency. HNSW is the right choice.

**Why DiskANN for cold tier?**
- DiskANN stores the graph on SSD and only loads the query path into memory.
- At 768d with PQ compression to 96 bytes, a 5M-vector shard needs ~480 MB on SSD
  (vs 15 GB in DRAM for HNSW). 30x memory savings.
- Search latency: ~20-40ms per shard (1-2 SSD reads).

### 5.4 Real-Time Insert Pipeline

```
Insert Flow:
  API → Kafka (topic: vector_inserts, partitioned by tenant_id)
      → Insert Worker Pool
          │
          ├── 1. Write to WAL (S3 + local)
          ├── 2. Add to in-memory buffer (flat index, brute-force searchable)
          ├── 3. When buffer reaches 10K vectors: flush to segment
          │       ├── Build HNSW index for segment
          │       └── Write segment to local NVMe + replicate to S3
          └── 4. Background: merge small segments into larger ones (compaction)

Query during insert:
  Search = HNSW(main_segments) UNION BruteForce(buffer) → merge → top-k
```

**Searchability latency**: Buffer is immediately searchable (brute-force over < 10K
vectors adds < 1ms). Target: new vectors searchable in < 5 seconds.

**Write amplification**: Each vector is written 3 times:
1. WAL (append-only, sequential).
2. Buffer flush to segment.
3. Compaction (merge segments).

Total write amplification: ~3-5x. At 5M inserts/hour = ~50 GB/hour raw data,
total write throughput = ~250 GB/hour. Manageable on NVMe.

### 5.5 Compaction Strategy

```
Level-Based Compaction:

  L0: Buffer flushes (10K vectors each, ~50 segments before compaction)
  L1: Merge L0 segments → 500K vector segments
  L2: Merge L1 segments → 5M vector segments (shard-level)

  Compaction trigger:
    L0 → L1: When L0 segment count > 50
    L1 → L2: When L1 segment count > 10

  During compaction:
    - Old segments remain searchable until new segment is ready
    - Atomic swap: Replace old segment set with new merged segment
    - Delete old segments after swap (with grace period for in-flight queries)
```

**Compaction and deletes:**
- Deleted vectors are marked with a tombstone (not immediately removed from index).
- During compaction, tombstoned vectors are excluded from the new index.
- This avoids the expensive operation of removing nodes from an HNSW graph
  (which requires re-linking neighbors).

**Compaction scheduling:**
- Off-peak hours for large compactions (L1 -> L2).
- Continuous for L0 -> L1 (small, fast, bounded).
- Throttle compaction IO to < 30% of disk bandwidth to protect query latency.

### 5.6 Multi-Tenant Isolation

**Performance isolation layers:**

1. **Index isolation**: Each tenant has dedicated index shards. No cross-tenant
   index sharing. A noisy neighbor's large index does not degrade another tenant's
   search.

2. **Query scheduling**: Per-tenant query queues with weighted fair scheduling.
   A tenant with 10x the contract gets 10x the query slots, not unbounded access.

3. **Rate limiting**: Per-tenant QPS and insert rate limits enforced at the
   query router level.

4. **Resource quotas**: Max vectors per tenant, max dimensions, max metadata size.

5. **Placement anti-affinity**: Large tenants' shards spread across many nodes.
   A single node failure affects at most 1 shard of a large tenant.

**Noisy neighbor mitigation:**
- A tenant running a batch of 10K queries in 1 second should not spike p99 for
  other tenants on the same query node.
- Solution: Query nodes are stateless and do scatter/gather. Shard nodes process
  queries in FIFO order with per-tenant admission control. If a tenant exceeds
  their fair share of shard-level compute, excess queries are queued (not rejected).

### 5.7 Filtered Search (Hybrid Query)

```
Query: "Find 100 nearest vectors to Q WHERE category='electronics' AND price < 100"

Approach 1: Pre-filter then search
  - Apply metadata filter first → candidate set
  - If candidate set is small (< 10K): brute-force search (fast)
  - If large: search HNSW on the full index, post-filter results
  - Problem: If filter is very selective, recall drops (HNSW may not visit matching vectors)

Approach 2: Attribute-aware HNSW (our approach)
  - During HNSW traversal, check metadata predicate at each visited node
  - Skip non-matching nodes but continue graph traversal
  - Increase ef_search dynamically: ef_search = base_ef * (1 / selectivity)
  - If selectivity < 1%: fall back to pre-filter + brute-force

Metadata storage:
  - RocksDB on each shard node: vector_id → {category, price, timestamp, ...}
  - Column-indexed for common filter fields (bitmap index on category, B-tree on price)
  - Filter evaluation: < 1 microsecond per vector (memory-mapped RocksDB)
```

---

## 6. Detailed Component Design

### 6.1 Query Path (Scatter/Gather)

```
1. Client sends query: {vector: [...], filter: {...}, top_k: 100, tenant_id: T}
2. Query Router:
   a. Identify shards for tenant T: [S1, S2, ..., Sn]
   b. For each shard, select a healthy replica (prefer local region)
   c. Scatter query to all selected replicas in parallel
3. Each Shard Node:
   a. Search HNSW index: return top-k candidates with distances
   b. Apply metadata filter (if attribute-aware HNSW, done during search)
   c. Return candidates to Query Node
4. Query Node:
   a. Merge results from all shards using a min-heap
   b. Global top-k selection
   c. Optional: re-rank using exact distance (if PQ-compressed shards)
   d. Return final results

Latency breakdown (p99 target: 50ms):
  - Network: Query Node → Shard Node → Query Node:  ~5ms
  - Shard search (HNSW, 5M vectors):                ~5ms
  - Scatter/gather overhead:                          ~2ms
  - Merge and rerank:                                 ~3ms
  - Total (single region):                           ~15ms
  - Tail latency (slowest shard, with jitter):       ~35ms
  - Budget remaining:                                 15ms (safety margin)
```

**Tail latency mitigation:**
- **Hedged requests**: If a shard does not respond in 20ms, send a duplicate request
  to the replica. Take whichever responds first.
- **Adaptive timeout**: If a shard consistently slow, temporarily route to its replica.
- **Partial results**: If 1 out of 20 shards is down, return results from 19 shards
  with a "degraded accuracy" flag. Better than failing the entire query.

### 6.2 Replication and Durability

```
Write path durability:
  1. Kafka (RF=3): Durable within seconds
  2. WAL on shard node (local NVMe): Durable within milliseconds
  3. Segment files replicated to S3: Durable within minutes
  4. Peer shard replica: Index rebuilt from WAL + segments

Recovery:
  - Node failure: Promote replica shard. Rebuild primary from S3 segments + WAL replay.
  - Rebuild time for 5M-vector shard: ~5 minutes (S3 download + HNSW rebuild).
  - During rebuild: Replica serves all queries (single point of failure temporarily).
```

---

## 7. Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Shard node failure | 1 shard unavailable | RF=2, auto-failover to replica in < 5s |
| Slow shard (GC, compaction) | p99 latency spike | Hedged requests, adaptive routing |
| Network partition (query ↔ shard) | Queries to affected shards fail | Retry via replica in alternate AZ |
| S3 outage | Cannot load cold-tier segments | Local NVMe cache; hot/warm tiers unaffected |
| HNSW index corruption | Wrong results | Checksums on segments; rebuild from vectors + WAL |
| OOM on shard node | Node crash | Memory limits per shard; evict cold shards first |
| Kafka consumer lag | Inserts delayed (not searchable) | Auto-scale consumers; alert on lag > 30s |
| Compaction storm | Disk IO saturates, queries slow | IO throttling; schedule large compactions off-peak |
| Tenant query spike | Other tenants degraded | Per-tenant rate limits; query admission control |
| Vector dimension mismatch | Search returns garbage | Schema validation on insert; reject mismatched dimensions |

---

## 8. Cost Analysis

| Component | Monthly Cost Estimate |
|---|---|
| Shard nodes — Hot (200 nodes, 64GB RAM) | $300K |
| Shard nodes — Warm (400 nodes, 32GB RAM + NVMe) | $250K |
| Shard nodes — Cold (100 nodes, minimal RAM) | $30K |
| Query nodes (50 nodes) | $40K |
| S3 storage (segments, WAL, snapshots) | $80K |
| Kafka cluster | $30K |
| Network (inter-AZ, cross-region) | $40K |
| Monitoring, management | $25K |
| **Total** | **~$795K/month** |

**Cost per vector**: $795K / 10B = $0.00008/vector/month.
**Cost per query**: $795K / (100K QPS * 2.6M sec/month) = $0.003/query.
**Cost per million vectors (blended)**: ~$0.08/M/month.

### Comparison to Managed Services

| Approach | Cost for 10B vectors |
|---|---|
| Pinecone (p2 pods) | ~$5M+/month |
| Qdrant Cloud (dedicated) | ~$2M+/month |
| This custom design (tiered) | ~$800K/month |
| All-hot (no tiering) | ~$5M/month |

Tiered storage provides 6x cost reduction vs all-hot for the same SLOs.

---

## 9. Evolution Path (Handling 10x Growth)

### 9.1 From 10B to 100B Vectors

- **Shard count**: 10B / 5M = 2,000 shards → 100B / 5M = 20,000 shards.
- **Scatter/gather**: A tenant with 5B vectors would have 1,000 shards. Querying
  all 1,000 in parallel has unacceptable tail latency.
- **Solution: Hierarchical search.** Build a coarse quantizer that maps the query
  to the top-50 most relevant shards (based on centroid proximity). Search only
  those 50 shards. Recall impact: < 2% if centroids are well-distributed.
- **Cost**: Tiering becomes even more important. At 100B vectors, > 80% should be
  cold/frozen.

### 9.2 Streaming Index Updates (CDC)

- Integrate with upstream databases via CDC (Debezium).
- When a product is updated, automatically re-embed and upsert the vector.
- Challenge: Embedding model inference is slow (~50ms per item). Batch updates.
- Guarantee: Vectors eventually consistent with source-of-truth within 60 seconds.

### 9.3 Multi-Modal Vectors

- Future: Store and search text, image, and audio embeddings in the same system.
- Challenge: Different dimensions (text: 768, image: 512, audio: 256).
- Solution: Per-collection schema with fixed dimensions. Cross-modal search via
  shared embedding spaces (CLIP-style models).

### 9.4 GPU-Accelerated Search

- For the hottest tenants, FAISS on GPU provides 10-100x throughput vs CPU HNSW.
- A single A100 can search 1B vectors with IVF-PQ in < 5ms.
- Use GPU nodes for top-tier tenants; CPU for the long tail.
- This is the 10x performance lever when CPU scaling hits diminishing returns.

---

## 10. Interview Narrative

### How to Present This in 45 Minutes

**Minutes 0-5: Scope and scale.** "10B vectors at 768d is ~30 TB of raw data.
The index overhead (HNSW graph) brings this to ~45 TB. The fundamental challenge
is that this does not fit in the memory of any single machine, so we must shard.
I will design the shard topology, tiered storage, and query path."

**Minutes 5-15: Shard strategy and index choice.** Walk through the 5M vectors/shard
sizing rationale. Explain HNSW parameter tuning (M=32, ef_search=128) and why these
values provide 98% recall at < 5ms per shard. Draw the scatter/gather query path.
This shows you understand ANN algorithm internals, not just API calls.

**Minutes 15-25: Tiered storage.** This is the cost optimization section that
separates senior from staff. Walk through the hot/warm/cold/frozen tiers, the
cost model ($5M vs $800K), and the tier placement policy based on query rate.
Explain why DiskANN is the right algorithm for the cold tier (SSD-optimized,
30x memory savings vs HNSW).

**Minutes 25-35: Real-time inserts and compaction.** Explain the WAL + buffer +
segment + compaction pipeline. Show that new vectors are searchable in < 5s via
the buffer brute-force search. Discuss compaction scheduling to avoid impacting
query latency.

**Minutes 35-45: Filtered search, multi-tenancy, and failure modes.** The filtered
search problem (attribute-aware HNSW with dynamic ef_search) is a nuanced topic.
Multi-tenant isolation via dedicated shards, fair scheduling, and anti-affinity
shows operational maturity. End with failure modes and hedged request strategy.

### Key Phrases That Signal Staff-Level Thinking

- "Tiered storage is the single largest cost lever. Moving from all-hot to tiered
  reduces the monthly bill from $5M to $800K — a 6x improvement with no SLO
  degradation for 90% of queries."
- "HNSW gives us 98% recall at 5ms, but it requires the full graph in memory.
  For the cold tier, DiskANN achieves 95% recall at 30ms with 30x less memory.
  The algorithm choice is dictated by the tier's cost and latency constraints."
- "Scatter/gather over 1,000 shards has unacceptable tail latency. At 100B scale,
  we need hierarchical search with a coarse quantizer to prune shards before the
  detailed search."
- "The compaction strategy is level-based, not size-tiered, because we need
  predictable write amplification. Size-tiered compaction can trigger massive
  merges that saturate disk IO and spike query latency."
- "For filtered search with selectivity < 1%, we fall back to pre-filter +
  brute-force. Trying to use HNSW with a highly selective filter leads to
  excessive graph traversal and recall collapse."
