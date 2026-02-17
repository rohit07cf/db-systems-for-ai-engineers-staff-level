# Redis Cluster -- Quick Reference

## Architecture at a Glance

| Property | Value |
|---|---|
| Hash slots | **16,384** (0 -- 16383) |
| Slot assignment | `CRC16(key) % 16384` |
| Min nodes for HA | **6** (3 masters + 3 replicas) |
| Max recommended nodes | ~1,000 (practical limit) |
| Replication model | Async (default), `WAIT` for semi-sync |
| Gossip protocol | Binary protocol on port **+10000** (e.g., 6379 -> 16379) |
| Config epoch | Monotonically increasing; drives failover precedence |

## Slot Model Summary

```
Key "user:1234"
   |
   v
CRC16("user:1234") = 0x3E2F = 15919
   |
   v
15919 % 16384 = 15919  -->  Node C owns slots 10923-16383
   |
   v
Request routed to Node C
```

**Hash tags**: Force keys to same slot with `{tag}`.
Example: `user:{1234}:profile` and `user:{1234}:cart` both hash on `1234`.

> **Interview Tip**: Multi-key operations (MGET, MSET, Lua scripts, transactions)
> only work when ALL keys map to the **same slot**. Use hash tags deliberately.

## Client-Side Routing

```
Smart client (Jedis, Lettuce, redis-py):
  1. On connect: fetch slot map via CLUSTER SLOTS
  2. Route commands directly to correct node
  3. On -MOVED: update local slot map, retry
  4. On -ASK: send ASKING + command to target (one-time redirect)
  5. Periodically refresh slot map (every 30-60s)

Proxy mode (Twemproxy, Envoy, Redis Cluster Proxy):
  - Client sees single endpoint
  - Proxy handles routing
  - Tradeoff: extra hop (+0.2-0.5 ms) vs. simpler clients
```

> **When to use proxy**: non-Redis-aware clients, legacy apps, or when
> you need connection pooling across thousands of application instances.

## Essential Commands

| Command | Purpose |
|---|---|
| `CLUSTER SLOTS` | List slot-to-node mappings |
| `CLUSTER NODES` | Full cluster topology |
| `CLUSTER INFO` | Cluster health summary |
| `CLUSTER KEYSLOT <key>` | Which slot a key maps to |
| `CLUSTER SETSLOT <slot> MIGRATING <node>` | Start slot migration |
| `CLUSTER SETSLOT <slot> IMPORTING <node>` | Target accepts migration |
| `CLUSTER FAILOVER` | Manual replica promotion |
| `CLUSTER FORGET <node-id>` | Remove node from topology |
| `-MOVED <slot> <host>:<port>` | Redirect: slot permanently moved |
| `-ASK <slot> <host>:<port>` | Redirect: slot mid-migration |

## Failure Detection & Failover Timings

| Parameter | Default | Production Recommendation |
|---|---|---|
| `cluster-node-timeout` | 15,000 ms | 5,000 -- 15,000 ms |
| PFAIL (suspected failure) | After `cluster-node-timeout` | -- |
| FAIL (confirmed) | Majority of masters agree PFAIL | -- |
| Replica election delay | `500ms + random(0-500ms) + rank*1000ms` | -- |
| **Total failover time** | **15 -- 30 s** (default) | **5 -- 15 s** (tuned) |
| `cluster-replica-validity-factor` | 10 | 0 for strictest freshness |

> **Key Number**: Default worst-case failover = ~30 seconds of unavailability
> for the affected slots. Plan for this in SLA discussions.

## Memory Estimation Formulas

```
Per-key overhead:    ~70 bytes (dictEntry + redisObject + SDS header)
String value:        len(value) + SDS header (~3-9 bytes)
Hash (ziplist):      ~64 bytes base + entries (up to hash-max-ziplist-entries=128)
Hash (hashtable):    ~160 bytes base + 70 bytes/field
Sorted Set (ziplist): ~64 bytes + entries (up to zset-max-ziplist-entries=128)
Sorted Set (skiplist): ~200 bytes base + ~100 bytes/member

Total cluster overhead:  ~1-2 MB per node for gossip buffers
Replication buffer:      client-output-buffer-limit replica 256mb 64mb 60
```

**Quick estimate**: `total_memory = num_keys * (avg_key_size + avg_value_size + 70)`

## Performance Tuning Checklist

- [ ] **Pipeline commands** -- batch 50-100 commands per pipeline round-trip
- [ ] **Use hash tags** for related keys accessed together
- [ ] **Avoid large keys** -- keep values < 10 KB; lists/sets < 10K elements
- [ ] **Set maxmemory-policy** -- `allkeys-lru` for caches, `volatile-ttl` for mixed
- [ ] **Disable KEYS command** in production (rename-command)
- [ ] **Use SCAN** instead of KEYS for iteration
- [ ] **Monitor slow log** -- `SLOWLOG GET 25` regularly
- [ ] **Tune tcp-backlog** to 511+ and `somaxconn` at OS level
- [ ] **Disable THP** (Transparent Huge Pages) on Linux
- [ ] **Set `hz` to 50-100** for faster eviction/expiry processing
- [ ] **Replica read routing** -- offload reads with `READONLY` on replicas

## Common Pitfalls

| Pitfall | Why It Hurts | Fix |
|---|---|---|
| Hot key (one key gets 80%+ traffic) | Single node bottleneck | Split into N sub-keys, client-side cache |
| Big key (>1 MB value) | Blocks event loop, replication lag | Decompose into smaller keys/hashes |
| Cross-slot MULTI/EXEC | Fails with `-CROSSSLOT` | Hash tags or redesign data model |
| Thundering herd on cache miss | Backend overwhelmed | Mutex/lock pattern, stale-while-revalidate |
| Slot migration during peak | Latency spikes from ASK redirects | Migrate during low-traffic windows |
| No persistence + reboot | Data loss, full resync | Enable AOF `appendonly yes`, `appendfsync everysec` |
| Gossip overload (>200 nodes) | High CPU on gossip protocol | Limit cluster size; shard into multiple clusters |

## Data Structures -- When to Use What

| Structure | Best For | Memory Efficiency | Max Recommended Size |
|---|---|---|---|
| String | Simple K/V, counters, locks | Baseline | 512 MB (hard limit) |
| Hash | Object storage, user profiles | Excellent (ziplist < 128 fields) | 10K fields |
| List | Queues, activity feeds | Good | 10K elements |
| Set | Tags, unique tracking | Moderate | 10K members |
| Sorted Set | Leaderboards, rate limiters | Good (ziplist < 128 members) | 10K members |
| Stream | Event log, message queue | Good | Trim with MAXLEN |
| HyperLogLog | Cardinality estimation | **12 KB fixed** | N/A (probabilistic) |
| Bitmap | Feature flags, presence | **Excellent** | 512 MB |

## Cluster Scaling Decision

```
Single Redis           Cluster needed?
  |                         |
  +-- < 25 GB RAM?     NO --+-- Use standalone + Sentinel
  |                         |
  +-- < 100K ops/s?    NO --+-- Use standalone + read replicas
  |                         |
  YES to either              |
  |                         |
  v                         v
Redis Cluster           Consider:
(min 6 nodes)           - Shard count = ceil(total_memory / per_node_memory)
                        - Leave 30% headroom per node
                        - Each master: max ~25 GB for safe replication
```

## Things That Will Impress in an Interview

1. **Explain the CRC16 slot model** and why 16,384 slots was chosen
   (tradeoff: gossip packet size vs. granularity; 16K slots = 2KB bitmap in heartbeat)
2. **Describe MOVED vs ASK redirects** and the slot migration state machine
3. **Articulate the split-brain risk**: async replication means writes to old master
   during partition can be lost after failover; `WAIT` provides semi-sync guarantees
4. **Quantify failover**: "Default cluster-node-timeout is 15s, so worst-case
   failover is ~30s. We tuned it to 5s for sub-10s failover at the cost of
   more false-positive PFAIL detections."
5. **Hot key mitigation**: describe client-side caching with tracking invalidation
   (Redis 6+ `CLIENT TRACKING`) or deterministic sub-sharding
6. **Memory efficiency**: "We used hash ziplist encoding for small objects,
   keeping entries under 128 and value size under 64 bytes, cutting memory 5x
   vs. naive key-per-field"
7. **Cross-cluster replication**: mention CRDT-based Active-Active (Redis Enterprise)
   or custom CDC pipelines for multi-region
8. **Pub/Sub limitation**: cluster Pub/Sub broadcasts to ALL nodes; for high-throughput
   messaging, use Redis Streams with consumer groups instead

## Key Numbers to Remember

| Metric | Value |
|---|---|
| Hash slots | 16,384 |
| Max cluster nodes (practical) | ~1,000 |
| Per-key memory overhead | ~70 bytes |
| Default node timeout | 15 seconds |
| Pipeline sweet spot | 50-100 commands |
| Recommended max value size | 10 KB |
| Recommended max node memory | 25 GB |
| Gossip port offset | +10,000 |
| Single-threaded throughput (6.x) | ~100K-300K ops/s |
| io-threads throughput (7.x) | ~500K-1M+ ops/s |
