# How Staff Engineers Think About Databases

## Executive Summary (for Staff Interview)

- Staff engineers choose databases based on **failure modes**, not feature lists
- Every database decision is a **10-year commitment** — migration cost is the hidden variable
- The right question is never "which DB is best?" but "what are we optimizing for, and what are we willing to sacrifice?"
- Staff engineers think in **SLOs first**, then work backward to storage architecture
- Operational cost (on-call burden, upgrade path, talent availability) often outweighs raw performance

## Architectural Tradeoffs

| Decision | Pros | Cons | At Small Scale | At 10x Scale |
|---|---|---|---|---|
| Use managed service (RDS, DynamoDB) | Zero ops, automatic patching | Vendor lock-in, cost ceiling | Always correct | Cost explodes, need custom tuning |
| Self-host (PostgreSQL, Cassandra) | Full control, cost optimization | Ops burden, upgrade risk | Rarely worth it | Often necessary for customization |
| Single database for everything | Simple ops, one backup strategy | Scaling ceiling, workload interference | Perfect | Breaks — reads starve writes |
| Polyglot persistence | Right tool for each job | Operational complexity, consistency gaps | Over-engineered | Necessary and expected |
| Strong consistency | Correctness guarantees | Latency penalty, availability risk | Easy default | Cross-region becomes painful |
| Eventual consistency | Low latency, high availability | App complexity for conflict handling | Unnecessary complexity | Mandatory for global scale |

## Deep Technical Internals

**The Staff Engineer's Mental Model:**

```
Business Requirement
    |
    v
SLO Definition (p99 latency, availability, durability)
    |
    v
Access Pattern Analysis (read:write ratio, query shapes, data locality)
    |
    v
Data Model Selection (relational, document, key-value, graph, vector)
    |
    v
Consistency Requirement (strong, eventual, causal, session)
    |
    v
Scaling Strategy (vertical, horizontal, read replicas, sharding)
    |
    v
Operational Model (managed, self-hosted, hybrid)
    |
    v
Cost Model ($ per query, $ per GB, $ per ops engineer)
```

**What separates Staff from Senior:**

- Senior: "I'll use PostgreSQL because I know it well"
- Staff: "PostgreSQL gives us ACID guarantees we need for payment processing, but we'll need to shard at ~5TB. I'd rather accept that migration cost in 2 years than add DynamoDB complexity now, because our team has zero DynamoDB expertise and on-call would suffer"

**Key thinking patterns:**

- **Blast radius thinking**: What happens when this database goes down? How many services are affected?
- **Migration cost awareness**: How hard is it to move off this choice in 3 years?
- **Talent market reality**: Can we hire people who know this system?
- **Failure mode preference**: I'd rather have slow responses than wrong responses (or vice versa)
- **Cost trajectory modeling**: This is cheap now, but what's the cost at 10x data?

## Failure Modes at Scale

| What Breaks | Why | How to Detect | Mitigation |
|---|---|---|---|
| Connection pool exhaustion | Too many microservices, each holding connections | Connection count metrics, p99 latency spike | PgBouncer, connection limits, service consolidation |
| Replication lag | Write volume exceeds replica apply rate | Replica lag metric, stale read complaints | Read from primary for critical paths, throttle writes |
| Lock contention | Hot rows/tables under concurrent writes | Lock wait time metrics, slow query log | Optimistic locking, queue writes, partition hot data |
| Storage IOPS ceiling | More data than disk can serve | IOPS utilization metric, latency cliff | Tiered storage, caching layer, archive cold data |
| Schema migration failure | Large table ALTER in production | Migration monitoring, table size tracking | Online DDL tools (gh-ost, pt-osc), blue-green |

## Multi-Region Considerations

- **First question**: Do users in region B need to write, or just read?
  - Read-only remote: Simple replication, stale reads acceptable
  - Read-write remote: Conflict resolution is now your problem
- **Latency math**: US-East to EU-West ~80ms, US-East to AP-Southeast ~200ms
  - Synchronous cross-region replication adds this to every write
  - At 10K writes/sec, that's 10K * 200ms = queueing theory nightmare
- **Data residency**: GDPR, data sovereignty laws constrain where data can live
  - This is an architectural constraint, not a nice-to-have
- **Failover testing**: If you haven't tested regional failover, you don't have regional failover

## Cost Model Thinking

| Cost Driver | What People Underestimate |
|---|---|
| Storage | Not the disk — it's the IOPS. EBS gp3 is cheap; io2 is 10x more |
| Compute | Not the CPU — it's the memory for caching/indexes |
| Network | Cross-AZ: $0.01/GB. Cross-region: $0.02-0.09/GB. Replication is network |
| Operations | One full-time DBA costs ~$200K/yr. Managed service might be cheaper |
| Migrations | Zero-downtime migration of 10TB takes months of engineering time |
| Opportunity cost | Time spent fighting the wrong DB is time not building features |

## Interview Narrative

**"How I would explain this in a Staff interview"**

1. "I start every database decision with the SLO. If we need p99 < 10ms reads, that eliminates most disk-based solutions and points us toward in-memory stores or well-cached systems."

2. "I evaluate databases on their failure modes, not their happy path. Every system is fast when it works — I need to know what happens when a node dies, when replication lags, when we hit a hot partition."

3. "I always model the cost trajectory. A database that costs $5K/month at launch might cost $500K/month at 100x scale. I need to understand the cost curve shape, not just the starting point."

4. "Operational complexity is a first-class metric. If my team has never run Cassandra, choosing it over DynamoDB adds 6 months of learning curve and increases on-call burden significantly."

5. "I think about blast radius. If this database serves 15 microservices and goes down, that's a company-wide outage. I'd rather have two smaller databases with independent failure domains."

6. "Migration cost is the hidden variable. Choosing a proprietary database (DynamoDB, Spanner) means high performance now but expensive migration later. I make that tradeoff explicitly, not accidentally."

7. "At Staff level, I don't just pick the database — I design the data architecture. That means defining consistency boundaries, caching strategies, replication topologies, and operational runbooks before writing any code."
