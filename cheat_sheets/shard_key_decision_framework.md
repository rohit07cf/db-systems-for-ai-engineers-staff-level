# Shard Key Decision Framework -- Quick Reference

## The Core Principle

> A shard key determines **data locality** and **query routing**. The right key maximizes
> single-shard queries and distributes writes evenly. The wrong key creates hot partitions
> or forces scatter-gather on every read.

## Decision Tree

```
Step 1: Identify your PRIMARY access pattern
         |
         v
Is there a natural tenant/entity ID? (user_id, org_id, device_id)
    |                          |
   YES                        NO
    |                          |
    v                          v
Is the entity high-cardinality?    Is there a time-series pattern?
(>10K distinct values)              |            |
    |          |                   YES           NO
   YES         NO                   |            |
    |          |                    v            v
    v          v             Compound key:   Consider
Single-key  Compound key:   time_bucket +   hash-based
candidate   entity_id +     entity_id       synthetic key
    |       secondary_field       |
    v              |              v
Step 2: Check      v         Beware: latest
for hot       Good for       time bucket
partitions    low-cardinality  becomes hot
    |         entities
    v
Does top 1% of entities generate >20% of traffic?
    |                    |
   YES                   NO
    |                    |
    v                    v
Apply sub-sharding     Use entity_id as
(entity_id + modN)     shard key directly
or read replicas
```

## Shard Key Properties Scorecard

Rate each candidate key 1-5:

| Property | What to Evaluate | Weight |
|---|---|---|
| **Cardinality** | How many distinct values? Need >> shard count | Critical |
| **Frequency uniformity** | Are values evenly distributed? | Critical |
| **Query isolation** | Do most queries target a single shard? | Critical |
| **Monotonicity** | Monotonic keys (auto-increment, timestamp) create hot spots | Important |
| **Immutability** | Can the key value change? Changing shard keys is expensive | Important |
| **Joinability** | Can related data co-locate on the same shard? | Moderate |
| **Growth pattern** | Does distribution stay even as data grows? | Moderate |

## Anti-Patterns Table

| Anti-Pattern | Why It Fails | Real-World Example | Fix |
|---|---|---|---|
| **Auto-increment ID** | Monotonic: all writes go to last shard | Naive MySQL sharding by `id` | Hash the ID or use UUID |
| **Timestamp only** | Latest time window gets all writes | Time-series with shard = date | Compound: `device_id + time_bucket` |
| **Low-cardinality key** | Few distinct values = max shards capped | Shard by `country` (< 200 values) | Compound: `country + user_id_hash` |
| **User ID for celebrity problem** | Power-law: top users get 1000x traffic | Twitter sharding by `author_id` | Sub-shard hot users, or shard by `tweet_id` for writes + secondary index for reads |
| **Mutable field** | Resharding on every update | Shard by `status` (pending -> completed) | Use immutable entity ID |
| **Random UUID** | Excellent distribution but impossible range scans | Shard by UUIDv4 | Use UUIDv7 (time-ordered) or ULID if you need range queries |
| **Geo key only** | Manhattan has more data than Montana | Shard by `zip_code` | Hash-based partitioning or geo-hash with rebalancing |

## Real-World System Examples

### Instagram (Sharded PostgreSQL)

```
Shard key:  user_id
Shards:     Several thousand logical shards
ID format:  [timestamp_ms | 13 bits][logical_shard_id | 13 bits][sequence | 10 bits]

Why it works:
- Photos, likes, follows all keyed by user_id
- ID encodes shard ID -> no lookup needed for routing
- Timestamp prefix enables time-ordered IDs
- 8,192 logical shards mapped to fewer physical shards
```

### Discord (Cassandra -> ScyllaDB)

```
Shard key:  channel_id   (for messages)
Sort key:   message_id   (Snowflake: time-ordered)

Why it works:
- Messages queried by channel (primary access pattern)
- Hot channels (millions of members) are the challenge
  -> Bucket by channel_id + time_bucket to bound partition size
- Target: < 100 MB per partition

Partition strategy:
  partition_key = (channel_id, bucket)
  bucket = message_id >> 22 / bucket_interval
```

### DynamoDB Design Patterns

```
Pattern 1: Simple entity
  PK = USER#<user_id>
  SK = PROFILE | ORDER#<order_id> | ...

Pattern 2: Many-to-many (adjacency list)
  PK = USER#<user_id>,  SK = GROUP#<group_id>
  PK = GROUP#<group_id>, SK = USER#<user_id>

Pattern 3: Write-heavy with GSI
  PK = hash(event_id)                    -- even distribution
  GSI: PK = user_id, SK = timestamp      -- query pattern

Anti-pattern: PK = date (all traffic to one partition)
Fix:         PK = date#shard_N (scatter-gather with N shards)
```

### Vitess (YouTube / Slack)

```
Shard key:  Vindex (virtual index)
Types:
  - Hash vindex:     Even distribution, no range queries
  - Functional vindex: Custom routing logic
  - Lookup vindex:   Secondary lookup table for non-PK routing

Key insight: Vitess decouples logical shards from physical,
enabling online resharding without application changes.
```

## Migration Checklist -- Resharding / Changing Shard Key

- [ ] **Audit all queries**: catalog every query pattern hitting the table
- [ ] **Simulate new key**: run shadow traffic analysis with proposed key
- [ ] **Estimate new distribution**: histogram of values in the new key
- [ ] **Dual-write phase**: write to old AND new shard layout
- [ ] **Backfill historical data**: copy/transform existing data to new layout
- [ ] **Verify consistency**: compare read results old vs. new (shadow reads)
- [ ] **Cutover reads**: switch reads to new layout (feature flag)
- [ ] **Cutover writes**: stop dual-write, write only to new layout
- [ ] **Cleanup**: drop old shards after retention period
- [ ] **Update monitoring**: new shard-level metrics and alerts

> **Timeline estimate**: 2-6 months for a major resharding at FAANG scale.
> Instagram's migration from one shard scheme took ~1 year with zero downtime.

## Scatter-Gather vs. Single-Shard Query

| Query Type | Latency | When It Happens |
|---|---|---|
| Single-shard (partition key in WHERE) | **p50: 2-5 ms** | Shard key matches query predicate |
| Scatter-gather (all shards) | **p50: 20-200 ms** | No shard key in query; fan-out to N shards |
| Cross-shard join | **p50: 50-500 ms** | Joins across different shard keys |

**Rule**: > 80% of queries should be single-shard. If not, reconsider the key.

## Compound Shard Key Patterns

| Pattern | Example | Use When |
|---|---|---|
| `entity_id` | `user_id` | Single dominant access pattern |
| `entity_id + sort_key` | `user_id + timestamp` | Range queries within entity |
| `tenant_id + entity_id` | `org_id + doc_id` | Multi-tenant SaaS |
| `hash(entity_id)` | `hash(user_id)` | Need even distribution, no range scans |
| `entity_id + bucket` | `channel_id + time_bucket` | Bound partition size for hot entities |
| `geo_hash + entity_id` | `geohash_4 + store_id` | Geo-local queries |

## Key Principles for Interviews

1. **"The shard key is the most consequential schema decision."**
   Changing it later requires migrating every row. Get it right or plan for migration.

2. **"Optimize for the dominant query pattern."**
   If 90% of queries include `user_id`, shard by `user_id` even if it complicates the 10%.

3. **"Cardinality must exceed shard count by 100x."**
   Low cardinality = uneven distribution = hot shards. Need headroom for future growth.

4. **"Monitor partition size, not just count."**
   Even distribution by key count != even by data size. A few large entities skew storage.

5. **"Plan for the celebrity problem from day one."**
   Power-law distributions exist in almost every social/commerce system.
   Design sub-sharding or caching before you need it.

6. **"Every cross-shard query is a distributed transaction you chose to accept."**
   Minimize cross-shard operations by co-locating related data.

7. **"Logical shards decouple from physical."**
   Start with more logical shards than physical nodes (e.g., 4096 logical -> 16 physical).
   Resharding becomes moving logical shards between physical nodes, not rewriting data.

## Quick Reference: How Many Shards?

```
Estimate:
  data_size_gb / target_shard_size_gb = min_shards (storage)
  peak_qps / per_shard_qps_limit      = min_shards (throughput)
  shards_needed = max(storage_shards, throughput_shards) * 1.5 (headroom)

Example:
  2 TB data, 50 GB target/shard   -> 40 shards (storage)
  200K QPS, 10K QPS/shard         -> 20 shards (throughput)
  Answer: 40 * 1.5 = 60 shards

Tip: Use powers of 2 (64, 128, 256) for easy splitting.
Start with more logical shards than you think you need.
```
