# Hot Key Mitigation: Detecting and Neutralizing Partition Hotspots

## Why This Matters at Staff+ Level

A single hot key can bring down an entire shard -- and with it, every other entity co-located on
that shard. Hot keys are the most common cause of cascading failures in sharded databases. At the
staff/principal level you must design systems that detect hotspots before they become outages and
implement mitigation patterns that do not compromise correctness.

---

## 1. Anatomy of a Hot Key

```
Normal Distribution:                Hot Key Distribution:

  Requests                            Requests
  per key                             per key
    │                                   │
  5k│ ██ ██ ██ ██ ██ ██               5k│ ██ ██ ██ ██ ██ ██
    │ ██ ██ ██ ██ ██ ██                 │ ██ ██ ██ ██ ██ ██
    │ ██ ██ ██ ██ ██ ██                 │ ██ ██ ██ ██ ██ ██
    │ ██ ██ ██ ██ ██ ██             50k│                    ██████
    └─────────────────                  └──────────────────────────
      k1 k2 k3 k4 k5 k6                 k1 k2 k3 k4 k5 k6  k_hot

  One key consuming 10x the resources of any other key.
  That key's shard is at 95% CPU while others sit at 15%.
```

### Common Hot Key Sources

| Source                        | Example                                         |
|-------------------------------|--------------------------------------------------|
| Viral content                 | A tweet going viral (tweet_id = hot key)         |
| Celebrity accounts            | Follower list for @taylorswift                   |
| Global counters               | site_visits_total, likes_count                   |
| Temporal convergence          | "today's date" as partition key in time-series   |
| Default/null values           | unassigned_queue, default_tenant                 |
| Bot/scraper traffic           | Repeated reads on product catalog pages          |
| Feature flag lookups          | Single row read by every request                 |

---

## 2. Detection Strategies

### Real-Time Detection Architecture

```
  ┌──────────┐     ┌──────────────┐     ┌──────────────┐
  │ Database  │────>│ Access Log   │────>│ Stream       │
  │ Proxy     │     │ / Metrics    │     │ Processor    │
  └──────────┘     └──────────────┘     │ (Flink/KCL)  │
                                         └──────┬───────┘
                                                │
                                         ┌──────▼───────┐
                                         │ Hot Key      │
                                         │ Detector     │
                                         │              │
                                         │ Algorithm:   │
                                         │ Count-Min    │
                                         │ Sketch +     │
                                         │ Top-K heap   │
                                         └──────┬───────┘
                                                │
                                    ┌───────────┼───────────┐
                                    │           │           │
                              ┌─────▼────┐ ┌───▼────┐ ┌───▼──────┐
                              │ Alert    │ │ Auto   │ │ Dashboard│
                              │ (PagerD) │ │ Mitig. │ │ (Grafana)│
                              └──────────┘ └────────┘ └──────────┘
```

### Count-Min Sketch for Frequency Estimation

A probabilistic data structure that answers "how often has key X been accessed?" using
sub-linear memory.

```
  Hash functions: h1, h2, h3
  Counters: 3 rows x W columns

              0    1    2    3    4    5    6    7
  h1(key):  [ 0 ][ 3 ][ 0 ][ 7 ][ 0 ][ 2 ][ 0 ][ 0 ]
  h2(key):  [ 0 ][ 0 ][ 5 ][ 0 ][ 0 ][ 7 ][ 0 ][ 1 ]
  h3(key):  [ 2 ][ 0 ][ 0 ][ 0 ][ 7 ][ 0 ][ 3 ][ 0 ]

  Estimated frequency of key X = min(h1[X], h2[X], h3[X])
  Memory: O(W * num_hashes) -- typically a few KB for millions of keys
```

### Monitoring Metrics for Hot Key Detection

| Metric                             | Threshold              | Action                    |
|------------------------------------|------------------------|---------------------------|
| Per-key QPS (sampled)              | > 10x median key QPS   | Flag as candidate hot key |
| Per-shard CPU utilization          | > 80% while others <30%| Investigate shard's keys  |
| Per-shard P99 latency              | > 3x cluster P99       | Immediate investigation   |
| Key access distribution (Gini)    | > 0.8                  | Structural skew warning   |
| Per-partition byte throughput      | > 2x mean partition    | Capacity planning alert   |

---

## 3. Key-Prefix Salting

Spread a single hot key across N sub-keys by appending a random or deterministic salt.

### Write Path

```
Original:  key = "celebrity_user_123"
Salted:    key = "celebrity_user_123:salt_0"
           key = "celebrity_user_123:salt_1"
           key = "celebrity_user_123:salt_2"
           ...
           key = "celebrity_user_123:salt_N"

Write:  salt = random(0, N-1)
        write(key + ":salt_" + salt, value)
```

### Read Path

```
Read (aggregation required):
  results = []
  for i in range(N):
      results.append(read(key + ":salt_" + i))
  return aggregate(results)

  Example: counter value = sum(results)
  Example: latest value = max(results, key=timestamp)
```

### Tradeoff Analysis

| N (salt range) | Write Distribution | Read Amplification | Consistency Complexity |
|:--------------:|:-----------------:|:------------------:|:----------------------:|
| 1              | None (baseline)   | 1x                 | Simple                 |
| 4              | 4x improvement    | 4x                 | Moderate               |
| 16             | 16x improvement   | 16x                | High                   |
| 256            | Excellent         | 256x (expensive)   | Very High              |

**Guideline:** Use N = number of shards for maximum distribution, but be aware this turns every
point read into a scatter-gather query.

---

## 4. Local Aggregation Before Writes

Instead of writing every increment directly to the database, buffer writes locally and flush
periodically.

```
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Server 1 │  │ Server 2 │  │ Server 3 │
  │          │  │          │  │          │
  │ Local:   │  │ Local:   │  │ Local:   │
  │ count=47 │  │ count=31 │  │ count=55 │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │              │              │
       │   flush every 1 second      │
       │              │              │
       └──────────────┼──────────────┘
                      │
                ┌─────▼──────┐
                │  Database   │
                │  INCREMENT  │
                │  BY 133     │  (instead of 133 individual writes)
                └────────────┘
```

### Implementation Considerations

| Aspect            | Detail                                                     |
|-------------------|------------------------------------------------------------|
| Flush interval    | 1-5 seconds (tradeoff: freshness vs write reduction)       |
| Server crash      | Lose up to 1 flush interval of increments (acceptable?)    |
| Accuracy          | Eventually consistent; lag = flush interval                |
| Write reduction   | 100-1000x fewer database writes for high-frequency counters|
| Memory overhead   | One counter per hot key per server (minimal)               |

### When To Use

- Like counts, view counts, impression counters
- Rate limiting counters (approximate is acceptable)
- Real-time analytics (seconds of lag is fine)

### When NOT To Use

- Financial balances (every write must be durable)
- Inventory counts (overselling risk)
- Unique constraint checks (requires exact state)

---

## 5. Read Replicas for Hot Reads

When a key is hot for reads (not writes), direct traffic to multiple read replicas.

```
                    ┌──────────────┐
                    │   Primary    │
                    │   (writes)   │
                    └──────┬───────┘
                           │ replication
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼────┐ ┌────▼─────┐ ┌───▼──────┐
        │ Replica 1│ │ Replica 2│ │ Replica 3│
        │ (reads)  │ │ (reads)  │ │ (reads)  │
        └──────────┘ └──────────┘ └──────────┘

  Hot key reads distributed across 3 replicas:
  Each replica handles ~33% of read traffic for the hot key.
```

### Consistency Considerations

| Replication Type   | Staleness    | Read Throughput | Use Case                |
|--------------------|-------------|-----------------|-------------------------|
| Synchronous        | 0 (strong)  | Lower (latency) | Financial reads         |
| Semi-synchronous   | ~ms         | Moderate        | User profiles           |
| Asynchronous       | ~100ms-seconds | Highest      | Social media feeds      |

### Smart Routing

```
if key in known_hot_keys:
    if query.requires_strong_consistency:
        route_to(primary)
    else:
        route_to(random_replica)
else:
    route_to(shard_for_key(key))
```

---

## 6. Request Coalescing

When many concurrent requests ask for the same key, serve them all from a single database read.

```
  T0: Request A asks for key "viral_post_123"  ─┐
  T0: Request B asks for key "viral_post_123"   ├─ Coalesced into ONE DB read
  T0: Request C asks for key "viral_post_123"  ─┘
  T1: DB returns result
  T1: All three requests receive the same result

  Without coalescing:  3 DB reads
  With coalescing:     1 DB read
```

### Implementation: SingleFlight Pattern

```go
// Go's singleflight (conceptual)
var group singleflight.Group

func GetPost(postID string) (*Post, error) {
    result, err, _ := group.Do(postID, func() (interface{}, error) {
        // Only ONE goroutine executes this, even if 1000 call GetPost concurrently
        return db.Query("SELECT * FROM posts WHERE id = ?", postID)
    })
    return result.(*Post), err
}
```

### Tradeoff Table

| Aspect                    | Without Coalescing     | With Coalescing         |
|---------------------------|------------------------|-------------------------|
| DB load for hot key       | O(concurrent requests) | O(1)                    |
| Latency for first request | Baseline               | Baseline                |
| Latency for queued reqs   | Baseline               | Slightly higher (wait)  |
| Error blast radius        | Per-request             | All coalesced reqs fail |
| Cache interaction         | Each request fills cache| One request fills cache  |

---

## 7. Cache-Aside for Hot Data

Place a caching layer that absorbs hot key reads before they reach the database.

```
  ┌──────────┐     Cache     ┌──────────┐
  │  Client   │───── HIT ───>│  Cache    │
  │           │               │ (Redis/   │
  │           │               │  Memcached│
  │           │               └─────┬────┘
  │           │                     │ MISS
  │           │               ┌─────▼────┐
  │           │<── response──│ Database  │
  │           │               │           │
  └──────────┘               └──────────┘

  Cache-aside flow:
  1. Check cache for key
  2. If HIT: return cached value
  3. If MISS: read from DB, store in cache, return
  4. On write: invalidate cache entry (or write-through)
```

### Hot Key Specific Cache Strategies

| Strategy              | Description                                    | Risk                     |
|-----------------------|------------------------------------------------|--------------------------|
| Write-through         | Update cache on every write                    | Write latency increases  |
| Write-behind          | Queue cache updates, async flush               | Stale reads on crash     |
| TTL-based expiry      | Cache for N seconds, then refresh              | Thundering herd on expiry|
| Jittered TTL          | TTL + random(0, jitter) to stagger expiry      | Reduces herd effect      |
| Negative caching      | Cache "key not found" results                  | Must invalidate on create|
| Cache warming         | Pre-populate cache for predicted hot keys      | Memory waste if wrong    |

### Thundering Herd Mitigation

```
Problem: TTL expires -> 10,000 concurrent requests all miss -> 10,000 DB reads

Solution 1: Singleflight + cache
  - Only one request on miss goes to DB
  - Others wait for the result

Solution 2: Early refresh
  - Refresh cache at 80% of TTL (background)
  - Key never actually expires under load

Solution 3: Lock-based refresh
  - First request on miss acquires a lock
  - Others receive stale data (if available) or wait
  - Lock holder refreshes cache
```

---

## 8. DynamoDB Adaptive Capacity

DynamoDB's approach to hot partitions is a reference architecture for managed databases.

```
  Provisioned: 10,000 RCU across 10 partitions (1,000 RCU each)

  Before Adaptive Capacity:
    Partition 1: 3,000 RCU demand -> THROTTLED at 1,000
    Partitions 2-10: 200 RCU demand each

  After Adaptive Capacity:
    Partition 1: 3,000 RCU demand -> SERVED (borrows from idle partitions)
    Partitions 2-10: 200 RCU demand each
    Total: 3,000 + 1,800 = 4,800 < 10,000 provisioned -> no throttling

  ┌────────────────────────────────────────────┐
  │  Table: 10,000 RCU provisioned             │
  │                                            │
  │  P1 [████████████████████] 3,000 (hot)     │
  │  P2 [████]                   200           │
  │  P3 [████]                   200           │
  │  P4 [████]                   200           │
  │  ...                                       │
  │  Unused capacity redistributed to P1       │
  └────────────────────────────────────────────┘
```

### Key DynamoDB Features for Hot Keys

| Feature                | Behavior                                              |
|------------------------|-------------------------------------------------------|
| Adaptive capacity      | Unused RCU/WCU from cold partitions redistributed     |
| Burst capacity         | 300 seconds of unused capacity available for spikes   |
| DAX (in-memory cache)  | Microsecond reads for hot items, integrated cache     |
| Auto-split             | Hot partitions split automatically                    |
| On-demand mode         | No capacity planning; pay per request                 |

---

## 9. Monitoring and Alerting for Key Distribution Skew

### Key Metrics Dashboard

```
  ┌─────────────────────────────────────────────────────┐
  │  SHARD HEALTH DASHBOARD                             │
  │                                                     │
  │  Shard Load Distribution (QPS)                      │
  │  ┌─────────────────────────────────────┐            │
  │  │ S0  ████████████████  12,400        │            │
  │  │ S1  ██████            4,200         │            │
  │  │ S2  █████             3,800         │            │
  │  │ S3  █████████████████████ 48,200 !! │ <- ALERT   │
  │  │ S4  ██████            4,100         │            │
  │  └─────────────────────────────────────┘            │
  │                                                     │
  │  Top 10 Keys by QPS (sampled)                       │
  │  1. viral_post_928371     41,000 QPS                │
  │  2. trending_hashtag_42    3,200 QPS                │
  │  3. user_profile_celeb     2,800 QPS                │
  │                                                     │
  │  Distribution Gini Coefficient: 0.87 (CRITICAL)     │
  │  CV (Coefficient of Variation): 2.4                 │
  └─────────────────────────────────────────────────────┘
```

### Alerting Rules

```yaml
alerts:
  - name: shard_load_skew
    condition: max(shard_qps) / avg(shard_qps) > 5
    severity: warning
    action: page_oncall

  - name: hot_key_detected
    condition: single_key_qps > 10000
    severity: critical
    action: auto_enable_caching + page_oncall

  - name: partition_cpu_imbalance
    condition: max(partition_cpu) > 80 AND min(partition_cpu) < 20
    severity: warning
    action: investigate_key_distribution

  - name: throttle_rate_spike
    condition: throttled_requests / total_requests > 0.01
    severity: critical
    action: enable_adaptive_capacity + page_oncall
```

---

## 10. Mitigation Decision Tree

```
                     Hot Key Detected
                           │
                  ┌────────▼────────┐
                  │ Read or Write   │
                  │ hotspot?        │
                  └───┬─────────┬───┘
                      │         │
                  Read Hot   Write Hot
                      │         │
               ┌──────▼──┐  ┌──▼────────┐
               │ Add read │  │ Can batch │──Yes──> Local aggregation
               │ replicas │  │ / buffer? │         (flush every N sec)
               └──┬───────┘  └──┬────────┘
                  │              │ No
            ┌─────▼──────┐  ┌───▼───────┐
            │ Add cache  │  │ Salt the  │
            │ (Redis/DAX)│  │ key with  │
            └──┬─────────┘  │ N prefixes│
               │            └───────────┘
         ┌─────▼──────┐
         │ Enable     │
         │ request    │
         │ coalescing │
         └────────────┘
```

---

## Interview Narrative

**When asked about hot key problems, structure your answer as follows:**

> "Hot keys are the most common cause of cascading failures in sharded systems, so I address them
> at three layers: detection, mitigation, and prevention.
>
> For **detection**, I instrument the database proxy layer to sample key access frequencies using a
> Count-Min Sketch. This gives me approximate top-K keys in real-time with minimal memory overhead.
> I also monitor per-shard CPU and latency skew -- if one shard's P99 latency is 3x the cluster
> average, that is a strong signal for a hot key, even before I identify the specific key.
>
> For **mitigation**, the strategy depends on whether the hotspot is read-heavy or write-heavy.
> For read hotspots, I layer three defenses: first, a cache-aside pattern with jittered TTLs to
> avoid thundering herd; second, request coalescing (singleflight pattern) so that concurrent
> requests for the same key result in one database read; third, read replicas with smart routing
> that sends hot-key reads to replicas while other reads go to the primary.
>
> For write hotspots -- like a viral post's like counter -- I use local aggregation: each
> application server buffers increments in memory and flushes to the database every 1-2 seconds.
> This converts 50,000 individual writes per second into 50 batched increments. If the counter
> must be distributed (no single aggregation point), I use key-prefix salting with N = number of
> shards, so writes spread evenly. Reads then scatter-gather across all N salted keys and sum
> the results.
>
> For **prevention**, the best defense is shard key design. If I know certain entities will be hot
> (large tenants, global resources), I co-design the shard key to spread their data. For example,
> a composite key of (entity_id, time_bucket) ensures that even the hottest entity's writes
> distribute across time-bucket partitions.
>
> The key tradeoff across all these techniques is read amplification vs write distribution.
> Salting a key by N reduces write hotspot severity by N but increases read cost by N. The right
> N depends on the read/write ratio of the specific key."

**Follow-up traps to prepare for:**

- "What if the hot key changes unpredictably (e.g., viral content)?" (Answer: automatic detection
  + automatic cache promotion; DynamoDB's adaptive capacity is the reference pattern)
- "How does key salting interact with transactions?" (Answer: each salted sub-key is independent;
  transactions across sub-keys require distributed coordination, which defeats the purpose -- so
  use salting only for commutative operations like counters)
- "What is the memory cost of local aggregation at scale?" (Answer: one counter per hot key per
  server; with 100 hot keys and 1000 servers, that is 100K counters -- trivial memory cost)
