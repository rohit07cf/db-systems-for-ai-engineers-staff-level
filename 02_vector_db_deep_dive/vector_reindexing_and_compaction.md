# Vector Reindexing and Compaction

## Staff/Principal Level — Database Systems for AI Engineers

---

## 1. Why Reindexing Is Unavoidable

Unlike traditional B-tree indexes that handle inserts gracefully, vector indexes degrade
over time. Three forces necessitate periodic reindexing:

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  Force 1: DATA DRIFT                                        │
  │  New vectors shift the distribution. IVF centroids trained  │
  │  on old data become suboptimal. HNSW graph structure        │
  │  reflects historical insertion order, not current topology. │
  │                                                             │
  │  Force 2: DELETE ACCUMULATION                               │
  │  Tombstoned vectors waste memory and pollute graph edges.   │
  │  An HNSW node with 50% deleted neighbors has degraded       │
  │  navigability. IVF clusters with many deleted vectors       │
  │  waste scan time.                                           │
  │                                                             │
  │  Force 3: EMBEDDING MODEL UPDATES                           │
  │  When you upgrade from text-embedding-ada-002 (1536d)      │
  │  to text-embedding-3-small (1536d), ALL vectors must be    │
  │  re-embedded and reindexed. The vector space is entirely   │
  │  different even if dimensions match.                        │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

---

## 2. Index Lifecycle Management

### 2.1 Lifecycle Stages

```
  ┌──────┐    ┌──────────┐    ┌────────┐    ┌──────────┐    ┌─────────┐
  │CREATE│───→│  ACTIVE   │───→│DEGRADED│───→│REINDEXING│───→│RETIRED  │
  │      │    │(serving)  │    │(recall │    │(rebuild  │    │(deleted)│
  └──────┘    └──────────┘    │ drops)  │    │ in prog) │    └─────────┘
                  ▲            └────────┘    └─────┬────┘
                  │                                │
                  └────────────────────────────────┘
                     New index becomes ACTIVE

  Transition Triggers:
  ─────────────────────
  ACTIVE → DEGRADED:
    - Recall below threshold (measured by canary queries)
    - Delete ratio > 20%
    - Model version change
    - Vector count > 2x at index creation time

  DEGRADED → REINDEXING:
    - Automatic (threshold-based) or manual trigger
    - Scheduled maintenance window
```

### 2.2 Monitoring Index Health

```python
  # Index health monitoring system
  class IndexHealthMonitor:
      def __init__(self, db, canary_queries, ground_truth):
          self.db = db
          self.canaries = canary_queries      # 100-1000 queries with known GT
          self.ground_truth = ground_truth

      def check_recall(self) -> float:
          """Run canary queries and measure recall against ground truth."""
          recalls = []
          for q, gt in zip(self.canaries, self.ground_truth):
              results = self.db.search(q, k=10)
              recall = len(set(results) & set(gt[:10])) / 10
              recalls.append(recall)
          return np.mean(recalls)

      def check_delete_ratio(self) -> float:
          """Fraction of vectors that are tombstoned."""
          stats = self.db.get_index_stats()
          return stats.deleted / (stats.total + stats.deleted)

      def should_reindex(self) -> bool:
          recall = self.check_recall()
          delete_ratio = self.check_delete_ratio()
          return (
              recall < 0.93                    # Below recall threshold
              or delete_ratio > 0.20           # Too many tombstones
              or self.db.segment_count() > 50  # Too many segments
          )
```

---

## 3. Online vs Offline Reindexing

### 3.1 Comparison

```
  ┌───────────────────────────────────────────────────────────────┐
  │                                                               │
  │  OFFLINE REINDEXING                                           │
  │  ─────────────────                                            │
  │  1. Stop writes (or queue them)                               │
  │  2. Export all vectors                                        │
  │  3. Build new index from scratch                              │
  │  4. Swap old index for new                                    │
  │  5. Replay queued writes                                      │
  │                                                               │
  │  Downtime: minutes to hours                                   │
  │  Quality: BEST (global optimization)                          │
  │  Complexity: LOW                                              │
  │                                                               │
  │  ONLINE REINDEXING                                            │
  │  ────────────────                                             │
  │  1. Continue serving from old index                           │
  │  2. Build new index in background from snapshot + WAL         │
  │  3. Catch up with writes that occurred during build           │
  │  4. Atomic switch when new index is caught up                 │
  │  5. Garbage collect old index                                 │
  │                                                               │
  │  Downtime: ZERO                                               │
  │  Quality: GOOD (may miss some optimizations)                  │
  │  Complexity: HIGH                                             │
  │                                                               │
  └───────────────────────────────────────────────────────────────┘
```

### 3.2 Online Reindexing Architecture

```
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  Write Path (dual-write during reindex):                     │
  │                                                              │
  │  Client                                                      │
  │    │                                                         │
  │    ▼                                                         │
  │  ┌──────┐                                                    │
  │  │ WAL  │───────────────────────┐                            │
  │  └──┬───┘                       │                            │
  │     │                           ▼                            │
  │     ▼                    ┌──────────────┐                    │
  │  ┌──────────────┐       │ New Index     │                    │
  │  │ Old Index    │       │ (building)    │                    │
  │  │ (serving)    │       │              │                    │
  │  └──────────────┘       └──────────────┘                    │
  │                                                              │
  │  Read Path (old index until switch):                         │
  │                                                              │
  │  Client → Old Index → Results                                │
  │                                                              │
  │  Switch Protocol:                                            │
  │  1. New index catches up to WAL tail                         │
  │  2. Pause writes briefly (< 100ms)                           │
  │  3. Drain in-flight queries on old index                     │
  │  4. Atomic pointer swap: active_index = new_index            │
  │  5. Resume writes and queries on new index                   │
  │  6. Delete old index after grace period                      │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

---

## 4. Segment Merging

### 4.1 The Segment Proliferation Problem

```
  Over time with streaming inserts:

  Day 1:   [Seg1: 100K vecs]
  Day 2:   [Seg1: 100K] [Seg2: 100K]
  Day 3:   [Seg1: 100K] [Seg2: 100K] [Seg3: 50K]
  ...
  Day 30:  [Seg1: 100K] [Seg2: 100K] ... [Seg30: 80K]

  Problems with 30 segments:
  ─────────────────────────────
  1. Query must search ALL 30 segments and merge results
  2. Each segment has its own suboptimal index
  3. Cross-segment neighbors are invisible to each index
  4. More segments = more file handles, more I/O
  5. Merge step becomes bottleneck at high QPS
```

### 4.2 Merge Strategies

```
  Strategy 1: Size-Tiered Compaction (Cassandra-style)
  ──────────────────────────────────────────────────────
  Group segments by size tier. Merge when tier has enough segments.

  Tier 0 (< 1MB):     [s1] [s2] [s3] [s4] → merge → Tier 1
  Tier 1 (1-10MB):     [S1] [S2] [S3] [S4] → merge → Tier 2
  Tier 2 (10-100MB):   [SS1] [SS2] → merge → Tier 3
  Tier 3 (100MB-1GB):  [SSS1]  ← final size

  Pros: Write-optimized, predictable I/O
  Cons: Space amplification (up to 2x), read amplification


  Strategy 2: Leveled Compaction
  ────────────────────────────────
  Each level has a size limit. Overflow triggers merge into next level.

  L0: Growing segments (unmerged)
  L1: Merged, max 500MB total
  L2: Merged, max 5GB total
  L3: Merged, max 50GB total

  Pros: Better read performance, less space amplification
  Cons: More write amplification, CPU-intensive


  Strategy 3: Hybrid (Recommended for vectors)
  ──────────────────────────────────────────────
  Minor compaction: Size-tiered for small segments (fast, frequent)
  Major compaction: Full rebuild for large segments (scheduled, infrequent)
```

### 4.3 Compaction Impact on Query Latency

```
  Latency During Compaction:
  ──────────────────────────

  p99 Latency (ms)
  20 ┤
     │              ████
  15 ┤          ████    ████
     │      ████            ████
  10 ┤  ████                    ████████
     │██                                ██████████████
   5 ┤
     │
   0 ┤
     └──┬───────┬───────┬───────┬───────┬───────┬────
       0:00   0:30    1:00    1:30    2:00    2:30
                    Time (hours)
              ▲                    ▲
        Compaction starts    Compaction ends

  Causes of latency increase:
  1. I/O bandwidth shared between compaction and queries
  2. CPU consumed by index rebuilding
  3. Memory pressure from holding old + new segments
  4. Cache pollution from sequential compaction reads
```

### 4.4 Mitigating Compaction Impact

| Strategy | How It Works | Tradeoff |
|---|---|---|
| I/O throttling | Limit compaction IOPS to 30% of available | Slower compaction |
| Off-peak scheduling | Run compaction at 2 AM | Delays cleanup |
| Dedicated compaction threads | Pin to specific CPU cores | Reduces available query CPU |
| Read-ahead disabling | Don't prefetch during compaction | Slower compaction reads |
| Incremental compaction | Compact one segment at a time | Longer total time |

---

## 5. Blue-Green Index Deployment

### 5.1 Architecture

```
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │              Load Balancer / Router                          │
  │                     │                                        │
  │            ┌────────┴────────┐                               │
  │            │   BLUE (active) │                               │
  │            │   Index v2      │←── serving 100% traffic       │
  │            │   Recall: 0.96  │                               │
  │            └─────────────────┘                               │
  │                                                              │
  │            ┌─────────────────┐                               │
  │            │  GREEN (standby)│                               │
  │            │  Index v3       │←── building / validating      │
  │            │  Recall: TBD    │                               │
  │            └─────────────────┘                               │
  │                                                              │
  │  Deployment Steps:                                           │
  │  ─────────────────                                           │
  │  1. Build GREEN index from current data                      │
  │  2. Run validation suite on GREEN:                           │
  │     - Canary recall check (must beat BLUE by >= 0%)          │
  │     - Latency benchmark (must meet SLA)                      │
  │     - Vector count parity check                              │
  │  3. Shadow traffic: route 5% to GREEN, compare results       │
  │  4. If GREEN passes: atomic switch (GREEN becomes active)    │
  │  5. Keep BLUE warm for 1 hour (rollback safety)              │
  │  6. Decommission BLUE                                        │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

### 5.2 Validation Checklist

```
  Pre-switch validation for new index:
  ┌──────────────────────────────────────────────────────┐
  │ [ ] Vector count matches (within 0.01% tolerance)    │
  │ [ ] Canary recall >= old index recall                │
  │ [ ] p50 latency <= 1.2x old index p50               │
  │ [ ] p99 latency <= 1.5x old index p99               │
  │ [ ] Memory usage within node capacity                │
  │ [ ] Shadow traffic results within 95% agreement      │
  │ [ ] No OOM during stress test                        │
  │ [ ] Metadata filters produce correct results         │
  │ [ ] Delete/update operations functional              │
  └──────────────────────────────────────────────────────┘
```

---

## 6. Schema Evolution for Embeddings

### 6.1 Dimension Changes

```
  The Dimension Change Problem:
  ─────────────────────────────
  Old model: text-embedding-ada-002 → 1536 dimensions
  New model: text-embedding-3-large → 3072 dimensions

  You CANNOT mix vectors of different dimensions in the same index.
  Every vector must be the same dimension.

  Options:
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  Option A: Truncation / Padding (BAD)                        │
  │  Pad 1536d vectors to 3072d with zeros.                     │
  │  Problem: Zeros distort distances. Recall degrades.          │
  │                                                              │
  │  Option B: Parallel Index (GOOD)                             │
  │  Run old and new indexes side by side.                       │
  │  New vectors go to new index.                                │
  │  Backfill: re-embed old documents with new model.            │
  │  Merge when backfill complete.                               │
  │                                                              │
  │  Option C: Matryoshka Embeddings (BEST if supported)         │
  │  Models like text-embedding-3 support truncation.            │
  │  Use 768d slice of 3072d vector.                             │
  │  All vectors use same truncated dimension.                   │
  │  Slight quality loss but operationally simple.               │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

### 6.2 Dimension Change Migration Plan

```
  Phase 1: Dual-Write (1-2 weeks)
  ┌──────────────────────────────────────────────┐
  │ New documents → embed with new model          │
  │                → write to NEW index (3072d)   │
  │                → write to OLD index (1536d)   │
  │                                               │
  │ Queries → search OLD index only               │
  └──────────────────────────────────────────────┘

  Phase 2: Backfill (1-4 weeks depending on corpus size)
  ┌──────────────────────────────────────────────┐
  │ Re-embed ALL old documents with new model     │
  │ Insert into NEW index                         │
  │ Rate-limit to avoid starving production       │
  │ Monitor: backfill_progress / total_docs       │
  └──────────────────────────────────────────────┘

  Phase 3: Shadow + Switch (1 week)
  ┌──────────────────────────────────────────────┐
  │ Shadow: query both indexes, log differences  │
  │ Validate: new index recall >= old            │
  │ Switch: route all queries to NEW index       │
  │ Teardown: decommission OLD index             │
  └──────────────────────────────────────────────┘

  Total migration time: 3-7 weeks for a large corpus
```

---

## 7. Model Version Migration

### 7.1 The Stale Embeddings Problem

```
  Timeline:
  ─────────────────────────────────────────────────────
  Jan 2024: Index 100M docs with model-v1
  Jun 2024: Model-v2 released (better quality)
  Jul 2024: New docs embedded with model-v2

  Problem: Queries use model-v2 embeddings, but 100M docs
  still have model-v1 embeddings. Cross-model similarity
  is MEANINGLESS — even if dimensions match.

  ┌────────────────────────────────────────────────────┐
  │                                                    │
  │  model-v1 space         model-v2 space             │
  │                                                    │
  │    A ● ─── ● B          A ●       ● B              │
  │       \   /                 \   /                   │
  │        ● C                   ● C                    │
  │                                                    │
  │  Same documents, DIFFERENT positions in space.      │
  │  Cross-model search returns GARBAGE.                │
  │                                                    │
  └────────────────────────────────────────────────────┘
```

### 7.2 Migration Strategies

| Strategy | Approach | Cost | Downtime | Quality |
|---|---|---|---|---|
| **Big bang** | Re-embed everything, swap index | $$$ (compute) | Hours | Best |
| **Incremental** | Re-embed in batches, dual-index | $$ | None | Good (during transition) |
| **Lazy** | Re-embed on read (cache new embedding) | $ | None | Degrades until complete |
| **Adapter** | Train projection matrix: v1 → v2 space | $ | None | Approximate |

### 7.3 Adapter / Projection Approach

```
  Train a linear projection: W * v1_embedding ≈ v2_embedding

  ┌────────────────────────────────────────────────────────────┐
  │                                                            │
  │  Training:                                                 │
  │  1. Sample 10K-100K documents                              │
  │  2. Embed each with both model-v1 and model-v2            │
  │  3. Learn W = argmin ||W * V1 - V2||_F (least squares)    │
  │  4. W is a d_new × d_old matrix                           │
  │                                                            │
  │  At query time:                                            │
  │  - Query embedded with model-v2 (no projection needed)    │
  │  - Old vectors projected: v2_approx = W * v1_original     │
  │  - Search in projected space                               │
  │                                                            │
  │  Pros: No re-embedding needed, instant migration           │
  │  Cons: 5-15% recall loss, only works for similar models    │
  │                                                            │
  └────────────────────────────────────────────────────────────┘
```

---

## 8. Compaction Triggers and Policies

### 8.1 Trigger Configuration

```yaml
  # Production compaction policy
  compaction:
    minor:
      trigger:
        segment_count: 10          # Merge when > 10 small segments
        min_segment_size: 10MB     # Only compact segments < 10MB
      schedule: "every 2 hours"
      max_concurrent: 1
      io_limit_mbps: 100           # Throttle to protect queries

    major:
      trigger:
        delete_ratio: 0.20         # > 20% deleted vectors
        recall_degradation: 0.05   # > 5% recall drop (canary-measured)
      schedule: "daily at 02:00 UTC"
      max_concurrent: 1
      io_limit_mbps: 200

    full_reindex:
      trigger:
        model_version_change: true
        manual: true
      schedule: "manual or weekly"
      strategy: "blue-green"
      validation_required: true
```

### 8.2 Compaction State Machine

```
  ┌──────────┐   trigger    ┌───────────┐   complete   ┌──────────┐
  │  IDLE    │ ────────────→│ COMPACTING│ ────────────→│ SWAPPING │
  │          │              │           │              │          │
  └──────────┘              └─────┬─────┘              └─────┬────┘
       ▲                          │                          │
       │                     error│                     success│
       │                          ▼                          │
       │                   ┌───────────┐                     │
       │                   │  FAILED   │                     │
       │                   │ (retry or │                     │
       └───────────────────│  alert)   │◄────────────────────┘
                           └───────────┘       failure
```

---

## 9. Impact on Query Latency During Rebuild

### 9.1 Latency Budget Analysis

```
  Normal query path (no reindex):
    Index lookup:     0.5ms
    Distance compute: 0.3ms
    Network/overhead: 0.2ms
    Total:           1.0ms

  During online reindex:
    Index lookup:     0.5ms (unchanged — querying old index)
    Distance compute: 0.3ms
    I/O contention:  +0.5ms (compaction shares disk)
    CPU contention:  +0.3ms (index build uses CPU)
    Network/overhead: 0.2ms
    Total:           1.8ms  ← 80% regression

  During segment merge (worst case):
    Must search N segments instead of 1:
    Query time ≈ N_segments * per_segment_time + merge_time
    With 20 segments: 20 * 0.3ms + 1ms = 7ms  ← 7x regression
```

### 9.2 Latency Protection Strategies

| Strategy | Implementation | Effectiveness |
|---|---|---|
| Resource isolation | Compaction on dedicated CPU cores + I/O queue | High |
| Read replicas | Route queries to replicas not being compacted | High |
| Rate limiting | Cap compaction I/O at 30% of available bandwidth | Medium |
| Priority scheduling | Query threads have higher priority than compaction | Medium |
| Off-peak scheduling | Compact during low-traffic hours only | Medium |
| Incremental compaction | Compact one small segment at a time | Low overhead |

---

## 10. Operational Runbook

### 10.1 Emergency Reindex Procedure

```
  Trigger: Recall drops below 85% (critical threshold)

  Step 1: DIAGNOSE (5 min)
    - Check canary recall metrics
    - Check delete ratio
    - Check segment count and sizes
    - Check if model version changed

  Step 2: MITIGATE (immediate)
    - Increase efSearch / nprobe by 2x (buys recall at latency cost)
    - Alert on-call team

  Step 3: PLAN REINDEX (15 min)
    - Estimate build time based on vector count
    - Verify sufficient disk space for blue-green (2x index size)
    - Verify sufficient RAM for new index

  Step 4: EXECUTE (hours)
    - Start blue-green reindex
    - Monitor build progress
    - Run validation suite on new index
    - Shadow traffic comparison

  Step 5: SWITCH (5 min)
    - Atomic switch to new index
    - Verify recall restored
    - Keep old index for 1 hour rollback window

  Step 6: POST-MORTEM
    - Why did recall degrade?
    - Can we detect earlier?
    - Adjust monitoring thresholds
```

---

## 11. Interview Narrative

### How to Present This in a Staff/Principal Interview

**Opening framing (30 seconds):**
> "Vector indexes are not self-maintaining. Unlike B-trees that handle inserts gracefully,
> vector indexes degrade through data drift, delete accumulation, and embedding model changes.
> The key operational challenge is reindexing without downtime, which I approach through
> blue-green deployment with automated validation gates."

**When asked "How do you handle embedding model upgrades?":**

1. **Frame the problem:** Old vectors and new vectors are in incompatible spaces. You cannot
   mix them. Every document must be re-embedded with the new model.

2. **Size the effort:** 100M documents at 100ms/embedding = ~3 hours on 1000 parallel workers.
   Cost: ~$500 in API calls (at $0.0001/embedding). Time is the real constraint.

3. **Execute with zero downtime:** Dual-write to old and new index. Backfill old documents
   in batches. Shadow traffic to validate. Atomic switch when complete.

4. **Discuss the adapter shortcut:** For quick transitions, train a projection matrix from
   old to new embedding space. 5-15% recall loss but instant. Good for emergency situations.

**Demonstrating E7 depth:**
- Explain why HNSW graph quality degrades with streaming inserts (insertion order matters)
- Quantify the compaction impact on p99 latency and propose mitigation strategies
- Discuss the tension between freshness (more segments) and recall (fewer, larger segments)
- Design the canary recall monitoring system with specific threshold values
- Walk through the blue-green validation checklist and explain each gate

**Red flags to avoid:**
- Don't say "just rebuild the index" without discussing downtime impact.
- Don't ignore the space requirement for blue-green (you need 2x the storage temporarily).
- Don't propose mixing embeddings from different models in the same index.
- Don't forget that reindexing at billion scale takes hours — plan accordingly.
