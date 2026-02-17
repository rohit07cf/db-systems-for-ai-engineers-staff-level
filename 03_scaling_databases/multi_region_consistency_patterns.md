# Multi-Region Consistency Patterns: From Theory to Production

## Why This Matters at Staff+ Level

Multi-region database deployments are where distributed systems theory meets business requirements
head-on. At the staff/principal level, you must translate abstract consistency models into concrete
architecture decisions: which data gets strong consistency (and pays the latency penalty), which
gets eventual consistency (and handles conflicts), and how consensus protocols actually behave
under partition. This is the domain where "it depends" must become a specific, defensible answer.

---

## 1. CAP Theorem in Practice

### The Actual Statement

During a network partition (P), a distributed system must choose between:
- **Consistency (C)**: Every read receives the most recent write
- **Availability (A)**: Every request receives a response (not an error)

When there is no partition, you can have both C and A.

```
                   Consistency
                      /\
                     /  \
                    / CP \
                   /______\
                  /\      /\
                 /  \    /  \
                / CA \  / AP \
               /______\/______\
         Availability    Partition
                         Tolerance

  CP Systems: Spanner, CockroachDB, etcd, ZooKeeper
  AP Systems: Cassandra, DynamoDB, Riak
  CA Systems: Single-node PostgreSQL (no partition = no choice needed)
```

### Why CAP Is Insufficient: PACELC

CAP only describes behavior during partitions. PACELC extends it:

**If Partition (P), choose Availability (A) or Consistency (C); Else (E), choose Latency (L) or Consistency (C).**

| System       | During Partition (PAC) | Normal Operation (ELC) | Classification |
|-------------|:----------------------:|:----------------------:|:--------------:|
| Spanner      | PC                     | EC                     | PC/EC          |
| CockroachDB  | PC                     | EC                     | PC/EC          |
| Cassandra     | PA                     | EL                     | PA/EL          |
| DynamoDB      | PA                     | EL (default)           | PA/EL          |
| MongoDB       | PC (w:majority)        | EC                     | PC/EC          |
| PostgreSQL    | PC                     | EC                     | PC/EC          |

**The staff-level insight:** The more important question is not "CAP" but "what do you choose
in the E (else, no partition) case?" because partitions are rare (minutes/year) but the
latency-consistency tradeoff affects every single request.

---

## 2. Consistency Models Spectrum

```
  Strongest ◄──────────────────────────────────────► Weakest

  Linearizable  Sequential  Causal  Session  Eventual
      │             │         │        │         │
      │             │         │        │         │
   "Behaves as    "All ops   "Cause   "Your    "All
    single copy"   in some    before   writes   replicas
                   total      effect"  visible  converge
                   order"              to you"  eventually"

  Latency cost:
  High ◄───────────────────────────────────────────► Low

  Availability:
  Lower ◄──────────────────────────────────────────► Higher
```

### Detailed Comparison

| Model          | Guarantee                                    | Cross-Region Latency  | Use Case              |
|----------------|----------------------------------------------|:---------------------:|-----------------------|
| Linearizable   | Real-time ordering of all operations         | 100-300ms (consensus) | Financial ledger      |
| Sequential     | All nodes see same operation order           | 50-200ms              | Distributed locks     |
| Causal         | Causally related ops ordered; concurrent free| 10-50ms               | Collaboration tools   |
| Session        | Read-your-writes within a session            | 5-20ms                | User-facing apps      |
| Eventual       | All replicas converge given no new writes    | <5ms (local read)     | Social media, metrics |

---

## 3. Spanner's TrueTime

Google Spanner achieves external consistency (linearizability) across global datacenters using
hardware-assisted time synchronization.

### How TrueTime Works

```
  TrueTime API:
    TT.now() -> [earliest, latest]    (not a single timestamp)
    Uncertainty interval: typically 1-7ms

  GPS receivers + atomic clocks in every datacenter:
  ┌───────────────────────────────────────────┐
  │  Datacenter                               │
  │  ┌─────────┐  ┌─────────┐  ┌──────────┐  │
  │  │ GPS     │  │ Atomic  │  │ GPS      │  │
  │  │ Clock 1 │  │ Clock   │  │ Clock 2  │  │
  │  └────┬────┘  └────┬────┘  └────┬─────┘  │
  │       └────────────┼───────────┘          │
  │              ┌─────▼──────┐               │
  │              │ Time Master│               │
  │              │ (marzipan) │               │
  │              └─────┬──────┘               │
  │                    │                      │
  │         ┌──────────▼──────────┐           │
  │         │  Spanserver Daemon  │           │
  │         │  TT.now() -> [e,l] │           │
  │         └─────────────────────┘           │
  └───────────────────────────────────────────┘
```

### Commit Wait Protocol

```
  Transaction T1 commits at timestamp s1:
    1. Acquire locks
    2. Choose commit timestamp s1 >= TT.now().latest
    3. WAIT until TT.now().earliest > s1  (commit wait)
    4. Release locks, apply write

  This ensures: if T2 starts after T1 commits (in real time),
  then T2's timestamp > T1's timestamp (guaranteed ordering).

  ┌──────────────────────────────────────────────────┐
  │  Timeline:                                       │
  │                                                  │
  │  T1: ──[lock]──[choose s1]──[WAIT]──[commit]──  │
  │                                  │               │
  │                            TT.now().earliest > s1│
  │                                  │               │
  │  T2:                             ──[start]──     │
  │                                                  │
  │  Guarantee: s2 > s1 (because T2 started after    │
  │  T1's commit wait completed)                     │
  └──────────────────────────────────────────────────┘

  Latency cost: commit wait = uncertainty interval (~1-7ms)
  This is why Spanner invests in GPS/atomic clocks: smaller
  uncertainty = shorter commit wait = lower latency
```

---

## 4. CockroachDB's Hybrid Logical Clocks (HLC)

CockroachDB achieves similar guarantees to Spanner without specialized hardware, using Hybrid
Logical Clocks.

### HLC Structure

```
  HLC timestamp = (physical_time, logical_counter)

  physical_time: wall clock (NTP synchronized)
  logical_counter: increments when physical_time ties

  Comparison: (pt1, lc1) < (pt2, lc2) iff
    pt1 < pt2, OR (pt1 == pt2 AND lc1 < lc2)
```

### Clock Skew Handling

```
  Node A (clock ahead by 50ms):  physical_time = 1000
  Node B (clock correct):        physical_time = 950

  When B receives message from A with timestamp 1000:
    B updates its HLC: max(950, 1000) + 1 = 1001
    B's future timestamps >= 1001

  Problem: NTP skew can be 100-250ms
  Spanner's uncertainty: 1-7ms (GPS/atomic)

  CockroachDB mitigation:
  - max_offset parameter (default 500ms)
  - If observed clock skew > max_offset, node shuts down
  - Read uncertainty window = [commit_timestamp, commit_timestamp + max_offset]
  - Reads may need to "restart" if they encounter writes in uncertainty window
```

### Tradeoff vs Spanner

| Aspect                  | Spanner (TrueTime)        | CockroachDB (HLC)           |
|-------------------------|--------------------------|------------------------------|
| Clock uncertainty       | 1-7ms (GPS/atomic)       | 100-500ms (NTP)              |
| Commit latency          | Low (short wait)         | Higher (larger uncertainty)  |
| Hardware requirement    | GPS receivers, atomic clocks | Standard servers          |
| Deployment              | Google Cloud only        | Any infrastructure           |
| Read restarts           | None                     | Possible in uncertainty window|
| Operational cost        | Higher (hardware)        | Lower (software-only)        |

---

## 5. Conflict Resolution Strategies

When replicas accept writes independently (AP systems), conflicts are inevitable.

### Last-Write-Wins (LWW)

```
  Region US:  SET key = "A"  at T=100
  Region EU:  SET key = "B"  at T=101

  Conflict resolution: T=101 > T=100, so "B" wins.

  Problem: If clocks are skewed, the "last" write may not be
  the causally latest. User in US may see their write overwritten
  by an earlier concurrent write from EU.

  ┌──────────────────────────────────────────┐
  │  Use LWW when:                           │
  │  - Data loss from conflicts is acceptable│
  │  - Operations are idempotent             │
  │  - Clock skew is bounded and small       │
  │  - Example: session data, preferences    │
  │                                          │
  │  Do NOT use LWW when:                    │
  │  - Conflicts represent real business     │
  │    conflicts (two users editing same doc)│
  │  - Data has financial implications       │
  └──────────────────────────────────────────┘
```

### Vector Clocks

Track causal history per replica to detect true conflicts (concurrent writes).

```
  Vector clock: [A:0, B:0, C:0] (one entry per replica)

  Replica A writes: VC = [A:1, B:0, C:0]
  Replica B writes: VC = [A:0, B:1, C:0]

  Neither dominates the other -> TRUE CONFLICT
  (A:1 > A:0 but B:0 < B:1)

  Resolution: application-level merge function
  Example: shopping cart -> union of items
  Example: text document -> operational transform / CRDT

  Replica A reads both:
    VC_A = [A:1, B:0, C:0]
    VC_B = [A:0, B:1, C:0]
    Merged: [A:1, B:1, C:0] with merged value
```

### CRDTs (Conflict-Free Replicated Data Types)

Data structures that mathematically guarantee convergence without coordination.

```
  G-Counter (Grow-only counter):
    Each replica maintains its own counter.
    Value = sum of all replica counters.
    Merge = element-wise max.

    Replica A: [A:5, B:0, C:0]  value = 5
    Replica B: [A:0, B:3, C:0]  value = 3
    Replica C: [A:0, B:0, C:7]  value = 7

    Merge: [A:5, B:3, C:7]  value = 15

  PN-Counter (increment/decrement):
    Two G-Counters: one for increments, one for decrements
    Value = sum(increments) - sum(decrements)

  LWW-Register:       Last-writer-wins for single values
  OR-Set:             Observed-Remove Set (add/remove elements)
  LWW-Element-Set:    Combination for practical use
```

### Conflict Resolution Decision Matrix

| Strategy       | Complexity | Data Loss Risk | Latency | Use Case                   |
|---------------|:----------:|:--------------:|:-------:|----------------------------|
| LWW           | Low        | High           | Lowest  | Caches, sessions           |
| Vector Clocks | Medium     | None (manual)  | Medium  | Shopping carts, documents  |
| CRDTs         | High       | None (auto)    | Lowest  | Counters, sets, flags      |
| App-level merge| Highest   | None (custom)  | Medium  | Domain-specific logic      |

---

## 6. Consensus Protocols

### Raft (Used by: etcd, CockroachDB, TiKV)

```
  Normal operation:
  ┌──────────┐     AppendEntries     ┌──────────┐
  │  Leader   │ ──────────────────── │ Follower │
  │  (Node A) │ ──────────────────── │ (Node B) │
  │           │ ──────────────────── │          │
  │           │                      │ (Node C) │
  └──────────┘                      └──────────┘

  Write committed when majority (2/3) acknowledge.

  Leader election (on leader failure):
  1. Follower times out waiting for heartbeat
  2. Becomes candidate, increments term, votes for self
  3. Requests votes from other nodes
  4. Wins if receives majority of votes
  5. Becomes new leader, begins accepting writes

  Term: 1          Term: 2
  Leader: A        Leader: B (after A fails)
  ───────────── X ─────────────────────
  A: [log][log]   B: [log][log][log]
  B: [log][log]   C: [log][log][log]
  C: [log][log]
```

### Multi-Paxos (Used by: Spanner, Chubby)

```
  Phase 1 (Prepare): Proposer sends prepare(n) to acceptors
  Phase 2 (Accept):  If majority promised, send accept(n, value)
  Phase 3 (Learn):   If majority accepted, value is chosen

  Multi-Paxos optimization:
  - Skip Phase 1 for subsequent rounds (leader lease)
  - Amortize leadership across many decisions
  - Practical throughput similar to Raft
```

### Raft vs Paxos Comparison

| Aspect               | Raft                          | Multi-Paxos                  |
|-----------------------|-------------------------------|------------------------------|
| Understandability     | Designed for clarity          | Notoriously complex          |
| Leader election       | Built-in, well-defined        | Separate protocol needed     |
| Log compaction        | Snapshotting                  | Implementation-dependent     |
| Performance           | Similar in practice           | Similar in practice          |
| Membership changes    | Joint consensus               | Varies by implementation     |
| Industry adoption     | etcd, CockroachDB, TiKV      | Spanner, Chubby, Megastore   |

---

## 7. Multi-Region Topology Patterns

### Pattern 1: Single Leader, Multi-Region Replicas

```
  ┌─────────────────┐
  │  US-EAST (Leader)│
  │  Reads + Writes  │
  └────────┬────────┘
           │ async replication
     ┌─────┼─────────────┐
     │     │             │
  ┌──▼──┐ ┌──▼──┐    ┌──▼──┐
  │EU-W │ │AP-SE│    │US-W │
  │Read │ │Read │    │Read │
  │Only │ │Only │    │Only │
  └─────┘ └─────┘    └─────┘

  Write latency: low (local to US-EAST)
  Read latency: low (local replicas)
  Cross-region writes: high (must go to US-EAST)
  Failover: promote replica to leader (manual or automated)
```

### Pattern 2: Multi-Leader (Active-Active)

```
  ┌─────────────┐         ┌─────────────┐
  │  US-EAST    │◄───────►│  EU-WEST    │
  │  Leader     │  async  │  Leader     │
  │  Read+Write │  repl   │  Read+Write │
  └──────┬──────┘         └──────┬──────┘
         │                       │
         └───────────┬───────────┘
                     │
              ┌──────▼──────┐
              │  AP-SOUTH   │
              │  Leader     │
              │  Read+Write │
              └─────────────┘

  Write latency: low (local leader)
  Read latency: low (local leader)
  Conflict rate: proportional to concurrent writes to same key
  Complexity: HIGH (conflict resolution required)
```

### Pattern 3: Partitioned Leaders (Spanner-style)

```
  Data partitioned by region; each partition has a Paxos group.

  US user data:  Leader in US-EAST,  replicas in US-WEST, EU-WEST
  EU user data:  Leader in EU-WEST,  replicas in EU-EAST, US-EAST
  AP user data:  Leader in AP-SOUTH, replicas in AP-EAST, US-WEST

  ┌──────────────────────────────────────────────────┐
  │          US-EAST          EU-WEST    AP-SOUTH    │
  │  US data: [LEADER]        [replica]  [replica]   │
  │  EU data: [replica]       [LEADER]   [replica]   │
  │  AP data: [replica]       [replica]  [LEADER]    │
  └──────────────────────────────────────────────────┘

  User reads/writes to their data: local leader (low latency)
  Cross-region data access: routed to remote leader (high latency)
  No conflicts: single leader per partition
```

---

## 8. Practical Patterns by Domain

### Financial Systems

```
  Requirement: Strong consistency, no lost writes, audit trail
  Pattern: Synchronous replication with Raft consensus
  Database: Spanner / CockroachDB
  Consistency: Serializable isolation, linearizable reads
  Latency budget: 200-500ms per transaction (acceptable for payments)
  Conflict strategy: Not applicable (single leader per partition)

  Architecture:
  - Primary region: handles all writes for a currency/jurisdiction
  - Witness region: participates in consensus but serves no reads
  - Read region: serves stale reads for dashboards (async replica)
```

### Social Media / Content Platform

```
  Requirement: Low latency, high availability, eventual consistency OK
  Pattern: Multi-leader with LWW or CRDTs
  Database: Cassandra / DynamoDB Global Tables
  Consistency: Eventual (reads), local quorum (writes)
  Latency budget: <50ms for reads, <100ms for writes
  Conflict strategy: LWW for posts, CRDT counters for likes/views

  Architecture:
  - Every region is active (reads + writes)
  - Async replication between regions
  - Accept that a like count may be off by a few for seconds
```

### AI/ML Feature Store

```
  Requirement: Low-latency reads, batch writes, versioned features
  Pattern: CQRS with time-versioned read model
  Database: Write path (Kafka -> Iceberg), Read path (Redis / DynamoDB)
  Consistency: Eventual (features lag behind by seconds to minutes)
  Latency budget: <5ms for feature reads during inference
  Conflict strategy: Version-based (latest feature version wins)

  Architecture:
  - Feature computation pipeline writes to central store
  - Each region has a local read cache populated via CDC
  - Model serving reads from local cache (never cross-region)
```

---

## 9. Failure Mode Analysis

| Failure Scenario                | CP System Response                  | AP System Response              |
|---------------------------------|------------------------------------|---------------------------------|
| Network partition (1 region)    | Minority side unavailable          | All regions serve (diverge)     |
| Leader node crash               | Election (~1-5s downtime)          | Other replicas continue         |
| Clock skew > threshold          | Node self-quarantines (CockroachDB)| LWW may resolve incorrectly    |
| Replication lag spike           | Reads block until caught up        | Reads return stale data         |
| Split-brain (network healed)    | Log reconciliation (automatic)     | Conflict resolution needed      |
| Full region outage              | Failover to surviving regions      | Surviving regions continue      |

---

## Interview Narrative

**When asked about multi-region consistency, structure your answer as follows:**

> "I start by classifying the data into consistency tiers, because different data within the same
> system has different consistency requirements. Financial transactions need linearizability.
> User preferences need session consistency. Like counts need eventual consistency. Applying one
> consistency model to all data either wastes latency or risks correctness.
>
> For data that requires strong consistency across regions, I use a partitioned-leader model like
> Spanner's: each data partition has a Raft/Paxos group, and the leader is placed in the region
> closest to the majority of that partition's users. Cross-region writes pay the consensus latency
> (100-300ms round-trip), but that is acceptable for operations like balance transfers. The key
> enabler is tight clock synchronization -- Spanner uses GPS/atomic clocks to bound uncertainty
> to 7ms; CockroachDB uses NTP with a configurable max offset.
>
> For data that tolerates eventual consistency, I use multi-leader replication with CRDTs for
> data types that support them (counters, sets) and last-write-wins for simple values. The critical
> design decision is conflict detection vs conflict avoidance. I prefer conflict avoidance through
> data modeling: if each user's data is primarily written by that user, conflicts are rare. For
> shared mutable state (collaborative editing), I invest in CRDTs or operational transform.
>
> The PACELC framework is more useful than CAP in practice: even without partitions, every
> cross-region read forces a choice between latency (read locally, possibly stale) and consistency
> (read from leader, higher latency). I make this choice per-query, not per-system, using
> consistency levels in the query API.
>
> For AI/ML workloads specifically, feature stores are almost always eventually consistent -- the
> model serving path reads from a local cache, and feature freshness of a few seconds is
> acceptable. The training pipeline uses batch reads that can tolerate minutes of staleness. This
> means I can optimize the feature store entirely for read latency without cross-region
> coordination."

**Follow-up traps to prepare for:**

- "How does Spanner achieve linearizability without a single leader?" (Answer: each partition HAS
  a single leader; Spanner's innovation is using TrueTime to order transactions across different
  partitions' leaders without cross-partition coordination)
- "What happens to CRDTs when the network heals after a long partition?" (Answer: CRDTs merge
  deterministically; the merge function is commutative, associative, and idempotent, so order
  of merge does not matter -- convergence is guaranteed)
- "Can you have causal consistency across regions?" (Answer: yes, using causal dependency tracking
  via vector clocks or hybrid logical clocks; Cosmos DB offers a session consistency mode that
  approximates this)
