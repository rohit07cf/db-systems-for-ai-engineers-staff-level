# Tradeoff Frameworks for Database Architecture

## Executive Summary (for Staff Interview)

- Every database decision is a tradeoff — the job is making the tradeoff **explicit and documented**
- CAP theorem is table stakes; real tradeoffs are PACELC, consistency vs cost, and operational complexity vs performance
- Staff engineers use **decision matrices** and **reversibility analysis**, not gut feeling
- The best architecture is the one where the tradeoffs align with business priorities
- "It depends" is not an answer — "It depends on X, and here's how I'd decide" is

## Architectural Tradeoffs

| Decision | Pros | Cons | At Small Scale | At 10x Scale |
|---|---|---|---|---|
| Optimize for read latency | Fast user experience | Write amplification, index overhead | Simple caching works | Need read replicas, CDN, denormalization |
| Optimize for write throughput | High ingest rate | Read complexity, eventual consistency | Append-only log is enough | LSM trees, partitioning, async replication |
| Normalize data | Data integrity, no duplication | Join cost at scale | Always correct | Joins become bottleneck — denormalize hot paths |
| Denormalize data | Fast reads, no joins | Write complexity, consistency risk | Premature optimization | Necessary for sub-ms reads |
| Single leader | Simple consistency model | Write bottleneck, single region | Perfect | Multi-region writes impossible |
| Multi-leader | Multi-region writes | Conflict resolution complexity | Over-engineered | Required for global apps |

## Deep Technical Internals

### Framework 1: PACELC (Beyond CAP)

```
If there is a Partition (P):
    Choose Availability (A) or Consistency (C)
Else (E) — when system is running normally:
    Choose Latency (L) or Consistency (C)

Examples:
- DynamoDB:  PA/EL  (available during partition, low latency normally)
- Spanner:   PC/EC  (consistent always, at cost of latency)
- Cassandra:  PA/EL  (tunable, but defaults to availability)
- PostgreSQL: PC/EC  (single leader, strong consistency)
```

### Framework 2: The 5-Dimension Tradeoff Space

| Dimension | Spectrum | Tension |
|---|---|---|
| **Consistency** | Strong ↔ Eventual | Latency and availability |
| **Latency** | Sub-ms ↔ Seconds | Cost (memory vs disk) |
| **Durability** | Sync flush ↔ Async write | Write throughput |
| **Availability** | 99.999% ↔ 99.9% | Consistency and cost |
| **Cost** | Cheap ↔ Expensive | Everything else |

You cannot optimize all five. Pick three. Sacrifice two.

### Framework 3: Reversibility Analysis

| Decision Type | Reversibility | Strategy |
|---|---|---|
| Adding a cache layer | High (remove it) | Try it, measure, adjust |
| Choosing SQL vs NoSQL | Low (data model change) | Invest heavily in analysis |
| Sharding strategy | Very Low (data redistribution) | Get it right or plan for migration |
| Cloud provider for managed DB | Low (vendor lock-in) | Abstract the interface layer |
| Adding a read replica | High (remove it) | Do it when needed |
| Schema design | Medium (migrations possible) | Plan for evolution |

**Rule**: Spend decision-making time proportional to irreversibility.

### Framework 4: The "What Breaks at 10x" Test

For every architectural choice, ask:

```
Current:  1K requests/sec, 100GB data, 1 region
At 10x:   10K requests/sec, 1TB data, still 1 region?
At 100x:  100K requests/sec, 10TB data, 3 regions
At 1000x: 1M requests/sec, 100TB data, 5 regions

For each level:
1. Does the data model still work?
2. Does the query pattern still work?
3. Does the consistency model still work?
4. Does the cost model still work?
5. Does the operational model still work?
```

### Framework 5: SLO-Driven Architecture

```
Step 1: Define SLOs
  - p50 read latency < 5ms
  - p99 read latency < 50ms
  - p99 write latency < 100ms
  - Availability: 99.95%
  - Data durability: 99.999999999%

Step 2: Derive architectural constraints
  - p50 < 5ms → Must serve from memory (cache or in-memory DB)
  - p99 < 50ms → Cannot cross regions for reads
  - p99 write < 100ms → Async replication OK (no cross-region sync)
  - 99.95% availability → Single-region OK with multi-AZ
  - 11 nines durability → Need replication factor ≥ 3

Step 3: Select and validate
  - Redis for hot reads (p50 < 1ms)
  - PostgreSQL for writes (p99 ~ 20ms local)
  - Async replication to read replicas
  - Multi-AZ deployment for availability
```

## Failure Modes at Scale

| Tradeoff Gone Wrong | Symptom | Root Cause | Fix |
|---|---|---|---|
| Over-normalized at scale | p99 reads >500ms | 7-way joins on hot path | Denormalize read model, CQRS |
| Over-denormalized | Inconsistent data, bugs | Write paths diverge | Event sourcing, single write path |
| Wrong consistency level | Lost writes, duplicate charges | Eventual consistency for payments | Strong consistency for money paths |
| Over-cached | Serving stale data hours old | No cache invalidation strategy | TTL + event-driven invalidation |
| Under-cached | DB overloaded | Every read hits primary | Add read-through cache, measure hit rate |

## Multi-Region Considerations

**The fundamental multi-region tradeoff:**

```
                    CONSISTENCY
                        |
                        |
           Spanner ●    |
                        |
        CockroachDB ●   |
                        |
    ────────────────────+──────────────── LATENCY
                        |
            Cassandra ● |
                        |
         DynamoDB GT ●  |
                        |
                    AVAILABILITY
```

- You can move along this triangle but never escape it
- Business requirements determine your position
- Different data types can live at different positions

## Cost Model Thinking

**Cost of getting tradeoffs wrong:**

| Wrong Tradeoff | Cost Impact |
|---|---|
| Strong consistency when eventual is fine | 3-5x latency, 2-3x infra cost |
| Eventual consistency when strong is needed | Data loss, customer trust, legal liability |
| Premature sharding | 2-4x operational cost for no benefit |
| Late sharding | Emergency migration under load, downtime risk |
| Over-indexing | 2-5x storage cost, slower writes |
| Under-indexing | Full table scans, p99 latency explosion |

## Interview Narrative

**"How I would explain this in a Staff interview"**

1. "I use a structured framework for database tradeoffs. First, I define the SLOs — what does the business actually need? Then I map those SLOs to architectural constraints. A p99 of 10ms means something very different from a p99 of 500ms."

2. "I think about reversibility. If a decision is easy to undo (adding a cache), I bias toward action. If it's hard to undo (sharding strategy), I invest in analysis upfront."

3. "PACELC is more useful than CAP in practice. CAP only talks about partition scenarios. PACELC tells me what happens during normal operation too — and normal operation is 99.99% of the time."

4. "I apply the '10x test' to every design. If we're at 1TB now, does this design work at 10TB? At 100TB? Where exactly does it break, and what's the migration plan?"

5. "The most expensive tradeoff mistakes aren't technical — they're organizational. Choosing a system nobody knows how to operate costs more than the 'wrong' database that your team can run well."

6. "I document tradeoffs as ADRs (Architecture Decision Records). When I choose eventual consistency for a read path, I write down why, what the alternatives were, and what would trigger reconsidering."
