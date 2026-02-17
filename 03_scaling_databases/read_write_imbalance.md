# Read/Write Imbalance: Optimizing for Asymmetric Workloads

## Why This Matters at Staff+ Level

Most production databases exhibit a severe read/write imbalance -- typically 10:1 to 1000:1.
Designing a system that treats reads and writes symmetrically wastes resources and creates
artificial bottlenecks. At the staff/principal level, you must identify the dominant access
pattern, apply the correct architectural pattern (CQRS, read replicas, write-behind), and
navigate the consistency tradeoffs that each pattern introduces.

---

## 1. Measuring the Imbalance

Before optimizing, quantify the imbalance precisely.

### Key Metrics

| Metric                        | How to Measure                                  |
|-------------------------------|--------------------------------------------------|
| Read/Write QPS ratio          | `SELECT` count vs `INSERT/UPDATE/DELETE` count    |
| Read/Write byte ratio         | Bytes scanned vs bytes written per second         |
| Read latency (P50/P99)        | Query-level latency histograms                    |
| Write latency (P50/P99)       | Mutation-level latency histograms                 |
| Read/Write CPU split          | Per-query CPU time attribution                    |
| Connection pool utilization   | Read connections vs write connections              |
| Lock contention               | Row lock wait time, deadlock frequency            |

### Workload Classification

```
  Read/Write Ratio        Workload Class         Example
  ─────────────────────────────────────────────────────────
  > 100:1                 Read-dominated          Product catalog
  10:1 - 100:1            Read-heavy              Social media feed
  2:1 - 10:1              Mixed                   E-commerce checkout
  1:2 - 1:1               Write-heavy             IoT telemetry
  < 1:10                  Write-dominated          Log ingestion

  ┌──────────────────────────────────────────────────────┐
  │  Typical Workload Spectrum                           │
  │                                                      │
  │  Reads ◄─────────────────────────────────► Writes    │
  │                                                      │
  │  CDN     Catalog   Feed    Cart   Events   Logs      │
  │  1000:1  100:1     20:1    5:1    1:5      1:100     │
  └──────────────────────────────────────────────────────┘
```

---

## 2. Read-Heavy Optimization Patterns

### 2a. Read Replicas

The most fundamental pattern: replicate writes to multiple read-only copies.

```
                     ┌──────────────┐
        Writes ─────>│   Primary    │
                     │   (Leader)   │
                     └──────┬───────┘
                            │ replication stream
               ┌────────────┼────────────┐
               │            │            │
         ┌─────▼────┐ ┌────▼─────┐ ┌───▼──────┐
  Reads─>│ Replica 1│ │ Replica 2│ │ Replica 3│
         └──────────┘ └──────────┘ └──────────┘

  Read capacity scales linearly with replicas.
  Write capacity stays constant (single primary).
```

### Consistency Model Tradeoffs

| Replication Mode   | Replication Lag | Write Latency | Read Consistency       |
|--------------------|:--------------:|:-------------:|------------------------|
| Synchronous        | 0              | High (+RTT)   | Strong                 |
| Semi-synchronous   | ~0 (1 replica) | Medium        | Strong (1), eventual (N-1)|
| Asynchronous       | 10ms - 5s      | Low           | Eventual               |

### Read-After-Write Consistency

The most common pain point with read replicas: a user writes data, then immediately reads
it from a replica that has not yet received the write.

```
  T0: User writes comment to Primary
  T1: User reads comment list from Replica  <-- comment missing!
  T2: Replication delivers write to Replica
  T3: User refreshes -- comment appears

  Solutions:
  ┌─────────────────────────────────────────────────────────────┐
  │ 1. Sticky sessions: route user's reads to primary for N    │
  │    seconds after their last write                          │
  │                                                            │
  │ 2. Read-your-writes token: write returns LSN; reads with   │
  │    LSN > replica_LSN route to primary                      │
  │                                                            │
  │ 3. Client-side cache: optimistically show the written      │
  │    data from local state, reconcile on next server read    │
  │                                                            │
  │ 4. Causal consistency: tag reads with the causal           │
  │    dependency (write timestamp), replica waits until       │
  │    it has caught up to that timestamp before responding    │
  └─────────────────────────────────────────────────────────────┘
```

### 2b. Materialized Views / Precomputed Reads

For complex queries (joins, aggregations), precompute the result and serve it directly.

```
  Raw Tables:                    Materialized View:
  ┌──────────┐  ┌──────────┐    ┌───────────────────────┐
  │ orders   │  │ products │    │ order_summary_by_user │
  │ (10M)    │  │ (100K)   │    │                       │
  │          │  │          │    │ user_id | total | cnt  │
  │ user_id  │──│ prod_id  │    │ u1      | $5200 | 47  │
  │ prod_id  │  │ name     │    │ u2      | $890  | 12  │
  │ amount   │  │ category │    │ ...                   │
  └──────────┘  └──────────┘    └───────────────────────┘

  Query on raw tables:  500ms (join + aggregation)
  Query on mat. view:   2ms (single row lookup)
  Cost: storage + refresh latency
```

---

## 3. Write-Heavy Optimization Patterns

### 3a. Write-Behind Caching (Write-Back)

Buffer writes in a fast store (cache/queue), flush to the database asynchronously.

```
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  Client   │────>│  Write   │────>│  Queue   │
  │           │     │  Buffer  │     │ (Kafka/  │
  │           │     │ (Redis)  │     │  Redis)  │
  └──────────┘     └──────────┘     └────┬─────┘
                                         │ async flush
                                         │ (batch)
                                    ┌────▼─────┐
                                    │ Database │
                                    │ (batch   │
                                    │  insert) │
                                    └──────────┘
```

### Write-Behind Tradeoffs

| Aspect             | Direct Write            | Write-Behind                  |
|--------------------|------------------------|-------------------------------|
| Write latency      | 5-50ms (DB round-trip) | <1ms (buffer acknowledgment)  |
| Durability         | Immediate              | At-risk until flush           |
| Throughput         | Limited by DB IOPS     | Limited by buffer throughput  |
| Read-after-write   | Consistent             | Must read from buffer first   |
| Failure mode       | DB failure = write fails| Buffer crash = data loss     |
| Batch efficiency   | Single row inserts     | Bulk inserts (10-100x faster) |

### 3b. Batch Write Optimization

Group individual writes into batches to reduce per-write overhead.

```
  Individual Writes (N = 1000):
    1000 network round-trips
    1000 transaction commits
    1000 WAL syncs
    Total: ~5000ms

  Batched Writes (10 batches of 100):
    10 network round-trips
    10 transaction commits
    10 WAL syncs
    Total: ~50ms  (100x improvement)

  ┌──────────────────────────────────────────────┐
  │  Batch Size vs Throughput                    │
  │                                              │
  │  Throughput                                  │
  │  (rows/sec)                                  │
  │    │          ┌──────────────────             │
  │ 50k│         /                               │
  │    │        /                                │
  │ 10k│       /                                 │
  │    │      /                                  │
  │  1k│ ────/                                   │
  │    └────────────────────────────             │
  │      1   10   100  1000  10000               │
  │              Batch Size                      │
  │                                              │
  │  Diminishing returns after ~1000 rows/batch  │
  └──────────────────────────────────────────────┘
```

### 3c. Append-Only / LSM-Tree Storage

For write-dominated workloads, use storage engines optimized for sequential writes.

```
  B-Tree (read-optimized):          LSM-Tree (write-optimized):
  ┌──────────────────┐              ┌──────────────────┐
  │  Random I/O on   │              │  Sequential I/O  │
  │  every write     │              │  (memtable flush) │
  │  (page splits,   │              │                  │
  │   rebalancing)   │              │  Compaction cost │
  │                  │              │  amortized over  │
  │  Read: O(log N)  │              │  time            │
  │  Write: O(log N) │              │                  │
  │  + random I/O    │              │  Read: O(log N)  │
  └──────────────────┘              │  + L levels      │
                                    │  Write: O(1)     │
                                    │  amortized       │
                                    └──────────────────┘

  Use B-Tree (PostgreSQL, MySQL/InnoDB): read-heavy, point lookups
  Use LSM (RocksDB, Cassandra, ScyllaDB): write-heavy, time-series, append-only
```

---

## 4. CQRS: Command Query Responsibility Segregation

The architectural pattern for extreme read/write imbalance: separate the read model from the
write model entirely.

### Architecture

```
  ┌──────────┐                              ┌──────────────┐
  │ Commands │                              │   Queries    │
  │ (writes) │                              │   (reads)    │
  └────┬─────┘                              └──────┬───────┘
       │                                           │
  ┌────▼──────────┐     Event Bus          ┌──────▼────────┐
  │  Write Model   │───────────────────────>│  Read Model   │
  │  (normalized)  │   (Kafka / EventBridge)│ (denormalized)│
  │                │                        │               │
  │  PostgreSQL    │                        │  Elasticsearch│
  │  (source of    │                        │  Redis        │
  │   truth)       │                        │  DynamoDB     │
  └────────────────┘                        └───────────────┘

  Write path: Client -> Command Handler -> Write DB -> Event Published
  Read path:  Client -> Query Handler -> Read Store -> Response
  Sync:       Event Consumer updates Read Store from Write DB events
```

### CQRS Tradeoff Matrix

| Aspect                    | Monolithic DB        | CQRS                          |
|---------------------------|---------------------|-------------------------------|
| Write throughput          | Shared with reads   | Dedicated, optimized          |
| Read throughput           | Shared with writes  | Dedicated, scaled independently|
| Read model schema         | Same as write       | Optimized per query pattern   |
| Consistency               | Strong (same DB)    | Eventually consistent         |
| Operational complexity    | Low                 | High (two systems + sync)     |
| Schema evolution          | One migration       | Two migrations + event schema |
| Failure modes             | One DB = one failure| Independent failure domains   |

### When CQRS Is Worth the Complexity

```
  Use CQRS when:
  ┌─────────────────────────────────────────────────────┐
  │ - Read/write ratio > 50:1                           │
  │ - Read and write schemas are fundamentally different│
  │ - Reads require denormalized/precomputed data       │
  │ - Read and write scaling needs diverge              │
  │ - You need different storage engines per path       │
  │   (e.g., PostgreSQL for writes, Elasticsearch for   │
  │    full-text search reads)                          │
  └─────────────────────────────────────────────────────┘

  Do NOT use CQRS when:
  ┌─────────────────────────────────────────────────────┐
  │ - Read/write ratio < 10:1                           │
  │ - Strong consistency is required for all reads      │
  │ - Team lacks experience with event-driven systems   │
  │ - System is early stage (premature optimization)    │
  │ - Read and write models are nearly identical        │
  └─────────────────────────────────────────────────────┘
```

### CQRS Event Flow Detail

```
  1. User places order
  2. Command handler validates and writes to Order DB
  3. OrderPlaced event emitted to Kafka

  ┌─────────────────────────────────────────────────────────┐
  │  OrderPlaced Event                                      │
  │  {                                                      │
  │    "event_id": "evt_abc123",                           │
  │    "type": "OrderPlaced",                              │
  │    "timestamp": "2024-01-15T10:30:00Z",                │
  │    "data": {                                           │
  │      "order_id": "ord_789",                            │
  │      "user_id": "usr_456",                             │
  │      "items": [...],                                   │
  │      "total": 149.99                                   │
  │    }                                                   │
  │  }                                                     │
  └─────────────────────────────────────────────────────────┘

  4. Event consumers update:
     - User's order history (DynamoDB)
     - Inventory counts (Redis)
     - Analytics aggregations (ClickHouse)
     - Search index (Elasticsearch)
  5. Each consumer can fail/retry independently
```

---

## 5. Connection Pool Tuning for Mixed Workloads

### The Problem

A single connection pool serving both reads and writes leads to:
- Long-running reads starving writes (or vice versa)
- Read replica connections mixed with primary connections
- No ability to prioritize writes during peak write periods

### Split Pool Architecture

```
  ┌─────────────────────────────────────────────────────┐
  │  Application Server                                 │
  │                                                     │
  │  ┌──────────────┐          ┌──────────────┐         │
  │  │ Write Pool   │          │ Read Pool    │         │
  │  │ max: 20      │          │ max: 80      │         │
  │  │ timeout: 5s  │          │ timeout: 30s │         │
  │  │ -> Primary   │          │ -> Replicas  │         │
  │  └──────┬───────┘          └──────┬───────┘         │
  └─────────┼──────────────────────────┼────────────────┘
            │                          │
       ┌────▼────┐          ┌──────────▼──────────┐
       │ Primary │          │   Load Balancer     │
       │  (1)    │          │   (read replicas)   │
       └─────────┘          │  ┌────┐ ┌────┐ ┌────┐
                            │  │ R1 │ │ R2 │ │ R3 │
                            │  └────┘ └────┘ └────┘
                            └─────────────────────┘
```

### Pool Sizing Guidelines

| Parameter              | Write Pool               | Read Pool                    |
|------------------------|-------------------------|------------------------------|
| Max connections        | num_cores * 2           | num_cores * 2 * num_replicas |
| Min idle connections   | num_cores               | num_cores * num_replicas     |
| Connection timeout     | 3-5 seconds             | 10-30 seconds                |
| Query timeout          | 5-30 seconds            | 30-120 seconds               |
| Eviction interval      | 60 seconds              | 60 seconds                   |
| Validation query       | SELECT 1                | SELECT 1                     |

### Priority Queuing

```
  Under contention, prioritize by operation type:

  Priority 1 (highest): Transaction commits
  Priority 2:           Single-row writes
  Priority 3:           Point reads (by PK)
  Priority 4:           Short range scans
  Priority 5 (lowest):  Analytical queries / reports

  Implementation: separate thread pools with weighted fair queuing
```

---

## 6. Practical Patterns by Workload Type

### AI/ML Training Pipeline (Write-Heavy)

```
  Pattern: Batch append with periodic compaction
  Write Model: Append-only parquet files to object storage
  Read Model:  Columnar table (Delta Lake / Iceberg) for training reads
  Ratio: 1:1 during training, 100:1 writes during data collection

  ┌──────────┐    ┌───────────┐    ┌─────────────┐
  │ Feature  │───>│ Append to │───>│ Compact &   │
  │ Pipeline │    │ Parquet   │    │ Register in │
  │          │    │ (S3)      │    │ Catalog     │
  └──────────┘    └───────────┘    └──────┬──────┘
                                          │
                                   ┌──────▼──────┐
                                   │ Training    │
                                   │ Job Reads   │
                                   │ (columnar   │
                                   │  scan)      │
                                   └─────────────┘
```

### Social Media Feed (Read-Heavy)

```
  Pattern: Fan-out on write + precomputed timeline
  Write Model: Single write to post table + fan-out to follower timelines
  Read Model:  Pre-materialized timeline per user (Redis sorted set)
  Ratio: 100:1 reads to writes

  Write: new_post -> post_table -> fan_out_to_N_followers -> update_timelines
  Read:  ZRANGEBYSCORE user:timeline:-inf +inf LIMIT 0 20  (< 1ms)
```

### Financial Transaction System (Balanced, Consistency-Critical)

```
  Pattern: Strong consistency, no eventual reads
  Write Model: Serializable transactions on primary
  Read Model:  Synchronous replicas (zero lag guarantee)
  Ratio: 5:1 reads to writes, but ALL must be strongly consistent

  NO read replicas with async replication (regulatory requirement)
  Scale reads via: caching with short TTL + request coalescing
```

---

## 7. Measuring and Addressing Imbalance Over Time

### Continuous Monitoring Framework

```
  ┌──────────────────────────────────────────────────────────┐
  │  Weekly Workload Report                                  │
  │                                                          │
  │  Read/Write Ratio Trend (12 weeks)                       │
  │                                                          │
  │  Ratio                                                   │
  │  100│           *                                        │
  │     │        *     *  *                                  │
  │   50│     *              *  *                             │
  │     │  *                       *  *  *                   │
  │   10│                                                    │
  │     └────────────────────────────────                    │
  │      W1  W2  W3  W4  W5  W6  W7  W8  W9 W10 W11 W12    │
  │                                                          │
  │  Trend: Ratio declining (more writes proportionally)     │
  │  Action: Evaluate write-optimization patterns            │
  └──────────────────────────────────────────────────────────┘
```

### Rebalancing Triggers

| Signal                                 | Indicates                     | Action                        |
|----------------------------------------|-------------------------------|-------------------------------|
| Read latency rising, writes stable     | Read capacity exhausted       | Add read replicas             |
| Write latency rising, reads stable     | Write bottleneck              | Vertical scale or shard       |
| Both rising                            | Fundamental capacity limit    | Architecture review (CQRS?)   |
| Read ratio increasing over time        | Product evolving to read-heavy| Pre-provision read replicas   |
| Write ratio increasing over time       | New write-heavy feature       | Evaluate LSM / append-only    |

---

## Interview Narrative

**When asked about read/write imbalance, structure your answer as follows:**

> "The first step is always measurement: I instrument the database proxy to track read/write QPS,
> byte throughput, and latency percentiles separately for reads and writes. This gives me the
> actual ratio and tells me which side is the bottleneck.
>
> For read-heavy workloads (the most common case), I start with the simplest effective pattern:
> asynchronous read replicas behind a load balancer. This scales read capacity linearly. The key
> design decision is the consistency model -- for most consumer-facing reads, eventual consistency
> with a 100ms lag is acceptable. For read-after-write scenarios, I use a causal consistency token:
> the write returns a log sequence number, and the read request carries that LSN. If the replica
> has not caught up, the read routes to the primary.
>
> For extreme read/write divergence (100:1+), I consider CQRS -- separate the write model
> (normalized, optimized for integrity) from the read model (denormalized, optimized for query
> patterns). The write model emits events via Kafka, and independent consumers maintain
> purpose-built read stores: Elasticsearch for search, Redis for hot lookups, ClickHouse for
> analytics. The tradeoff is operational complexity and eventual consistency, which is why I only
> recommend CQRS when the ratio exceeds 50:1 and the read/write schemas are fundamentally
> different.
>
> For write-heavy workloads, I look at three optimizations: batch writes to reduce per-row
> overhead (10-100x throughput improvement), write-behind buffering to absorb spikes, and
> LSM-tree storage engines (RocksDB, Cassandra) that convert random writes to sequential I/O.
>
> Connection pool architecture matters too: I always split into separate read and write pools
> with independent sizing and timeouts. This prevents long-running analytics queries from
> starving transaction commits."

**Follow-up traps to prepare for:**

- "How do you handle the event bus being a bottleneck in CQRS?" (Answer: Kafka partitioning
  by entity ID ensures ordering per entity; scale partitions independently; if Kafka is down,
  the write path still works -- reads become stale but available)
- "What if the read model gets out of sync?" (Answer: every event is idempotent and replayable;
  rebuild the read model from the event log; this is the primary advantage of event sourcing)
- "How do you decide between read replicas and caching?" (Answer: replicas for diverse queries,
  caching for hot-key point lookups; in practice you use both -- cache in front of replicas)
