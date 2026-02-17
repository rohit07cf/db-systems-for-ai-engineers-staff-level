# Redis Multi-Region Design — Staff/Principal Deep Dive

## 1. Multi-Region Architecture Patterns

### Pattern Overview

```
  +---------------------------------------------------------------+
  |                Multi-Region Redis Patterns                     |
  +---------------------------------------------------------------+
  |                                                                |
  |  Active-Passive          Active-Active           Active-Active |
  |  (Replication)           (Proxy/App layer)       (CRDTs)      |
  |                                                                |
  |  +------+  async  +------+ +------+  conflict +------+       |
  |  |Master|-------->|Repl  | |Master|  resolve  |Master|       |
  |  |US-E  |         |EU-W  | |US-E  |<--------->|EU-W  |       |
  |  +------+         +------+ +------+           +------+       |
  |                                                                |
  |  Simplest              Medium                  Most Complex    |
  |  Read scaling          Write everywhere        True multi-write|
  |  Writes: 1 region      App resolves conflicts  CRDTs resolve   |
  |  RPO: repl lag         RPO: varies             RPO: ~0 (CRDT)  |
  |  RTO: failover time    RTO: routing change     RTO: ~0         |
  +---------------------------------------------------------------+
```

---

## 2. Active-Passive: Cross-Region Replication

### Architecture

```
  US-EAST (Primary)                    EU-WEST (DR / Read Replica)
  +---------------------------+        +---------------------------+
  | Redis Cluster             |        | Redis Cluster             |
  | +-------+ +-------+      |        | +-------+ +-------+      |
  | |Master | |Master |      |  async | |Repl   | |Repl   |      |
  | | A     | | B     |      | ------>| | A'    | | B'    |      |
  | +---+---+ +---+---+      |  repl  | +-------+ +-------+      |
  |     |         |           |  ~80ms | +-------+ +-------+      |
  | +---+---+ +---+---+      | ------>| |Repl   | |Repl   |      |
  | |Repl  | |Repl  |      |        | | C'    | | D'    |      |
  | | A1   | | B1   |      |        | +-------+ +-------+      |
  | +-------+ +-------+      |        +---------------------------+
  +---------------------------+        Serves reads only (READONLY)
  Serves all writes + reads           Failover target if primary down
```

### Cross-Region Replication Lag

```
  Latency Components:
  +--------------------------------------------------------+
  | Component                   | US-East to EU-West       |
  |-----------------------------|--------------------------|
  | Network RTT (one way)      | 35-50ms                  |
  | TCP retransmission (rare)  | +200ms per retransmit    |
  | Replication buffer flush   | 1-5ms                    |
  | Replica apply time         | 1-2ms                    |
  |                             |                          |
  | TOTAL typical lag          | 40-60ms                  |
  | TOTAL p99 lag              | 80-150ms                 |
  | TOTAL during congestion    | 200-500ms                |
  +--------------------------------------------------------+

  Impact on RPO:
    If primary region fails with 60ms replication lag:
    ~60ms worth of writes are lost
    At 50K writes/sec: ~3,000 writes lost per failover

  Impact on read consistency:
    EU-West readers see data 40-60ms stale
    For session data: usually acceptable
    For financial data: NOT acceptable
```

### Failover Procedure (Active-Passive)

```
  Step 1: Detect primary region failure
          - External health checker (not Redis itself)
          - Cross-region probe: ping primary every 5s
          - Declare failure after 3 consecutive misses (15s)

  Step 2: Promote secondary region
          - CLUSTER FAILOVER FORCE on each replica in EU-WEST
          - Update DNS/load balancer to route writes to EU-WEST
          - Or: update service mesh routing rules

  Step 3: Application cutover
          - Invalidate client connection pools
          - Re-resolve DNS (TTL must be low: 30-60s)
          - Accept brief unavailability during DNS propagation

  Step 4: After primary recovers
          - DO NOT auto-failback (risk of split-brain)
          - Manual process:
            a. Set recovered US-EAST as replica of EU-WEST
            b. Wait for full sync
            c. Plan maintenance window for failback
            d. Failback during low-traffic window

  Total RTO: 15s detection + 5s promotion + 30-60s DNS = 50-80s
```

---

## 3. Active-Active with CRDTs (Redis Enterprise)

### CRDT Data Types

```
  Conflict-Free Replicated Data Types in Redis Enterprise:

  +----------------------------------------------------------+
  | Redis Type   | CRDT Strategy         | Conflict Resolution|
  |--------------|-----------------------|--------------------|
  | String       | Last-Writer-Wins (LWW)| Wall-clock timestamp|
  | Counter      | PN-Counter (positive- | Merge: sum of all  |
  |              | negative counter)     | increments/decrements|
  | Set          | OR-Set (Observed-     | Add wins over remove|
  |              | Remove Set)           | in concurrent ops  |
  | Sorted Set   | LWW per element +     | Higher score wins  |
  |              | OR-Set for membership | for concurrent adds|
  | Hash         | LWW per field         | Per-field timestamp|
  | List         | Not CRDT-safe         | Avoid in active-   |
  |              |                       | active deployments |
  +----------------------------------------------------------+
```

### Active-Active Architecture

```
  Region US-EAST                        Region EU-WEST
  +---------------------------+         +---------------------------+
  | CRDB Instance A           |         | CRDB Instance B           |
  | +-------+                 |  CRDT   | +-------+                 |
  | |Master | <---sync layer--|-------->| |Master |                 |
  | | (R/W) |  (bi-directional|  merge  | | (R/W) |                 |
  | +---+---+  replication)   |         | +---+---+                 |
  |     |                     |         |     |                     |
  | +---+---+                 |         | +---+---+                 |
  | |Replica|                 |         | |Replica|                 |
  | +-------+                 |         | +-------+                 |
  +---------------------------+         +---------------------------+
         |                                       |
         v                                       v
  Local clients write with              Local clients write with
  sub-ms latency (no cross-             sub-ms latency (no cross-
  region RTT in write path)             region RTT in write path)

  Sync layer merges changes asynchronously using CRDT rules
  Convergence guaranteed by mathematical properties of CRDTs
```

### CRDT Conflict Resolution Examples

```
  Example 1: Counter (PN-Counter)
  -----------------------------------------------
  t=0: counter = 100 (both regions)
  t=1: US-EAST: INCR counter  (local: 101, delta: +1)
  t=1: EU-WEST: INCR counter  (local: 101, delta: +1)
  t=2: Sync merges: 100 + 1 + 1 = 102 (CORRECT, no conflict)

  Example 2: String (LWW)
  -----------------------------------------------
  t=0: key = "v1" (both regions)
  t=1.000: US-EAST: SET key "v2" (timestamp: 1.000)
  t=1.001: EU-WEST: SET key "v3" (timestamp: 1.001)
  t=2: Sync: EU-WEST wins (higher timestamp) -> key = "v3"

  CRITICAL: LWW requires synchronized clocks (NTP/PTP).
  Clock skew > 1ms can cause unexpected winners.
  Redis Enterprise uses vector clocks + wall clock hybrid.

  Example 3: Set (OR-Set)
  -----------------------------------------------
  t=0: myset = {a, b, c} (both regions)
  t=1: US-EAST: SADD myset d       (add d)
  t=1: EU-WEST: SREM myset b       (remove b)
  t=2: Sync merge: {a, c, d}       (add wins for d, remove wins for b)

  t=1: US-EAST: SADD myset x       (add x)
  t=1: EU-WEST: SREM myset x       (remove x -- concurrent with add)
  t=2: Sync merge: {a, c, d, x}    (ADD WINS over concurrent REMOVE)
```

### CRDT Limitations and Tradeoffs

| Aspect                 | Impact                                    | Mitigation                      |
|------------------------|-------------------------------------------|---------------------------------|
| LWW for strings        | Last writer wins may not match app intent  | Use counters/sets where possible|
| Clock synchronization  | NTP skew can cause unexpected resolution   | Use PTP, monitor clock drift    |
| Memory overhead         | CRDT metadata per key (~40-100 bytes)     | Budget additional 10-30% memory |
| Tombstone accumulation | Deleted keys leave metadata temporarily    | GC policy tuning                |
| Lists not supported    | Cannot merge concurrent list operations    | Use sorted sets with timestamps |
| Large objects          | Sync bandwidth proportional to change rate | Decompose into smaller keys     |
| Debugging conflicts     | Hard to trace which write "won"           | Enable CRDT conflict logging    |

---

## 4. Cross-Region Latency Impact on Design

### Latency Budget Analysis

```
  Typical API Request with Redis (single region):
  +--------------------------------------------------+
  | Operation          | Latency    | Cumulative     |
  |--------------------+------------|----------------|
  | Load balancer      | 1ms        | 1ms            |
  | App server receive | 0.5ms      | 1.5ms          |
  | Redis GET (local)  | 0.3ms      | 1.8ms          |
  | Business logic     | 2ms        | 3.8ms          |
  | Redis SET (local)  | 0.3ms      | 4.1ms          |
  | Response           | 0.5ms      | 4.6ms          |
  +--------------------------------------------------+

  Same request with cross-region Redis (naive approach):
  +--------------------------------------------------+
  | Operation          | Latency    | Cumulative     |
  |--------------------+------------|----------------|
  | Load balancer      | 1ms        | 1ms            |
  | App server receive | 0.5ms      | 1.5ms          |
  | Redis GET (cross)  | 80ms       | 81.5ms  !!!    |
  | Business logic     | 2ms        | 83.5ms         |
  | Redis SET (cross)  | 80ms       | 163.5ms !!!    |
  | Response           | 0.5ms      | 164ms          |
  +--------------------------------------------------+

  35x SLOWER — unacceptable for user-facing requests
```

### Design Principle: Region-Local Reads, Async Cross-Region Writes

```
  Correct Multi-Region Pattern:
  +--------------------------------------------------+
  | Request in US-EAST:                               |
  |   1. Read from LOCAL Redis (0.3ms)                |
  |   2. Write to LOCAL Redis (0.3ms)                 |
  |   3. Async replication to EU-WEST (background)    |
  |   Total user-facing: 4.6ms (same as single-region)|
  +--------------------------------------------------+

  Eventual Consistency Window:
  |<--- 40-80ms replication lag --->|
  US-EAST write ===================> EU-WEST visible
```

---

## 5. Global Session Store Design

### Architecture for Multi-Region Sessions

```
  User in US-EAST:
  +--------+     +----------+     +------------------+
  | Browser|---->| US-EAST  |---->| Redis US-EAST    |
  |        |     | App      |     | session:abc=     |
  |        |     | Server   |     | {user:1001,...}  |
  +--------+     +----------+     +--+---------------+
                                     |
                                     | async replication
                                     v
                                  +------------------+
                                  | Redis EU-WEST    |
                                  | session:abc=     |
                                  | {user:1001,...}  |
                                  +------------------+

  User roams to EU-WEST:
  +--------+     +----------+     +------------------+
  | Browser|---->| EU-WEST  |---->| Redis EU-WEST    |
  |        |     | App      |     | session:abc=     |
  |        |     | Server   |     | {user:1001,...}  |
  +--------+     +----------+     +------------------+
                                  Session found locally!
                                  (replicated earlier)
```

### Session Store Design Decisions

| Decision                | Option A                       | Option B                         |
|-------------------------|--------------------------------|----------------------------------|
| Session storage format  | Single JSON string             | Hash with per-field access       |
| TTL management          | Key-level TTL (simple)         | Sliding window TTL (custom Lua)  |
| Region affinity         | Sticky sessions (simple)       | Any-region read (needs replication)|
| Session size            | Small (<1KB, fast repl)        | Large (>10KB, repl bandwidth)    |
| Conflict handling       | LWW (last write wins)          | Merge (CRDT hash per field)      |
| Eviction on OOM         | volatile-lru (TTL keys only)   | allkeys-lru (risky for sessions) |

### Sliding Window Session TTL Pattern

```lua
  -- Lua script: Extend session TTL on every access
  -- Atomic read + TTL refresh

  local session = redis.call('HGETALL', KEYS[1])
  if #session > 0 then
      redis.call('EXPIRE', KEYS[1], ARGV[1])  -- refresh TTL
      return session
  else
      return nil  -- session expired or doesn't exist
  end

  -- Usage: EVALSHA <sha> 1 session:abc 3600
  -- Refreshes to 1 hour on every access
```

---

## 6. Conflict Resolution Strategies (Without CRDTs)

### Application-Level Conflict Resolution

```
  Pattern: Version Vector with Last-Writer-Wins Fallback

  Key format: {entity}:{id}
  Value: JSON with embedded version metadata

  {
    "data": { "name": "Alice", "status": "active" },
    "version": {
      "us-east": 5,
      "eu-west": 3
    },
    "last_modified": 1706140800000,
    "origin_region": "us-east"
  }

  Merge algorithm:
  1. Compare version vectors
  2. If one dominates (all components >=): take the dominant
  3. If concurrent (incomparable vectors): conflict!
     a. For idempotent data: merge fields independently
     b. For non-idempotent: LWW using last_modified
     c. For critical data: queue for human review
```

### Conflict Resolution Decision Matrix

```
  +----------------------------------------------------------+
  | Data Type          | Strategy           | Rationale        |
  |--------------------|--------------------|------------------|
  | User profile       | LWW per field      | Fields are       |
  |                    |                    | independent      |
  | Shopping cart      | Union merge        | Add always wins  |
  | View counter       | Sum of increments  | Commutative      |
  | Inventory count    | DO NOT multi-region| Requires strong  |
  |                    | (use single master)| consistency      |
  | Session data       | LWW (latest write) | User in one      |
  |                    |                    | region at a time |
  | Rate limit counter | Sum of increments  | Over-count is    |
  |                    |                    | safer than under |
  | Feature flags      | LWW with epoch     | Config changes   |
  |                    |                    | are sequential   |
  +----------------------------------------------------------+
```

---

## 7. Architecture Patterns for FAANG Scale

### Pattern 1: Regional Cache + Global Source of Truth

```
  +---------------------------+    +---------------------------+
  |  US-EAST                  |    |  EU-WEST                  |
  |  +-------+   +--------+  |    |  +-------+   +--------+  |
  |  | App   |-->| Redis  |  |    |  | App   |-->| Redis  |  |
  |  | Server|   | Cache  |  |    |  | Server|   | Cache  |  |
  |  +---+---+   +--------+  |    |  +---+---+   +--------+  |
  |      |                    |    |      |                    |
  |      | cache miss         |    |      | cache miss         |
  |      v                    |    |      v                    |
  |  +--------+              |    |  +--------+              |
  |  | Global |              |    |  | Global |              |
  |  | DB     |<============>|====|  | DB     |              |
  |  | (Spanner/            |    |  | (read  |              |
  |  |  CockroachDB)         |    |  |  replica)|            |
  |  +--------+              |    |  +--------+              |
  +---------------------------+    +---------------------------+

  - Redis is a regional cache (not source of truth)
  - Global DB handles consistency (Spanner/CockroachDB)
  - Redis TTL-based invalidation (eventual consistency OK)
  - Cache-aside pattern: read from Redis, miss goes to DB
  - Write-through: write to DB, invalidate Redis cache
```

### Pattern 2: Hub-and-Spoke Replication

```
  Hub Region (US-EAST) — accepts all writes
       |
       +-------> Spoke: EU-WEST (read-only)
       |
       +-------> Spoke: AP-SOUTHEAST (read-only)
       |
       +-------> Spoke: US-WEST (read-only)

  Properties:
  - Simple mental model (one writer)
  - No conflicts possible
  - Write latency for non-hub users is high (cross-region RTT)
  - Hub failure = global write outage until failover

  Optimization: Route writes through regional app servers
  to a message queue, which drains to the hub Redis:

  EU-WEST App --> Kafka (EU-WEST) --> Consumer (US-EAST) --> Redis (US-EAST)
                                      ~100ms async
```

### Pattern 3: Independent Regional Clusters + Async Sync

```
  US-EAST Redis Cluster          EU-WEST Redis Cluster
  +--------------------+         +--------------------+
  | Independent cluster|         | Independent cluster|
  | Full R/W locally   |         | Full R/W locally   |
  +--------+-----------+         +--------+-----------+
           |                              |
           +----------+-------------------+
                      |
               +------v------+
               | Sync Service|   Custom application
               | (Kafka/     |   - Captures changes via keyspace notifications
               |  Change     |   - Publishes to cross-region Kafka
               |  Stream)    |   - Consumer applies changes with conflict check
               +-------------+

  Sync Service Responsibilities:
  1. Subscribe to __keyevent@*__ notifications on source
  2. For each change, DUMP key -> serialize -> publish to Kafka
  3. Consumer in target region: RESTORE key -> apply
  4. Conflict detection: compare version vectors before apply
  5. Dead letter queue for unresolvable conflicts
```

---

## 8. Operational Considerations

### Monitoring Multi-Region Redis

```
  Per-Region Metrics:
  +-------------------------------------------------------------+
  | Metric                        | Alert Condition              |
  |-------------------------------|------------------------------|
  | cross_region_repl_lag         | > 200ms (WARNING)            |
  |                               | > 1s (CRITICAL)              |
  | cross_region_repl_link_status | != "up" for > 30s (CRITICAL) |
  | crdt_conflict_rate            | > 100/sec (WARNING)          |
  | crdt_gc_pending_tombstones    | > 10M (WARNING)              |
  | region_failover_readiness     | replica_lag > 5s (CRITICAL)  |
  | clock_drift_between_regions   | > 10ms (WARNING for LWW)     |
  +-------------------------------------------------------------+

  Cross-Region Dashboard:
  +---------------------------+     +---------------------------+
  |  US-EAST                  |     |  EU-WEST                  |
  |  ops/sec: 150,000         |     |  ops/sec: 120,000         |
  |  memory: 45GB/64GB        |     |  memory: 43GB/64GB        |
  |  repl lag TO eu: 55ms     |     |  repl lag TO us: 52ms     |
  |  conflicts/sec: 12        |     |  conflicts/sec: 12        |
  |  cluster_state: ok        |     |  cluster_state: ok        |
  +---------------------------+     +---------------------------+
```

### Cost Considerations at FAANG Scale

```
  Cross-Region Data Transfer Cost (AWS example):
  +---------------------------------------------+
  | Replication Rate | Monthly Transfer | Cost   |
  |------------------|------------------|--------|
  | 1 MB/s           | 2.5 TB           | ~$225  |
  | 10 MB/s          | 25 TB            | ~$2,250|
  | 100 MB/s         | 250 TB           | ~$22,500|
  +---------------------------------------------+

  Optimization: Delta compression for replication
  - Only send changed bytes, not full key values
  - Redis Enterprise: built-in compression for CRDB sync
  - Custom sync: use RESP3 diff protocol or custom binary

  Instance Cost (approximate):
  +---------------------------------------------+
  | Setup                    | Monthly Cost      |
  |--------------------------|-------------------|
  | 3-node cluster, 1 region | ~$3,000           |
  | 6-node cluster, 2 regions| ~$8,000           |
  | Active-active CRDB, 2 reg| ~$15,000+         |
  | Active-active, 4 regions | ~$35,000+         |
  +---------------------------------------------+
```

---

## Interview Narrative

**When asked "How would you design a multi-region Redis architecture for a global application?":**

> "The multi-region Redis design depends fundamentally on the consistency requirements of the data being stored. I would categorize the data into three tiers and apply different strategies to each.
>
> For ephemeral cache data like rendered pages or computed results, I would use independent regional Redis clusters with no replication. Each region caches from its local database read replica. Cache misses are acceptable and simply refill from the database. This is the simplest, cheapest, and most resilient pattern because there are no cross-region dependencies.
>
> For session data and user preferences where availability matters more than strict consistency, I would use active-passive replication with asynchronous cross-region replication. The primary region handles writes, and secondary regions serve reads from replicas. The replication lag of 40-80ms is acceptable for sessions because a user is typically in one region at a time. If the primary fails, we promote the secondary with DNS failover, accepting a small data loss window equal to the replication lag.
>
> For data requiring true multi-region writes like global rate limiters, shopping carts, or collaborative features, I would use Redis Enterprise with CRDT-based active-active replication. CRDTs guarantee convergence without coordination: counters use PN-Counters, sets use OR-Sets where add wins over concurrent remove, and strings use Last-Writer-Wins with synchronized clocks. The key constraint is that lists are not CRDT-safe, so we avoid them in multi-write scenarios.
>
> The critical architectural principle is keeping the write path region-local. Cross-region latency of 40-80ms in the write path is devastating for user-facing requests. With active-active CRDTs, writes are local with sub-millisecond latency, and cross-region sync happens asynchronously. The tradeoff is eventual consistency with a window equal to the replication lag.
>
> At FAANG scale, I would also consider the cost dimension. Cross-region data transfer at 10MB/s costs roughly $2,000-$3,000 per month on major cloud providers. For high-throughput workloads, delta compression and selective replication of only critical keys can significantly reduce this. I would instrument replication lag, conflict rates, and clock drift between regions as first-class SLOs for the system."
