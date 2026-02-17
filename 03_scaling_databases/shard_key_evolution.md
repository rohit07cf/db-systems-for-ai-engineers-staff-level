# Shard Key Evolution: From Naive Hashing to Production-Grade Partitioning

## Why This Matters at Staff+ Level

Shard key selection is the single most consequential schema decision in a distributed database. A poor
choice compounds over years -- causing hotspots, expensive resharding operations, and cross-shard
query explosions. At the staff/principal level you are expected to design shard key strategies that
survive 100x growth without architectural rewrites.

---

## 1. The Auto-Increment ID Problem

The instinct to shard on `id BIGINT AUTO_INCREMENT` is the most common early mistake.

```
Shard 0: id 1 - 1,000,000
Shard 1: id 1,000,001 - 2,000,000
Shard 2: id 2,000,001 - 3,000,000   <-- all new writes land here
```

### Failure modes

| Problem                  | Impact                                              |
|--------------------------|-----------------------------------------------------|
| Write hotspot            | Latest shard absorbs 100% of inserts                |
| Temporal correlation     | Recent data (most queried) lives on one shard       |
| Range scan locality loss | Related entities (same user) scattered across shards |
| Rebalancing difficulty   | Ranges are unbounded; splits require data movement  |

Auto-increment IDs also leak business information (order volume, user count) and create contention
on the sequence generator itself under high concurrency.

---

## 2. Hash-Based Distribution

Hash the shard key to distribute writes uniformly:

```
shard_id = hash(entity_id) % num_shards
```

```
+----------+     hash()      +---+---+---+---+
| entity_id| -------------> | 0 | 1 | 2 | 3 |  (shards)
+----------+                +---+---+---+---+
   "user_8291"  ->  hash = 74291  ->  74291 % 4 = 3  ->  Shard 3
```

### Tradeoffs

| Advantage                        | Disadvantage                              |
|----------------------------------|-------------------------------------------|
| Uniform write distribution       | Range queries require scatter-gather       |
| Simple implementation            | Adding shards invalidates all assignments  |
| No hotspot on latest data        | Related entities not co-located            |
| Predictable shard for point reads| Resharding = full data migration           |

### The Modulo Trap

When you grow from 4 to 5 shards, `hash(key) % 4` and `hash(key) % 5` disagree for ~80% of keys.
This is why naive hash-mod sharding leads to painful resharding events.

---

## 3. Range-Based Partitioning

Assign contiguous key ranges to shards, commonly used with time-series or lexicographically ordered data.

```
Shard A:  [aaa ... gzz]
Shard B:  [haa ... nzz]
Shard C:  [oaa ... zzz]

         Key Space
  |--------|--------|--------|
  A        B        C
  aaa     haa      oaa     zzz
```

### When Range Sharding Works

- Time-series ingestion with TTL (old shards drop entirely)
- Alphabetical tenant isolation (tenant names as shard keys)
- Sequential scan workloads (analytics, batch processing)

### When It Fails

- Celebrity/whale accounts create range hotspots
- Skewed key distributions (90% of users start with 'a'-'m')
- Write amplification on boundary splits

---

## 4. Composite Shard Keys

The staff-level answer: combine multiple dimensions into a single shard key.

```
shard_key = (tenant_id, timestamp_bucket)
```

This achieves:
- **Tenant isolation**: all data for tenant X lives on predictable shards
- **Time distribution**: within a tenant, data spreads across time buckets
- **Query locality**: queries scoped to tenant + time range hit minimal shards

### Architecture

```
                      Composite Key: (tenant_id, hour_bucket)

  tenant_1, hour_0  ──┐
  tenant_1, hour_1  ──┤──> hash((t1, h0)) % N ──> Shard 2
  tenant_1, hour_2  ──┘                            Shard 7
                                                    Shard 1
  tenant_2, hour_0  ──┐
  tenant_2, hour_1  ──┤──> hash((t2, h0)) % N ──> Shard 5
  tenant_2, hour_2  ──┘                            Shard 0
                                                    Shard 3
```

### Granularity Selection Table

| Bucket Size | Write Distribution | Query Scope | Storage per Shard |
|-------------|-------------------|-------------|-------------------|
| 1 minute    | Excellent         | Very narrow | Small, many shards |
| 1 hour      | Good              | Moderate    | Balanced          |
| 1 day       | Moderate          | Broad       | Large partitions  |
| 1 week      | Poor for spiky    | Very broad  | Potential hotspots |

---

## 5. Directory-Based Sharding

Maintain an explicit lookup table mapping entities to shards.

```
+------------------+          +-------------------+
|  Shard Directory  |          |    Data Shards    |
|  (metadata DB)    |          |                   |
|                   |          |  +-----+  +-----+ |
|  user_1 -> shard3 | -------> |  |  S0 |  |  S1 | |
|  user_2 -> shard1 |          |  +-----+  +-----+ |
|  user_3 -> shard0 |          |  +-----+  +-----+ |
|  ...              |          |  |  S2 |  |  S3 | |
+------------------+          |  +-----+  +-----+ |
                               +-------------------+
```

### Tradeoffs

| Advantage                            | Disadvantage                                |
|--------------------------------------|---------------------------------------------|
| Arbitrary placement per entity       | Directory is a single point of failure       |
| Resharding = update directory, move data | Extra hop for every query                |
| Supports heterogeneous shard sizes   | Directory must be highly available + cached  |
| Enables live migration per-entity    | Cache invalidation during moves is complex   |

### Making It Production-Ready

1. **Cache the directory** in application memory with TTL
2. **Use a consensus store** (etcd, ZooKeeper) for the directory
3. **Versioned entries** -- stale cache reads redirect, not fail
4. **Batch directory lookups** to amortize the extra hop

---

## 6. Virtual Sharding (Vnodes)

Map the key space to a large number of virtual shards (e.g., 4096), then map virtual shards to
physical nodes. Resharding moves virtual shards, not individual keys.

```
Key Space (hash ring)
  0 ─────────────────────── 2^32
  │  v0  v1  v2 ... v4095  │
  │   │   │   │       │    │
  │   ▼   ▼   ▼       ▼    │
  │  ┌─────────────────┐   │
  │  │ Physical Node Map│   │
  │  │ v0-v1023 -> N0   │   │
  │  │ v1024-v2047 -> N1│   │
  │  │ v2048-v3071 -> N2│   │
  │  │ v3072-v4095 -> N3│   │
  │  └─────────────────┘   │

  Adding N4: move v3072-v3583 from N3 to N4
  Only ~12.5% of data moves instead of ~80% with hash-mod
```

### Why 4096 Vnodes?

- Enough granularity for fine-grained rebalancing
- Small enough that the vnode map fits in memory
- Power of 2 for efficient bitwise hash mapping
- Cassandra default: 256 vnodes per node (tunable)

---

## 7. Real-World Case Studies

### Instagram: Sharding PostgreSQL

Instagram's original shard key: `(shard_id << 23) | (local_sequence)` embedded in snowflake-style IDs.

- 2000+ logical shards mapped to a few dozen physical PostgreSQL instances
- Each logical shard = a PostgreSQL schema within a database
- Shard key embedded in the photo/user ID itself
- Resharding = moving schemas between instances (not row-level)

### Discord: Message Storage Evolution

```
Phase 1: MongoDB, shard by guild_id
  Problem: Large guilds (millions of members) = massive hotspots

Phase 2: Cassandra, shard by (channel_id, message_bucket)
  Bucket = message_id range (~10 days of messages)
  Problem: Tombstone accumulation on channels with heavy deletes

Phase 3: ScyllaDB with refined compaction
  Same shard key, but tuned compaction strategies per-channel-size
```

### Slack: Workspace-Based Sharding

- Shard key: `workspace_id`
- Problem: enterprise workspaces 1000x larger than free tier
- Solution: large workspaces get dedicated shards (directory-based override)
- Lesson: **hybrid sharding** -- hash by default, directory override for outliers

---

## 8. Decision Tree for Shard Key Selection

```
                         Start
                           │
                     ┌─────▼──────┐
                     │ Single      │───Yes──> No sharding needed
                     │ machine OK? │          (vertical scale first)
                     └─────┬──────┘
                           │ No
                     ┌─────▼──────────┐
                     │ Natural tenant  │───Yes──> Composite key:
                     │ boundary?       │          (tenant_id, time_bucket)
                     └─────┬──────────┘
                           │ No
                     ┌─────▼──────────┐
                     │ Time-series     │───Yes──> Range partition on
                     │ workload?       │          time with TTL
                     └─────┬──────────┘
                           │ No
                     ┌─────▼──────────┐
                     │ Need range      │───Yes──> Range on query dim
                     │ queries?        │          + hash secondary
                     └─────┬──────────┘
                           │ No
                     ┌─────▼──────────┐
                     │ Uniform access  │───Yes──> Hash on primary
                     │ pattern?        │          entity ID
                     └─────┬──────────┘
                           │ No
                     ┌─────▼──────────┐
                     │ Unpredictable / │───Yes──> Directory-based
                     │ evolving access?│          with virtual shards
                     └─────┬──────────┘
                           │ No
                           ▼
                     Hybrid: hash default +
                     directory overrides for
                     outlier entities
```

---

## 9. Anti-Patterns to Call Out in Interviews

| Anti-Pattern                  | Why It Fails                                         |
|-------------------------------|------------------------------------------------------|
| Shard on user email           | Mutable field; changing email = cross-shard migration |
| Shard on auto-increment PK   | Write hotspot on latest shard                        |
| Too few shards (e.g., 4)     | Cannot rebalance without full reshard                |
| Shard on high-cardinality dim | Scatter-gather on every analytical query             |
| Ignore cross-shard joins      | 10x latency when joins span 90% of shards           |
| Shard key not in query path   | Every query becomes a broadcast                      |

---

## Interview Narrative

**When asked about shard key design, structure your answer as follows:**

> "Shard key selection is the most durable architectural decision in a distributed system, so I
> approach it by working backward from the query patterns and forward from the growth model.
>
> First, I identify the primary access pattern. If there is a natural tenant boundary -- like
> workspace_id in a SaaS product -- I start with a composite key: `(tenant_id, time_bucket)`. This
> gives me write distribution across time buckets while preserving query locality within a tenant.
>
> Second, I plan for outliers. Hash-based distribution works for the 99th percentile of tenants, but
> the top 0.1% (whale accounts) will still hotspot. For those, I use a directory-based override that
> places them on dedicated shards. This hybrid approach is what companies like Slack and Notion use
> in production.
>
> Third, I always use virtual shards (typically 4096+) mapped to physical nodes. This decouples the
> logical partitioning from physical topology, so when we need to scale from 8 to 12 nodes, we
> move virtual shards rather than rehashing the entire dataset.
>
> Finally, I embed the shard key in the entity ID itself -- Instagram's snowflake-style IDs are the
> gold standard here. This means any service holding an entity ID can derive the shard location
> without a directory lookup, which eliminates the metadata service as a bottleneck.
>
> The key mistake I have seen teams make is choosing a shard key that optimizes for writes but
> ignores the dominant read path. At scale, reads outnumber writes 10:1 to 100:1, so the shard
> key must align with the most frequent query predicate, not the insertion key."

**Follow-up traps to prepare for:**

- "What happens when your composite key's time bucket rolls over?" (Answer: pre-create shards,
  warm caches, stagger bucket boundaries across tenants)
- "How do you handle cross-shard transactions?" (Answer: Saga pattern, or avoid them by
  co-locating related entities on the same shard via shard key design)
- "What if the access pattern changes after launch?" (Answer: virtual shards + directory-based
  overrides give you escape hatches; instrument shard-level metrics from day one)
