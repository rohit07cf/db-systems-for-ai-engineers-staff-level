# Multi-Region Database Patterns -- Quick Reference

## Pattern Comparison Table

| Pattern | Consistency | Write Latency | Read Latency | Complexity | Data Loss Risk | Use Case |
|---|---|---|---|---|---|---|
| **Single-leader (primary region)** | Strong | High (cross-region) | Low (local reads if stale ok) | Low | RPO = repl lag | Most applications, strong consistency needed |
| **Multi-leader (active-active)** | Eventual | Low (local writes) | Low | High | RPO ~ 0 (conflict risk) | Global writes required, conflict-tolerant |
| **Leaderless (Dynamo-style)** | Tunable | Low (quorum local) | Low-Medium | Medium | Depends on quorum config | AP-first systems, high availability |
| **Read replicas (fan-out)** | Eventual | High (single writer) | Low | Low | RPO = repl lag | Read-heavy, writes centralized |
| **Partitioned by region** | Strong (per region) | Low | Low | Medium | Regional RPO = 0 | Data residency / GDPR, geo-local data |
| **Consensus-based (Spanner)** | Strong (global) | Medium (TrueTime) | Medium | Very High | RPO = 0 | Global strong consistency required |

## Consistency Model Reference

| Model | Guarantee | Real-World Example |
|---|---|---|
| **Linearizability** | Reads see latest write, globally ordered | Google Spanner (TrueTime), CockroachDB |
| **Sequential consistency** | All nodes see same order (may lag real-time) | ZooKeeper |
| **Causal consistency** | Causally related ops seen in order | MongoDB (causal sessions), Cosmos DB |
| **Session consistency** | Client sees its own writes | Azure Cosmos DB (default) |
| **Bounded staleness** | Reads lag by at most T seconds or K versions | Cosmos DB (bounded staleness tier) |
| **Eventual consistency** | All replicas converge eventually | DynamoDB (default), Cassandra (ONE) |

```
Strongest  <----------------------------------------------->  Weakest
Linearizable > Sequential > Causal > Session > Bounded Staleness > Eventual

Availability increases --->
Latency decreases     --->
```

> **CAP Interview Frame**: "In a network partition, we choose AP (available + partition-tolerant)
> or CP (consistent + partition-tolerant). In practice, we tune the spectrum using
> consistency levels, not a binary switch."

## Latency Numbers to Know

| Path | Latency |
|---|---|
| Same datacenter (same AZ) | **0.5 -- 1 ms** |
| Cross-AZ (same region) | **1 -- 3 ms** |
| US East <-> US West | **60 -- 80 ms** |
| US <-> Europe | **80 -- 120 ms** |
| US <-> Asia (Tokyo) | **150 -- 200 ms** |
| US <-> Australia | **180 -- 250 ms** |
| Round-the-world | **300 -- 400 ms** |
| Synchronous replication (cross-region) | **2x one-way latency** (write + ack) |
| Spanner commit (global) | **10 -- 20 ms** (TrueTime + Paxos) |

**Key insight**: Synchronous cross-region replication US-East <-> EU adds ~200 ms
to every write. This is why most systems choose async replication with conflict resolution.

## Conflict Resolution Strategy Comparison

| Strategy | How It Works | Pros | Cons | Best For |
|---|---|---|---|---|
| **Last Writer Wins (LWW)** | Highest timestamp wins | Simple, no coordination | Data loss (silent), clock skew | Caches, non-critical data |
| **Version Vectors** | Track causal history per replica | Detects true conflicts | Space overhead, complex merge | Shopping carts, collaborative |
| **CRDTs** | Mathematically guaranteed convergence | No coordination needed | Limited data types, space overhead | Counters, sets, registers |
| **Application-level merge** | Custom merge function per data type | Correct for domain | Engineering cost, complexity | Domain-specific (e.g., documents) |
| **Conflict-free by design** | Append-only / immutable data model | No conflicts possible | May not fit all use cases | Event sourcing, ledgers |
| **Consensus (Paxos/Raft)** | Agree before commit | No conflicts, strong consistency | High latency across regions | Financial, inventory |

### CRDT Quick Reference

| CRDT Type | Operation | Example Use |
|---|---|---|
| G-Counter | Increment only | Page view counts |
| PN-Counter | Increment / decrement | Like count (likes - unlikes) |
| G-Set | Add only | Tags applied to entity |
| OR-Set | Add / remove (observed-remove) | Shopping cart items |
| LWW-Register | Read / write (last writer wins) | User profile fields |
| MV-Register | Multi-value on conflict | Collaborative editing hints |

## Architecture Pattern Selection Guide

```
Do writes MUST happen in all regions?
     |                    |
    YES                   NO
     |                    |
     v                    v
Can you tolerate         Single-leader + read replicas
conflict resolution?     (route writes to primary)
     |           |
    YES          NO
     |           |
     v           v
Multi-leader    Need global strong consistency?
(CRDTs,LWW,         |              |
app-merge)          YES             NO
                     |              |
                     v              v
              Spanner/CockroachDB  Partition by region
              (consensus-based)    (data stays local)
```

## Database Multi-Region Capabilities

| Database | Multi-Region Mode | Consistency | Conflict Resolution |
|---|---|---|---|
| **Spanner** | Global (Paxos) | Linearizable | Consensus (no conflicts) |
| **CockroachDB** | Global (Raft) | Serializable | Consensus (no conflicts) |
| **DynamoDB Global Tables** | Active-active | Eventual | LWW (auto) |
| **Cosmos DB** | Multi-write regions | 5 consistency levels | LWW / custom |
| **Cassandra** | Multi-DC | Tunable (LOCAL_QUORUM) | LWW (timestamps) |
| **MongoDB** | Atlas Global Clusters | Causal (sessions) | Single writer (zone-sharding) |
| **PostgreSQL** | BDR / Citus | Eventual (BDR) | App-level / LWW (BDR) |
| **MySQL** | Group Replication / InnoDB Cluster | Eventual (async) / Strong (group) | Single-primary or LWW |
| **Redis Enterprise** | Active-Active (CRDB) | Eventual | CRDTs |
| **TiDB** | Placement rules | Raft-based | Consensus (no conflicts) |
| **YugabyteDB** | xCluster / geo-partitioning | Linearizable (sync) | Consensus / async LWW |

## Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Region outage | Writes/reads fail in that region | Failover to secondary leader; DNS/LB reroute |
| Network partition between regions | Split-brain (multi-leader) | Quorum-based writes; fencing tokens |
| Replication lag spike | Stale reads, potential conflicts | Monitor lag; circuit-break stale reads |
| Clock skew across regions | LWW gives wrong result | Use hybrid logical clocks (HLC) or TrueTime |
| Failover cascading load | Surviving region overwhelmed | Pre-provision capacity headroom (N+1 sizing) |
| DNS propagation delay | Clients still route to failed region | Low TTL (30-60s); client-side health checks |

## RPO / RTO Quick Reference

| Strategy | RPO (Data Loss) | RTO (Downtime) |
|---|---|---|
| Synchronous replication | **0** | Minutes (failover time) |
| Async replication (1s lag) | **~1 second** | Minutes |
| Async replication (1min lag) | **~1 minute** | Minutes |
| Daily backups | **Up to 24 hours** | Hours |
| Multi-leader active-active | **~0** (conflicts possible) | **~0** (no failover needed) |
| Consensus-based (Spanner) | **0** | **~0** (automatic leader election) |

## Cost Considerations

| Component | Relative Cost | Notes |
|---|---|---|
| Cross-region data transfer | $$$ | $0.02-0.09/GB between regions (cloud) |
| Synchronous replication | $$$$ | Doubles write path; adds latency = lower throughput |
| Multi-region compute | $$ per region | N regions = ~Nx compute cost |
| Global load balancing | $ | Relatively cheap vs. other components |
| Conflict resolution engineering | $$$$$ | Months of engineering for custom resolution |

**Quick math**: 1 TB/day cross-region replication at $0.05/GB = **$50/day = $1,500/month** per region pair.

## Multi-Region DNS & Traffic Routing

| Strategy | How It Works | Failover Speed | Use Case |
|---|---|---|---|
| **DNS-based (Route 53)** | Geo/latency routing policies | 30-60s (TTL) | Simple geo-routing |
| **Global Load Balancer** | Anycast IP, health checks | 10-30s | Low-latency failover |
| **Client-side routing** | SDK chooses region, client health checks | < 5s | Latency-critical apps |
| **Service mesh (Istio)** | Locality-aware routing | Seconds | Kubernetes-native |

## Data Residency & Compliance Quick Reference

| Regulation | Region Constraint | Key Requirement |
|---|---|---|
| **GDPR** (EU) | EU residents' data in EU | Right to deletion, consent, DPA |
| **CCPA** (California) | No strict residency | Disclosure, opt-out of sale |
| **PIPL** (China) | Data localization required | Government security assessment |
| **PDPA** (Singapore) | No strict residency | Consent, cross-border transfer rules |
| **LGPD** (Brazil) | No strict residency | Consent, DPO appointment |

> **Architectural implication**: Region-pinned partitions for regulated data,
> globally replicated partitions for non-regulated (product catalog, configs).

## Key Talking Points for Interviews

1. **"Start single-region, design for multi-region."** Use region-agnostic IDs, avoid
   auto-increment, plan schema for eventual partitioning.

2. **"Active-active is not free."** You trade consistency complexity for write latency.
   Quantify: "We accepted LWW with ~500ms convergence to avoid 150ms write latency penalty."

3. **"Partition by region when data is naturally geo-local."** User data in EU stays in EU
   (GDPR). Only cross-reference data (e.g., global product catalog) gets replicated everywhere.

4. **"Measure replication lag as a primary SLI."** Alert on lag > acceptable staleness window.
   If lag exceeds SLA, route reads to the leader.

5. **"N+1 regional capacity."** If you have 3 regions, each must handle ~50% of total traffic
   so that losing one region doesn't overwhelm the remaining two.

6. **"Conflict resolution is a product decision, not just engineering."**
   The business must define which write wins in a conflict, not the database team alone.

7. **"Test region failover quarterly."** Netflix Chaos Monkey principle -- if you haven't
   tested failover, you don't have failover.
