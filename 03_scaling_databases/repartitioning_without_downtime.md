# Repartitioning Without Downtime: Live Migration at Scale

## Why This Matters at Staff+ Level

Every growing system eventually outgrows its partition scheme. The ability to reshard, rebalance, or
migrate data without user-visible downtime separates production-grade systems from academic exercises.
At the staff/principal level, you must design migration strategies that maintain read/write
availability, data consistency, and rollback safety throughout multi-hour (or multi-day) operations.

---

## 1. The Fundamental Tension

```
  ┌──────────────────────────────────────────────────────┐
  │                  MIGRATION TRILEMMA                   │
  │                                                      │
  │           Consistency                                │
  │              /\                                      │
  │             /  \        Pick two during migration.   │
  │            /    \       The third degrades.          │
  │           /      \                                   │
  │          /________\                                  │
  │   Availability    Performance                        │
  │                                                      │
  └──────────────────────────────────────────────────────┘
```

Most production systems sacrifice a controlled amount of performance (increased latency, reduced
throughput) to maintain both consistency and availability during migration.

---

## 2. Online Schema Migration: The Double-Write Strategy

### Concept

Write to both the old and new shard layout simultaneously. Reads transition from old to new
once the new layout is fully populated and validated.

### Timeline

```
Phase 1: PREPARE (hours 0-1)
─────────────────────────────────────────────────
  - Deploy new shards (empty)
  - Update routing layer to support dual-write
  - Begin double-writing all new mutations

Phase 2: BACKFILL (hours 1-48)
─────────────────────────────────────────────────
  - Copy historical data from old shards to new shards
  - Double-writes continue for new mutations
  - Reads still served from old shards
  - Track backfill progress per partition

Phase 3: VERIFY (hours 48-52)
─────────────────────────────────────────────────
  - Run consistency checker (row counts, checksums)
  - Shadow-read from new shards, compare with old
  - Fix any discrepancies from race conditions

Phase 4: CUTOVER (hours 52-53)
─────────────────────────────────────────────────
  - Switch reads to new shards (canary -> 10% -> 100%)
  - Continue double-writes for rollback safety

Phase 5: CLEANUP (hours 53-72)
─────────────────────────────────────────────────
  - Stop writes to old shards
  - Monitor for errors / stale reads
  - Decommission old shards after bake period
```

### Architecture

```
            ┌─────────────┐
            │   App Layer  │
            └──────┬──────┘
                   │
            ┌──────▼──────┐
            │  Router /    │
            │  Proxy Layer │
            └──┬───────┬──┘
               │       │
        ┌──────▼──┐ ┌──▼──────┐
        │  Old    │ │  New     │
        │  Shards │ │  Shards  │
        │ (read)  │ │ (write)  │
        │ (write) │ │ (backfill│
        └─────────┘ └──────────┘

  Phase 1-2: Reads from Old, Writes to Both
  Phase 3:   Shadow reads from New
  Phase 4:   Reads from New, Writes to Both
  Phase 5:   Writes only to New
```

### Failure Modes

| Failure                        | Mitigation                                       |
|--------------------------------|--------------------------------------------------|
| Double-write to new shard fails | Async retry queue; flag for reconciliation       |
| Backfill falls behind mutations | Checkpoint-based backfill with change capture    |
| Consistency check finds gaps    | Pause cutover; run targeted re-backfill          |
| New shard performance is worse  | Roll back reads to old shards (writes still dual)|

---

## 3. Ghost Table Approach (gh-ost Pattern)

GitHub's `gh-ost` pioneered triggerless online schema migrations for MySQL. The pattern generalizes
to any repartitioning scenario.

### How It Works

```
Step 1: Create ghost table with new schema/partition layout
Step 2: Stream binlog (CDC) to apply ongoing mutations to ghost table
Step 3: Copy existing rows in chunks to ghost table
Step 4: When ghost table catches up to binlog position, do atomic rename

  ┌──────────────┐     binlog stream      ┌──────────────┐
  │  Original     │ ───────────────────── │  Ghost Table  │
  │  Table        │                        │  (new layout) │
  │               │     chunk copy         │               │
  │  100M rows    │ ────── 50k/batch ───> │  100M rows    │
  └──────────────┘                        └──────────────┘
                         │
                    RENAME TABLE
                    original -> _old
                    ghost -> original
                    (atomic, sub-second)
```

### Advantages Over Trigger-Based Approaches

| Aspect             | Trigger-Based (pt-osc)        | Binlog-Based (gh-ost)          |
|--------------------|-------------------------------|--------------------------------|
| Write amplification | 2x (trigger fires per row)   | 1x (async binlog consumption)  |
| Lock contention    | Row-level locks on trigger    | No locks during copy           |
| Throttling         | Difficult to control          | Built-in throttle based on lag |
| Replication lag    | Contributes to lag            | Can pause based on replica lag |
| Rollback           | Drop triggers, drop ghost     | Stop binlog stream, drop ghost |

### Critical Detail: Cut-Over

The final table swap must be atomic. gh-ost uses a technique involving:

1. Lock the original table (brief, sub-second)
2. Rename original -> _old, ghost -> original
3. Release lock

During this lock window (typically < 1 second), writes queue in the connection pool. This is
the only moment of "downtime" -- and it is measured in milliseconds.

---

## 4. Consistent Hashing for Live Resharding

When adding or removing nodes from a hash-based shard scheme, consistent hashing minimizes data
movement.

### Standard Hash Ring

```
        Node A          Node B
           \              /
            \   ┌────┐   /
             \  │    │  /
              ──│Ring│──
             /  │    │  \
            /   └────┘   \
           /              \
        Node D          Node C

  Adding Node E between A and B:
  - Only keys in range (A, E] move from B to E
  - All other keys stay put
  - Data movement: ~1/N of total (ideal)
```

### Virtual Nodes for Uniform Distribution

```
Physical Nodes: A, B, C
Virtual Nodes:  A0, A1, A2, B0, B1, B2, C0, C1, C2

Ring Position:  0----A0--B2--C1--A2--B0--C0--A1--B1--C2----MAX

Adding Node D (with D0, D1, D2):
  - D0 takes range from its predecessor to D0
  - D1 takes range from its predecessor to D1
  - D2 takes range from its predecessor to D2
  - Movement spread across all existing nodes (balanced)
```

### Migration Process with Consistent Hashing

```
T0: Current state
    Ring: [A, B, C] owning 33% each

T1: Add node D to ring metadata (not yet serving)
    Ring: [A, B, C, D] -- D marked as "joining"
    Reads:  still from [A, B, C]
    Writes: dual-write to old owner AND D for affected ranges

T2: Background data transfer
    Copy affected key ranges from [A, B, C] to D
    Track progress per vnode

T3: Validation
    Compare checksums for migrated ranges
    Shadow-read from D, compare with source

T4: Activation
    D marked as "active"
    Reads for D's ranges now served by D
    Stop dual-writes to old owners for those ranges

T5: Cleanup
    Old owners delete migrated key ranges (async, after bake period)
```

---

## 5. Backfill Patterns

### Chunk-Based Backfill

```python
# Pseudocode: chunk-based backfill with checkpointing
def backfill(source, target, chunk_size=5000):
    last_key = load_checkpoint() or MIN_KEY
    while True:
        rows = source.query(
            "SELECT * FROM data WHERE pk > %s ORDER BY pk LIMIT %s",
            last_key, chunk_size
        )
        if not rows:
            break
        target.bulk_insert(rows)
        last_key = rows[-1].pk
        save_checkpoint(last_key)
        throttle_if_needed(source.replica_lag())
```

### Backfill + CDC Race Condition

```
Timeline Problem:
  T1: Backfill copies row X (value=100) from old shard
  T2: User updates row X (value=200) on old shard
  T3: CDC delivers update (value=200) to new shard
  T4: Backfill writes row X (value=100) to new shard  <-- STALE!

Solution: Last-Write-Wins with version vectors
  - Every row has a monotonic version (e.g., binlog position)
  - New shard rejects writes with version <= current version
  - CDC events carry version; backfill rows carry version
  - Higher version always wins
```

### Throttling Strategy

| Signal               | Action                                   |
|----------------------|------------------------------------------|
| Replica lag > 5s     | Pause backfill for 30s                   |
| CPU > 80% on source  | Reduce chunk size by 50%                 |
| P99 latency > 2x     | Pause backfill, alert on-call            |
| Error rate > 0.1%    | Stop backfill, investigate               |
| Backfill progress    | Log ETA, expose in dashboard             |

---

## 6. Read/Write Splitting During Migration

### Gradual Read Cutover

```
Phase 1: 100% reads from old, 0% from new (baseline)
Phase 2:   1% reads from new (canary)
Phase 3:  10% reads from new (validate latency + correctness)
Phase 4:  50% reads from new (validate throughput)
Phase 5: 100% reads from new (full cutover)

  ┌──────────┐     Phase 2-4: compare responses
  │ App      │ ──── read(old) ──> response_old
  │ Layer    │ ──── read(new) ──> response_new
  │          │                    compare(old, new)
  │          │                    return response_old (safe)
  └──────────┘
```

### Shadow Read Validation

For each shadow read, log:
- Key/query parameters
- Response from old shard
- Response from new shard
- Match (yes/no)
- Latency delta

A mismatch rate > 0 must be investigated before proceeding to the next phase.

---

## 7. Rollback Strategies

### Rollback Decision Matrix

| Migration Phase | Rollback Complexity | Data Loss Risk | Method                        |
|-----------------|--------------------:|:--------------:|-------------------------------|
| Prepare         | Trivial             | None           | Remove dual-write config      |
| Backfill        | Low                 | None           | Stop backfill, drop ghost     |
| Verify          | Low                 | None           | Abandon verification          |
| Cutover (early) | Medium              | Low            | Switch reads back to old      |
| Cutover (late)  | High                | Medium         | Reverse dual-write direction  |
| Cleanup         | Very High           | High           | Restore from backup           |

### Golden Rule

**Never decommission old shards until the new shards have survived a full business cycle
(typically 7 days) without rollback.** Storage is cheap; re-migration is not.

---

## 8. Validation Approaches

### Row-Count Validation

```sql
-- Quick sanity check (not sufficient alone)
SELECT COUNT(*) FROM old_shard.table;
SELECT COUNT(*) FROM new_shard.table;
-- Must match after backfill + CDC catch-up
```

### Checksum Validation

```
For each partition P:
  old_checksum = CRC32(SELECT * FROM old_shard WHERE partition = P ORDER BY pk)
  new_checksum = CRC32(SELECT * FROM new_shard WHERE partition = P ORDER BY pk)
  assert old_checksum == new_checksum
```

### Application-Level Validation

```
For sample of recent transactions:
  result_old = query(old_shard, transaction_id)
  result_new = query(new_shard, transaction_id)
  assert result_old == result_new

  # Also validate aggregates
  sum_old = SELECT SUM(amount) FROM old_shard WHERE date = today
  sum_new = SELECT SUM(amount) FROM new_shard WHERE date = today
  assert abs(sum_old - sum_new) < epsilon
```

---

## 9. Real-World Patterns

### Stripe: Online Repartitioning of Payment Data

- Stripe uses a custom double-write framework
- Every migration has an explicit "migration object" tracked in a state machine
- States: `pending -> dual_writing -> backfilling -> verifying -> cutover -> complete`
- Each state transition requires human approval for payment-critical tables
- Rollback is automated for all states before cutover

### Shopify: Vitess-Based Resharding

- Shopify uses Vitess (MySQL sharding middleware)
- Vitess `Reshard` workflow: create target shards, start VReplication streams, verify, cutover
- VReplication = binlog-based CDC, similar to gh-ost
- Cutover uses a brief (< 1s) read-only period to ensure consistency

---

## Interview Narrative

**When asked about zero-downtime repartitioning, structure your answer as follows:**

> "I approach live repartitioning as a state machine with explicit phases: prepare, dual-write,
> backfill, verify, cutover, and cleanup. Each phase has defined entry/exit criteria, rollback
> procedures, and monitoring thresholds.
>
> The core technique is the double-write pattern: once new shards are provisioned, all writes go to
> both old and new shard layouts. Meanwhile, a chunk-based backfill copies historical data with
> checkpointing for restartability. The critical subtlety is the race condition between backfill
> and real-time writes -- I solve this with version vectors so that the latest write always wins
> regardless of arrival order.
>
> For the cutover, I use gradual read shifting: 1% canary, then 10%, 50%, 100%. At each stage,
> shadow reads compare old and new shard responses. Any mismatch halts the rollout. This is
> similar to what GitHub's gh-ost does for schema migrations, but generalized to shard topology
> changes.
>
> The most important operational principle is: never decommission old shards until the new layout
> has survived a full business cycle. I have seen teams rush cleanup and then discover edge cases
> (batch jobs, monthly reports) that still reference the old layout.
>
> For the specific mechanism, I prefer consistent hashing with virtual nodes because it minimizes
> data movement -- adding a node moves approximately 1/N of the data, and virtual nodes ensure
> that movement is evenly distributed across all existing nodes rather than burdening one neighbor.
>
> At Stripe-like scale, each phase transition goes through a human approval gate for critical
> data paths. For non-critical paths, the state machine can auto-advance with sufficient
> validation metrics."

**Follow-up traps to prepare for:**

- "What if the backfill takes longer than expected and you are dual-writing for days?" (Answer:
  dual-writes are idempotent and the write overhead is bounded; the real risk is write queue
  buildup if new shards are slower, so monitor write latency per shard)
- "How do you handle in-flight transactions during cutover?" (Answer: drain in-flight requests,
  brief read-only window, or use logical timestamps to order concurrent operations)
- "What about foreign key relationships across shards?" (Answer: co-locate related entities on
  the same shard via shard key design; cross-shard references use eventual consistency)
