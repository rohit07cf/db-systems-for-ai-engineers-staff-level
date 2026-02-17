# Vector Database Failure Modes

## Staff/Principal Level — Database Systems for AI Engineers

---

## 1. Why Vector Databases Fail Differently

Vector databases have unique failure modes that don't exist in traditional databases.
The root cause: vector indexes are large, memory-hungry, approximate data structures
that degrade silently. Unlike a failed SQL query that returns an error, a degraded
vector search returns plausible-looking but wrong results.

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  Traditional DB Failure        Vector DB Failure            │
  │  ─────────────────────         ──────────────────           │
  │  "Connection refused"          "Results returned, but       │
  │  "Disk full"                    recall dropped from         │
  │  "Deadlock detected"            0.95 to 0.60 and           │
  │                                  nobody noticed"            │
  │                                                             │
  │  LOUD and OBVIOUS              SILENT and INSIDIOUS         │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

---

## 2. Failure Mode Catalog

### 2.1 Index Corruption

```
  ┌─────────────────────────────────────────────────────────────┐
  │ FAILURE: Index Corruption                                    │
  │                                                             │
  │ Symptoms:                                                    │
  │   - Segfault on query or insert                             │
  │   - Inconsistent results for same query                     │
  │   - Index file checksum mismatch                            │
  │   - Panic: "invalid neighbor ID" or "out of bounds"         │
  │                                                             │
  │ Root Causes:                                                 │
  │   - Unclean shutdown during index write                     │
  │   - Bit rot on disk (silent data corruption)                │
  │   - Bug in concurrent insert/delete code path               │
  │   - OOM killer terminated process mid-write                 │
  │   - Incomplete segment merge (partial write)                │
  │                                                             │
  │ Impact: CRITICAL — queries crash or return garbage          │
  │                                                             │
  │ Recovery:                                                    │
  │   1. Detect: checksum verification on index load            │
  │   2. Isolate: remove corrupted segment from serving         │
  │   3. Rebuild: reconstruct index from WAL or raw vectors     │
  │   4. Prevent: enable WAL, use checksummed writes,           │
  │              graceful shutdown handlers                     │
  └─────────────────────────────────────────────────────────────┘
```

### 2.2 OOM During Index Construction

```
  ┌─────────────────────────────────────────────────────────────┐
  │ FAILURE: Out of Memory During Index Build                   │
  │                                                             │
  │ Memory profile during HNSW construction:                    │
  │                                                             │
  │  RAM Usage (GB)                                             │
  │  350 ┤                                      ████ ← OOM!     │
  │  300 ┤                                  ████                │
  │  250 ┤                              ████                    │
  │  200 ┤                          ████                        │
  │  150 ┤                      ████                            │
  │  100 ┤                  ████                                │
  │   50 ┤              ████                                    │
  │    0 ┤──────────████                                        │
  │      └──┬──────┬──────┬──────┬──────┬──────┬──────┬───     │
  │        0     20M    40M    60M    80M   100M   110M        │
  │                    Vectors Inserted                          │
  │                                                             │
  │ Problem: Memory grows linearly. If you estimated 300GB      │
  │ needed but have 256GB RAM, the OOM killer strikes at ~85M   │
  │ vectors, wasting HOURS of build time.                       │
  │                                                             │
  │ Root Causes:                                                 │
  │   - Underestimated memory (forgot graph overhead)           │
  │   - Memory fragmentation during long build                  │
  │   - Co-located services competing for RAM                   │
  │   - Swap disabled (common in production)                    │
  │                                                             │
  │ Prevention:                                                  │
  │   - Pre-calculate: N * (d*4 + M*2*8 + 40) * 1.2 safety    │
  │   - Build on dedicated nodes (no co-location)               │
  │   - Use memory-mapped files (OS manages paging)             │
  │   - Checkpoint every 10M vectors (resume on failure)        │
  │   - Use IVF-PQ if HNSW doesn't fit                         │
  └─────────────────────────────────────────────────────────────┘
```

### 2.3 Segment Imbalance

```
  ┌─────────────────────────────────────────────────────────────┐
  │ FAILURE: Segment Imbalance                                   │
  │                                                             │
  │ Healthy state:                                               │
  │ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐               │
  │ │ 250MB  │ │ 240MB  │ │ 260MB  │ │ 250MB  │               │
  │ └────────┘ └────────┘ └────────┘ └────────┘               │
  │ ~Equal sizes, balanced search cost                          │
  │                                                             │
  │ Imbalanced state:                                            │
  │ ┌──────────────────────────┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐    │
  │ │         900MB            │ │5M│ │3M│ │8M│ │2M│ │1M│    │
  │ └──────────────────────────┘ └──┘ └──┘ └──┘ └──┘ └──┘    │
  │ Giant segment dominates query time + many tiny fragments   │
  │                                                             │
  │ Symptoms:                                                    │
  │   - High variance in per-segment query time                 │
  │   - Overall query latency = MAX(per-segment latency)        │
  │   - Memory waste from tiny segment overhead                 │
  │   - Compaction can't merge giant + tiny efficiently         │
  │                                                             │
  │ Root Causes:                                                 │
  │   - Streaming inserts create many small segments            │
  │   - Uneven data ingestion rates                             │
  │   - Compaction not running or failing silently              │
  │   - Bulk import mixed with streaming                        │
  │                                                             │
  │ Recovery:                                                    │
  │   - Force minor compaction to merge small segments          │
  │   - Split oversized segments                                │
  │   - Set max_segment_size and min_segment_size policies      │
  │   - Monitor segment size distribution                       │
  └─────────────────────────────────────────────────────────────┘
```

### 2.4 Query Timeout Cascades

```
  ┌─────────────────────────────────────────────────────────────┐
  │ FAILURE: Query Timeout Cascade                               │
  │                                                             │
  │ Sequence of events:                                          │
  │                                                             │
  │  1. Single shard becomes slow (compaction, GC, disk issue)  │
  │                                                             │
  │  ┌──────┐   ┌──────┐   ┌──────┐                            │
  │  │Shard0│   │Shard1│   │Shard2│                            │
  │  │ 2ms  │   │ 2ms  │   │200ms │ ← slow                    │
  │  └──────┘   └──────┘   └──────┘                            │
  │                                                             │
  │  2. Scatter-gather waits for slowest shard                  │
  │     Query latency = max(2, 2, 200) = 200ms                 │
  │                                                             │
  │  3. Upstream service times out at 100ms                     │
  │     → Retry → now 2 queries in flight                      │
  │                                                             │
  │  4. Retries multiply load on slow shard                     │
  │     → Queue grows → latency increases → more timeouts      │
  │                                                             │
  │  5. Cascade: all shards become slow due to:                 │
  │     - Connection pool exhaustion                            │
  │     - Thread pool saturation                                │
  │     - Memory pressure from queued requests                  │
  │                                                             │
  │  Total time to cascade: 30 seconds to 2 minutes             │
  │                                                             │
  │                      ┌──────────────┐                       │
  │  Normal   ──────────→│  Degraded    │──────→ Full outage    │
  │  (< 5ms)             │  (200ms+)    │       (timeouts)     │
  │                      └──────────────┘                       │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

  Mitigations:
  ┌───────────────────────────────────────────────────────────┐
  │ 1. Hedged requests: Send to 2 replicas, take first reply │
  │ 2. Adaptive timeout: Per-shard timeout based on p99 hist │
  │ 3. Circuit breaker: Skip slow shard, return partial res  │
  │ 4. Backpressure: Reject new queries when queue > limit   │
  │ 5. Retry budget: Max 1 retry per request, not per shard  │
  └───────────────────────────────────────────────────────────┘
```

### 2.5 Replication Lag for Vectors

```
  ┌─────────────────────────────────────────────────────────────┐
  │ FAILURE: Replication Lag                                     │
  │                                                             │
  │  Primary                    Replica                          │
  │  ┌──────────────────┐      ┌──────────────────┐            │
  │  │ 100M vectors     │      │ 99.8M vectors    │            │
  │  │ Index v47        │      │ Index v45        │            │
  │  │ (current)        │      │ (2 versions behind) │         │
  │  └──────────────────┘      └──────────────────┘            │
  │         │                          ▲                        │
  │         │      replication         │                        │
  │         └──────────────────────────┘                        │
  │                 lag: 200K vectors                            │
  │                                                             │
  │ Why vector replication is harder than row replication:       │
  │                                                             │
  │ 1. Row replication: Apply INSERT → O(1) per row            │
  │    Vector replication: Insert + reindex → O(log N) per vec │
  │                                                             │
  │ 2. Row replication: Deterministic — same state guaranteed   │
  │    Vector replication: HNSW insert order affects graph      │
  │    → Primary and replica may have DIFFERENT graphs          │
  │    → DIFFERENT recall characteristics!                      │
  │                                                             │
  │ 3. Sealed segments can be replicated as files (fast)        │
  │    Growing segments must replay writes (slow)               │
  │                                                             │
  │ Impact:                                                      │
  │   - Read-after-write inconsistency                          │
  │   - Different recall on primary vs replica                  │
  │   - Failover to stale replica loses recent vectors          │
  │                                                             │
  │ Mitigation:                                                  │
  │   - Replicate sealed segments as immutable files            │
  │   - Write-ahead log for growing segment catch-up            │
  │   - Stale-read detection: compare vector counts             │
  │   - Sticky sessions: route same user to same replica        │
  └─────────────────────────────────────────────────────────────┘
```

### 2.6 Stale Embeddings Problem

```
  ┌─────────────────────────────────────────────────────────────┐
  │ FAILURE: Stale Embeddings (Silent Recall Degradation)       │
  │                                                             │
  │ Timeline:                                                    │
  │                                                             │
  │ Jan: Embed 50M docs with model-v1. Recall = 0.96.          │
  │ Mar: Embed 10M new docs with model-v1. Recall = 0.95.      │
  │ Jun: Model-v2 released. Start using for queries.            │
  │      Recall drops to... 0.40? Nobody notices.               │
  │ Sep: Customer complaints about search quality.              │
  │      Root cause identified: model version mismatch.         │
  │                                                             │
  │ Why it's silent:                                             │
  │   - No errors thrown                                         │
  │   - Results are still returned (just wrong ones)            │
  │   - Latency unchanged                                       │
  │   - Only recall@k degrades (not measured in production)     │
  │                                                             │
  │ Also causes of stale embeddings:                            │
  │   - Source document updated but vector not re-computed      │
  │   - Embedding API version silently updated (rare but real)  │
  │   - Fine-tuned model weights changed (A/B test contamination│
  │                                                             │
  │ Detection:                                                   │
  │   - Canary queries with known ground truth (run hourly)     │
  │   - Track model version metadata on each vector             │
  │   - Alert when % of vectors with old model > threshold      │
  │   - Monitor user engagement metrics (CTR, relevance ratings)│
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### 2.7 Recall Degradation Over Time

```
  ┌─────────────────────────────────────────────────────────────┐
  │ FAILURE: Recall Degradation Over Time                       │
  │                                                             │
  │  Recall@10                                                   │
  │  0.98 ┤●●●●●                                                │
  │       │      ●●●                                             │
  │  0.95 ┤         ●●●                                          │
  │       │            ●●                                        │
  │  0.92 ┤              ●●                                      │
  │       │                ●●                                    │
  │  0.89 ┤                  ●●                                  │
  │       │                    ●●●                               │
  │  0.85 ┤                       ●●●●●                          │
  │       │                             ●●●                      │
  │  0.80 ┤                                ●●●●●                 │
  │       └──┬────┬────┬────┬────┬────┬────┬────┬────┬──        │
  │        Week  Week  Week  Week  Week  Week  Week  Week       │
  │         1     4     8    12    16    20    24    28          │
  │                                                             │
  │ Contributing factors:                                        │
  │                                                             │
  │ ▓▓▓▓▓▓▓▓ Delete accumulation (40% of degradation)          │
  │ ░░░░░░░░ Data distribution shift (30%)                      │
  │ ████████ Segment fragmentation (20%)                        │
  │ ▒▒▒▒▒▒▒▒ Index parameter staleness (10%)                   │
  │                                                             │
  │ Each factor compounds. Total degradation is multiplicative. │
  │                                                             │
  │ Prevention cadence:                                          │
  │   - Weekly:  Minor compaction, canary recall check          │
  │   - Monthly: Major compaction, parameter re-tuning          │
  │   - Quarterly: Full reindex with fresh parameters           │
  └─────────────────────────────────────────────────────────────┘
```

---

## 3. Failure Detection Framework

### 3.1 Monitoring Dashboard

```
  ┌─────────────────────────────────────────────────────────────┐
  │              VECTOR DB HEALTH DASHBOARD                     │
  │                                                             │
  │  ┌─ Recall Health ──────────────────────────────────────┐   │
  │  │ Canary Recall@10:  0.94  [▓▓▓▓▓▓▓▓▓░] (target: 0.95)│  │
  │  │ Trend (7d):        -0.02  ⚠ DECLINING                │  │
  │  │ Model version mix: v1=60%, v2=40%  ⚠ MIXED           │  │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  ┌─ Latency Health ────────────────────────────────────┐    │
  │  │ p50: 1.2ms   p99: 8.5ms   p999: 45ms               │    │
  │  │ Timeout rate: 0.01%  [▓▓▓▓▓▓▓▓▓▓] OK               │    │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  ┌─ Index Health ──────────────────────────────────────┐    │
  │  │ Segment count: 23  (target: < 20)  ⚠ HIGH          │    │
  │  │ Delete ratio:  15%  (threshold: 20%)                │    │
  │  │ Largest segment: 2.1GB  Smallest: 3MB               │    │
  │  │ Imbalance ratio: 700:1  ⚠ HIGH                     │    │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  ┌─ Resource Health ───────────────────────────────────┐    │
  │  │ RAM: 245GB / 256GB (96%)  ⚠ HIGH                   │    │
  │  │ SSD IOPS: 120K / 500K (24%)  OK                     │    │
  │  │ CPU: 45% (query) + 20% (compaction) = 65%          │    │
  │  │ Replication lag: 12K vectors (< 30s)  OK            │    │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### 3.2 Alert Rules

| Alert | Condition | Severity | Action |
|---|---|---|---|
| Recall Drop | canary_recall < 0.90 | P1 - Critical | Page on-call, increase efSearch |
| Recall Trend | recall_7d_delta < -0.03 | P2 - Warning | Schedule reindex |
| OOM Risk | ram_usage > 90% | P2 - Warning | Scale up or offload segments |
| Timeout Spike | timeout_rate > 1% | P1 - Critical | Circuit break slow shards |
| Segment Sprawl | segment_count > 30 | P3 - Info | Trigger compaction |
| Delete Ratio | deleted/total > 25% | P2 - Warning | Trigger major compaction |
| Replication Lag | lag_vectors > 100K | P2 - Warning | Investigate replica health |
| Model Mix | mixed_model_pct > 10% | P2 - Warning | Accelerate re-embedding |

---

## 4. Recovery Procedures

### 4.1 Recovery Decision Tree

```
  ┌─ What failed?
  │
  ├─ Index corruption
  │   ├─ Single segment? → Rebuild from WAL for that segment
  │   └─ Multiple segments? → Full rebuild from raw vectors + WAL
  │
  ├─ OOM crash
  │   ├─ During query? → Restart, add memory, reduce efSearch
  │   └─ During build? → Resume from checkpoint or use smaller index type
  │
  ├─ Recall degradation
  │   ├─ Sudden drop? → Check model version, check for corruption
  │   └─ Gradual decline? → Schedule compaction + reindex
  │
  ├─ Timeout cascade
  │   ├─ Single shard? → Circuit break that shard, serve partial results
  │   └─ All shards? → Shed load (reject low-priority queries), scale up
  │
  ├─ Replication lag
  │   ├─ Lag < 1 hour? → Wait for catch-up, monitor
  │   └─ Lag > 1 hour? → Re-snapshot primary, bootstrap replica
  │
  └─ Stale embeddings
      ├─ Model mismatch? → Freeze query model, start re-embedding pipeline
      └─ Document drift? → Trigger re-embedding for stale documents
```

### 4.2 Index Reconstruction Procedure

```
  Full Reconstruction from WAL:
  ─────────────────────────────

  Step 1: Stop writes to affected collection
  Step 2: Identify last known good checkpoint
          SELECT max(checkpoint_id) FROM index_metadata
          WHERE status = 'verified' AND collection = ?

  Step 3: Replay WAL from checkpoint
          for entry in wal.read_from(checkpoint_id):
              if entry.type == INSERT:
                  new_index.add(entry.vector, entry.id)
              elif entry.type == DELETE:
                  new_index.mark_deleted(entry.id)

  Step 4: Verify reconstruction
          - Vector count matches expected
          - Canary recall meets threshold
          - Spot-check 1000 random vectors for data integrity

  Step 5: Swap to reconstructed index
  Step 6: Resume writes
  Step 7: Verify steady-state health for 1 hour
```

---

## 5. Preventing Failures: Design Patterns

### 5.1 Defensive Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                 DEFENSIVE VECTOR ARCHITECTURE                │
  │                                                             │
  │  1. CHECKSUMS EVERYWHERE                                     │
  │     - Per-segment checksum in manifest file                 │
  │     - Per-vector CRC32 for corruption detection             │
  │     - Verify on load, periodically verify at rest           │
  │                                                             │
  │  2. WRITE-AHEAD LOG                                         │
  │     - Every insert/delete/update goes to WAL first          │
  │     - WAL enables reconstruction after any crash            │
  │     - Replicate WAL for disaster recovery                   │
  │                                                             │
  │  3. IMMUTABLE SEGMENTS                                      │
  │     - Once sealed, segments never modified                  │
  │     - Deletes tracked in separate deletion bitmap           │
  │     - Eliminates corruption from concurrent modification    │
  │                                                             │
  │  4. CIRCUIT BREAKERS                                        │
  │     - Per-shard circuit breaker (5 failures → open)         │
  │     - Fallback: return results from healthy shards          │
  │     - Half-open: test with single query every 10s           │
  │                                                             │
  │  5. CANARY RECALL MONITORING                                │
  │     - 100 pre-computed query/result pairs                   │
  │     - Run every 5 minutes                                   │
  │     - Alert if recall drops > 5% from baseline              │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### 5.2 Chaos Engineering for Vector DBs

| Experiment | Method | Validates |
|---|---|---|
| Kill random shard | SIGKILL one node | Failover to replica, partial results |
| Fill memory to 95% | Memory balloon process | Graceful degradation, no OOM |
| Inject network delay | tc netem 200ms on one shard | Timeout handling, hedged requests |
| Corrupt segment file | Flip random bits | Checksum detection, auto-rebuild |
| Block SSD I/O | dm-delay or cgroups | DiskANN fallback behavior |
| Stale model injection | Insert vectors with wrong model version | Model version detection |

---

## 6. Failure Impact by System Component

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  Component           Failure Probability    Impact Severity      │
  │  ─────────────────   ─────────────────────  ──────────────       │
  │                                                                  │
  │  HNSW Graph          Low (in-memory,        CRITICAL             │
  │                       rare corruption)       (full rebuild)      │
  │                                                                  │
  │  IVF Centroids       Very Low               HIGH                 │
  │                       (small, read-only)     (retrain needed)    │
  │                                                                  │
  │  PQ Codebook         Very Low               HIGH                 │
  │                       (small, read-only)     (retrain needed)    │
  │                                                                  │
  │  Segment Metadata    Medium                  MEDIUM               │
  │                       (frequent writes)      (rebuild from WAL)  │
  │                                                                  │
  │  WAL                 Low                     CRITICAL             │
  │                       (append-only)          (data loss if lost) │
  │                                                                  │
  │  Raw Vectors (S3)    Very Low               CATASTROPHIC         │
  │                       (cloud durability)     (unrecoverable)     │
  │                                                                  │
  │  Query Router        Medium                  HIGH                 │
  │                       (stateless, but        (total outage)      │
  │                        single point)                              │
  │                                                                  │
  │  Embedding Model     External dependency    CRITICAL             │
  │  API                  (API outage)          (can't embed new     │
  │                                              queries/docs)       │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 7. Real-World War Stories

### 7.1 The Silent Recall Degradation

```
  Company: Series-B startup, RAG-based product
  Setup: Qdrant, 5M vectors, HNSW, single node

  What happened:
  - Embedding model provider updated their API endpoint
  - New embeddings were subtly different (different tokenizer version)
  - Old vectors: model-v1.2, new vectors: model-v1.3
  - No error. No dimension change. Just different vector spaces.
  - Over 3 months, 40% of corpus was model-v1.3, 60% model-v1.2
  - Recall degraded from 0.94 to 0.71
  - Customer NPS dropped 15 points before root cause identified

  Lesson: Track embedding model version as metadata on EVERY vector.
  Alert when version heterogeneity > 5%.
```

### 7.2 The Compaction-Induced Outage

```
  Company: E-commerce, product search
  Setup: Milvus, 50M vectors, distributed, 3 nodes

  What happened:
  - Auto-compaction triggered on all 3 nodes simultaneously
  - Compaction consumed 80% I/O bandwidth
  - Query latency spiked from 5ms to 500ms
  - Upstream services timed out at 100ms
  - Retry storm: 3x query volume hit the cluster
  - All 3 nodes became unresponsive
  - 15-minute outage during peak shopping hours

  Lesson: Stagger compaction across nodes. Never compact more than
  1 node in a replica set simultaneously. Rate-limit compaction I/O.
```

---

## 8. Interview Narrative

### How to Present This in a Staff/Principal Interview

**Opening framing (30 seconds):**
> "The most dangerous aspect of vector database failures is their silent nature. Unlike a
> traditional database where failures are loud — connection errors, constraint violations —
> vector search failures manifest as subtly wrong results. The system returns 10 results,
> they look plausible, but recall has silently dropped from 0.95 to 0.60. My approach to
> reliability centers on making the invisible visible through canary recall monitoring."

**When asked "What are the biggest operational risks with vector databases?":**

1. **Silent recall degradation** (the biggest risk): Caused by stale embeddings, delete
   accumulation, and data drift. Detected only through active recall monitoring with
   canary queries. I run 100 pre-computed queries every 5 minutes and alert if recall
   drops more than 5% from baseline.

2. **OOM during index construction:** HNSW memory grows linearly and is hard to predict
   precisely due to fragmentation. I always build on dedicated instances with 20%
   memory headroom and checkpoint every 10M vectors.

3. **Query timeout cascades:** In distributed setups, one slow shard can cascade to
   a full outage through retries. I use hedged requests, per-shard circuit breakers,
   and a retry budget of max 1 retry per request.

4. **Model version contamination:** Mixing embeddings from different model versions
   destroys recall silently. I track model version as metadata and alert on heterogeneity.

**Demonstrating E7 depth:**
- Explain why HNSW graph corruption is harder to detect than B-tree corruption
- Walk through the cascade failure sequence: slow shard -> retry storm -> full outage
- Discuss the WAL-based reconstruction process with specific verification steps
- Propose the chaos engineering experiments to validate your failure handling
- Quantify the business impact: "a 10% recall drop in RAG means 10% more hallucinations"

**Red flags to avoid:**
- Don't say "we'd just restart the service" without discussing data durability.
- Don't ignore the silent nature of recall degradation.
- Don't propose monitoring only latency and error rate — recall is the critical metric.
- Don't forget that vector replication is non-deterministic (different graph != same recall).
