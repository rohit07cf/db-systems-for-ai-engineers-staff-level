# Cost vs Latency vs Consistency

## Executive Summary (for Staff Interview)

- These three dimensions form the **iron triangle** of database architecture — you pick two, the third suffers
- Cost is not just dollars — it includes operational overhead, engineering time, and opportunity cost
- Latency is not just p50 — p99 and p99.9 determine user experience and cascade behavior
- Consistency is not binary — there's a spectrum from linearizable to eventual, and each level has different costs
- Staff engineers make these tradeoffs **per data path**, not per system

## Architectural Tradeoffs

| Decision | Pros | Cons | At Small Scale | At 10x Scale |
|---|---|---|---|---|
| Low latency + strong consistency | Correct and fast | Very expensive (memory, cross-region sync) | Achievable with single node | Requires Spanner-class infra ($$$) |
| Low latency + low cost | Cheap and fast | Eventual consistency, stale reads | Cache + single DB | Cache invalidation becomes critical |
| Strong consistency + low cost | Correct and cheap | High latency (disk-based, synchronous) | Single PostgreSQL node | Cross-region sync kills latency |
| All three optimized | Dream scenario | Impossible at scale | Possible at toy scale | Physics says no |

## Deep Technical Internals

### The Cost-Latency Curve

```
Cost ($)
  |
  |                                          * Spanner (global strong)
  |
  |                             * Redis Cluster (multi-region)
  |
  |                  * DynamoDB (on-demand)
  |
  |         * PostgreSQL (RDS)
  |
  |    * SQLite
  |
  +----+------+------+------+------+------+-----> Latency (ms)
      0.1    1     10    50   100   500

  Lower-left is impossible at scale.
  You're always on some curve trading cost for latency.
```

### Consistency Levels — Cost and Latency Impact

| Consistency Level | What It Guarantees | Latency Cost | Infra Cost | Use Case |
|---|---|---|---|---|
| **Linearizable** | Global ordering, freshest read | +50-200ms (cross-region) | $$$$ (synchronous replication) | Financial transactions |
| **Sequential** | Operations appear in some total order | +20-100ms | $$$ | Distributed locks |
| **Causal** | Respects happens-before | +5-20ms | $$ | Social feeds, messaging |
| **Session** | Read-your-writes within session | +1-5ms (sticky sessions) | $$ | User profile edits |
| **Eventual** | Converges eventually | +0ms (local reads) | $ | Analytics, recommendations |

### Per-Path Analysis (Real Example)

```
E-commerce Platform:

Path 1: Product catalog browse
  - Consistency: Eventual (stale by 30s is fine)
  - Latency: p99 < 20ms
  - Cost: Low (read replicas, cache)
  - Solution: Redis cache + read replicas

Path 2: Add to cart
  - Consistency: Session (read your own writes)
  - Latency: p99 < 50ms
  - Cost: Medium
  - Solution: Primary DB with session affinity

Path 3: Place order / payment
  - Consistency: Linearizable (double-charge is catastrophic)
  - Latency: p99 < 500ms (users tolerate slow checkout)
  - Cost: High (worth it)
  - Solution: Synchronous replication, distributed transaction

Path 4: Order history
  - Consistency: Eventual (minutes delay is fine)
  - Latency: p99 < 200ms
  - Cost: Low
  - Solution: Async replicated read store
```

### Latency Budget Breakdown

```
Total SLO: p99 < 100ms

Budget allocation:
  Network (client → LB):     10ms
  LB → Application:           5ms
  Application logic:          15ms
  Database query:             50ms  ← This is your DB budget
  Serialization/response:    10ms
  Buffer:                    10ms
                            ------
  Total:                    100ms

If DB needs cross-region:    +80ms ← Blows the budget
If DB needs disk seek:       +10ms ← Tight but possible
If DB serves from memory:     <1ms ← Plenty of room
```

### Cost Modeling at Scale

| Component | 1M users | 10M users | 100M users | Cost Driver |
|---|---|---|---|---|
| PostgreSQL RDS (db.r6g.2xlarge) | $1,200/mo | $4,800/mo (4 replicas) | $24,000/mo (multi-region) | CPU + storage IOPS |
| Redis (r6g.xlarge, cluster) | $800/mo | $3,200/mo | $16,000/mo (global) | Memory is expensive |
| DynamoDB (on-demand) | $500/mo | $8,000/mo | $120,000/mo | WCU/RCU pricing scales linearly |
| S3 | $23/mo | $230/mo | $2,300/mo | Storage is cheap; API calls add up |
| Cross-region replication | $100/mo | $2,000/mo | $40,000/mo | Network transfer pricing |

**Key insight**: DynamoDB looks cheap at small scale (on-demand) but can become the most expensive option at large scale. PostgreSQL has a higher base cost but better cost curve.

## Failure Modes at Scale

| Failure | What Happens | Detection | Mitigation |
|---|---|---|---|
| Consistency too strong | Writes block during partition, availability drops | Write timeout rate, availability SLO breach | Downgrade to eventual for non-critical paths |
| Consistency too weak | Users see stale data, duplicate actions | Customer complaints, idempotency key collisions | Strengthen consistency for affected paths |
| Latency budget exceeded | Cascading timeouts, service degradation | p99 breach, upstream timeout alerts | Add caching, move to faster tier, reduce consistency |
| Cost spiral | AWS bill doubles in a month | Cost anomaly alerts, per-service cost tracking | Implement caching, optimize queries, consider reserved capacity |
| Cache stampede | Cache miss → all traffic hits DB → DB dies | Cache hit rate drop, DB CPU spike | Probabilistic early refresh, mutex on cache miss |

## Multi-Region Considerations

**The Multi-Region Cost Multiplier:**

| Item | Single Region | 3 Regions (Active-Passive) | 3 Regions (Active-Active) |
|---|---|---|---|
| Compute | 1x | 1.5x (standby smaller) | 3x |
| Storage | 1x | 3x (full replicas) | 3x |
| Network | 1x | 2-3x (replication traffic) | 3-5x (bidirectional sync) |
| Operational complexity | 1x | 2x | 5-10x |
| Total estimated cost | 1x | 2-3x | 4-8x |

**Latency by region pair:**

| From → To | Latency (ms) | Impact on Synchronous Writes |
|---|---|---|
| US-East → US-West | ~60ms | Doubles write latency |
| US-East → EU-West | ~80ms | Triples write latency |
| US-East → AP-Southeast | ~200ms | Kills synchronous writes |
| Same AZ | <1ms | No impact |
| Cross AZ (same region) | 1-2ms | Minimal impact |

## Cost Model Thinking

**The hidden costs of each optimization:**

| Optimization | Direct Cost | Hidden Cost |
|---|---|---|
| Add Redis cache | $800/mo per node | Cache invalidation bugs, stale data incidents |
| Add read replicas | $600/mo per replica | Replication lag management, connection routing |
| Shard the database | $0 (same hardware) | 6 months of engineering, ongoing shard management |
| Move to DynamoDB | Varies | Team retraining, loss of SQL flexibility, vendor lock-in |
| Go multi-region | 3-5x infra cost | Conflict resolution engineering, testing complexity |
| Upgrade instance size | Linear cost increase | Diminishing returns past certain point |

**Cost optimization priority:**

1. **Query optimization** — Free, often 10-100x improvement
2. **Caching** — Cheap, 10-50x read improvement
3. **Read replicas** — Moderate cost, linear read scaling
4. **Vertical scaling** — Expensive, limited ceiling
5. **Sharding** — Very expensive (engineering time), unlimited ceiling
6. **Multi-region** — Most expensive, needed for global scale

## Interview Narrative

**"How I would explain this in a Staff interview"**

1. "I think of cost, latency, and consistency as a per-data-path decision, not a per-system decision. My product catalog can be eventually consistent and cached aggressively, while my payment path needs linearizable consistency regardless of cost."

2. "I start with a latency budget. If the total API SLO is 100ms p99, and I allocate 50ms to the database, that constrains whether I can do cross-region reads, whether I need an in-memory layer, and what consistency level I can afford."

3. "Cost modeling is about trajectory, not snapshots. I model cost at 1x, 10x, and 100x current load. Some systems (DynamoDB on-demand) are cheap to start but expensive to scale. Others (self-hosted PostgreSQL) are expensive to start but predictable at scale."

4. "The most impactful cost optimization is almost always query optimization and caching — not infrastructure changes. I've seen teams add $50K/month in Redis to compensate for a missing database index."

5. "Multi-region multiplies everything — cost, complexity, and failure modes. I always ask: do we actually need multi-region writes, or is read-local-write-remote sufficient? The answer changes the cost by 3-5x."
