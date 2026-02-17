# Designing for 10x Scale

## Executive Summary (for Staff Interview)

- Design for **current scale + one order of magnitude** — not two, not three
- Over-engineering for 1000x is as costly as under-engineering for 2x
- Every system has a **scaling cliff** — know where yours is before you hit it
- The bottleneck always shifts: CPU → Memory → Network → Disk IOPS → Coordination
- Staff engineers plan the **migration path**, not just the current architecture

## Architectural Tradeoffs

| Decision | Pros | Cons | At Small Scale | At 10x Scale |
|---|---|---|---|---|
| Build for current needs only | Fast delivery, simple | Painful migration later | Correct for startups | Requires rearchitecture |
| Build for 10x from day one | Smooth scaling path | Over-engineering, delayed delivery | Premature optimization | Justified if growth is predictable |
| Build for 100x from day one | Never need to migrate | Massive over-investment | Almost never correct | Only if you're already at 10x |
| Modular with migration hooks | Balanced approach | Requires discipline | Slight overhead | Easy to swap components |

## Deep Technical Internals

### The Scaling Cliff Map

```
Load -->
         |
         |    Cliff 3: Cross-region coordination
         |    (consensus overhead, conflict resolution)
         |    _______________
         |   /
         |  /  Cliff 2: Single-node limits
         | /   (memory, CPU, connections)
         |/    _______________
         |    /
         |   /  Cliff 1: Single-thread bottleneck
         |  /   (lock contention, connection limits)
         | /    _______________
         |/
         |
         +-----------------------------------------> Scale
         1K     10K    100K    1M     10M    100M

Each cliff requires architectural change, not just hardware scaling.
```

### Where Systems Break at 10x

| System | Comfortable Range | 10x Stress Point | What Breaks | Migration Path |
|---|---|---|---|---|
| Single PostgreSQL | <1TB, <10K QPS | 10TB, 100K QPS | Connection limits, replication lag, vacuum pressure | Read replicas → Sharding → Citus |
| Redis single node | <25GB, <100K ops/s | 250GB, 1M ops/s | Memory limit, single-thread CPU | Cluster mode (16384 slots) |
| MongoDB replica set | <5TB, <50K ops/s | 50TB, 500K ops/s | WiredTiger cache, oplog window | Sharded cluster |
| DynamoDB single table | <100GB, <40K WCU | 1TB, 400K WCU | Hot partition, GSI propagation delay | Partition key redesign, write sharding |
| Elasticsearch | <1B docs, <10K QPS | 10B docs, 100K QPS | Heap pressure, merge storms | Index lifecycle, frozen tiers |
| Kafka | <100K msgs/sec | 1M msgs/sec | Partition rebalance, consumer lag | More partitions, topic splitting |

### The 10x Checklist

For every database component, answer:

```
1. DATA VOLUME
   Current: ___TB    At 10x: ___TB
   [ ] Fits on single node?
   [ ] Backup/restore time acceptable?
   [ ] Schema migration feasible?

2. THROUGHPUT
   Current: ___QPS   At 10x: ___QPS
   [ ] Single node handles it?
   [ ] Read:write ratio changes?
   [ ] Connection pool sufficient?

3. LATENCY
   Current p99: ___ms   At 10x: still ___ms?
   [ ] Index performance degrades?
   [ ] Cache hit rate changes?
   [ ] Cross-shard queries needed?

4. OPERATIONAL
   [ ] Monitoring still works?
   [ ] Backup window still fits?
   [ ] Team can handle complexity?
   [ ] On-call burden acceptable?

5. COST
   Current: $___/mo    At 10x: $___/mo
   [ ] Linear cost scaling?
   [ ] Super-linear cost scaling?
   [ ] Budget approved?
```

### Scaling Strategy Ladder

```
Level 0: Single Node
  - Vertical scaling (bigger machine)
  - Good until: ~1TB data, ~10K QPS
  - Cost: $

Level 1: Read Replicas
  - Separate read/write paths
  - Good until: ~5TB data, ~50K read QPS
  - Cost: $$
  - Watch: Replication lag

Level 2: Caching Layer
  - Redis/Memcached in front of DB
  - Good until: ~500K read QPS (90%+ cache hit)
  - Cost: $$$
  - Watch: Cache invalidation, thundering herd

Level 3: Functional Partitioning
  - Different DBs for different services
  - Good until: Each partition < Level 1 limits
  - Cost: $$$
  - Watch: Cross-service queries, distributed transactions

Level 4: Horizontal Sharding
  - Same data split across nodes
  - Good until: Practically unlimited
  - Cost: $$$$
  - Watch: Shard key selection, cross-shard queries, rebalancing

Level 5: Multi-Region
  - Geo-distributed data
  - Good until: Global scale
  - Cost: $$$$$
  - Watch: Consistency, conflict resolution, latency
```

### What Staff Engineers Prepare in Advance

| Preparation | Why | Cost of Not Doing It |
|---|---|---|
| Shard-key aware data model | Sharding migration without data model change | Full data migration under load |
| Abstract storage interface | Swap implementations without app changes | Rewrite every caller |
| Connection pool monitoring | Detect limits before hitting them | Random failures under load |
| Capacity planning dashboard | See the cliff coming 3 months out | Emergency scaling at 3am |
| Runbook for migration | Tested procedure for the next level | Ad-hoc panic migration |
| Shadow traffic testing | Validate next architecture under real load | Hope-based engineering |

## Failure Modes at Scale

| Scale Transition | Common Failure | Why It Surprises People | Prevention |
|---|---|---|---|
| 1 node → 3 replicas | Replication lag during peak | Didn't account for write burst patterns | Throttle writes, monitor lag |
| No cache → cached | Thundering herd on cold start | All keys expire simultaneously | Staggered TTL, mutex on miss |
| Single DB → sharded | Cross-shard query performance | Queries that were fast now scatter-gather | Query pattern audit before sharding |
| Single region → multi-region | Split brain during network partition | Didn't test partition scenarios | Chaos engineering, partition testing |
| Vertical → horizontal scale | Connection count explosion | 10x nodes × 100 services = 1000 connections per DB | Connection pooling (PgBouncer) |

## Multi-Region Considerations

**Scaling from single region to multi-region is the biggest cliff:**

| Single Region | Multi-Region | Impact |
|---|---|---|
| Strong consistency is free | Strong consistency costs latency | Every write adds 80-200ms |
| Failover is fast (seconds) | Failover is complex (minutes) | DNS propagation, connection draining |
| One set of metrics | Three sets of metrics | Monitoring complexity triples |
| One deploy pipeline | Three deploy pipelines | Deploy risk multiplies |
| $X/month | $3-8X/month | Budget impact is significant |

**When to go multi-region:**

- Regulatory requirement (data residency)
- Latency requirement (<50ms globally)
- Availability requirement (>99.99%)
- **NOT** just because it sounds impressive in an interview

## Cost Model Thinking

**Cost scaling patterns:**

| Pattern | Example | Cost at 10x |
|---|---|---|
| Linear | Storage (S3, EBS) | 10x |
| Sub-linear | Managed services with tiered pricing | 7-8x |
| Super-linear | Network (cross-region replication) | 15-20x |
| Step function | Need a new shard/cluster | Jump at threshold |
| Logarithmic | Caching (higher hit rate at scale) | 3-5x |

**The 10x cost trap:**

```
Current cost: $10,000/month
Expected at 10x (linear): $100,000/month
Actual at 10x: $250,000/month

Why? Super-linear costs:
- Cross-shard queries: O(N²) network calls
- Replication fan-out: Each write hits 3-9 replicas
- Monitoring/observability: Data volume × node count
- Engineering time: Debugging distributed systems
```

## Interview Narrative

**"How I would explain this in a Staff interview"**

1. "I design for current scale plus one order of magnitude. If we're at 10K QPS, I design for 100K QPS. Designing for 10M QPS when you're at 10K is wasteful — the requirements will change before you get there."

2. "Every system has a scaling cliff. For PostgreSQL, it's around 5-10TB and 50K QPS. For Redis single node, it's around 25GB memory. I map these cliffs for every component and build migration plans before we reach them."

3. "I use the scaling strategy ladder: vertical scaling → read replicas → caching → functional partitioning → sharding → multi-region. Each step adds complexity, so I only climb when the current level is insufficient."

4. "The most important preparation for 10x is not the architecture — it's the migration path. How do we get from Level 2 to Level 3 without downtime? I build that plan and test it before we need it."

5. "Cost scaling is rarely linear. I model the cost curve explicitly. DynamoDB on-demand scales linearly in cost — which means at 10x load, the cost is 10x. But some systems have super-linear cost curves that catch teams off guard."

6. "Shadow traffic testing is how I validate the next architecture. I replay production traffic against the new system in staging, measure performance, and identify issues before the migration."

7. "The biggest 10x mistake I see is premature multi-region. Going multi-region adds 3-5x cost and 10x operational complexity. I always ask: can we solve this with a better single-region architecture first?"
