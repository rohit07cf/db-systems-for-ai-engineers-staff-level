# Multi-Tenant Vector Database Design

## Staff/Principal Level — Database Systems for AI Engineers

---

## 1. Why Multi-Tenancy Is Hard for Vector Databases

Unlike traditional databases where tenant isolation is achieved through row-level security
or schema separation, vector databases face unique challenges:

- **Index structures are global:** HNSW graphs and IVF centroids span all vectors — you
  cannot trivially partition an HNSW graph by tenant without destroying its navigability.
- **Memory is the bottleneck:** Each tenant's vectors compete for the same RAM.
- **Noisy neighbors affect recall, not just latency:** A large tenant's vectors can dominate
  ANN traversal, reducing recall for small tenants.
- **Metadata filtering is expensive:** Per-tenant filtering during search can degrade recall
  by 10-30% depending on the filtering strategy.

```
  The Multi-Tenancy Spectrum:
  ──────────────────────────────────────────────────────────────
  Shared Everything          Shared Index,           Dedicated
  (single collection,        Separate Storage        Collections
   metadata filter)          (namespace/partition)    (per tenant)
  ──────────────────────────────────────────────────────────────
  Cheapest                                           Most Expensive
  Worst isolation                                    Best isolation
  Hardest to scale per tenant                        Easiest
  ──────────────────────────────────────────────────────────────
```

---

## 2. Isolation Strategy Comparison

### 2.1 Strategy Overview

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                                                                 │
  │  Strategy A: Shared Collection + Metadata Filter                │
  │  ┌───────────────────────────────────────────────────────────┐  │
  │  │                  Single HNSW Index                        │  │
  │  │  [T1:v1] [T2:v1] [T1:v2] [T3:v1] [T2:v2] [T1:v3] ...  │  │
  │  │                                                           │  │
  │  │  Query: search(q, filter={tenant: "T1"})                 │  │
  │  └───────────────────────────────────────────────────────────┘  │
  │                                                                 │
  │  Strategy B: Namespace / Partition per Tenant                   │
  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           │
  │  │ Namespace T1 │ │ Namespace T2 │ │ Namespace T3 │           │
  │  │ [v1][v2][v3] │ │ [v1][v2]     │ │ [v1]         │           │
  │  │ Shared infra │ │ Shared infra │ │ Shared infra │           │
  │  └──────────────┘ └──────────────┘ └──────────────┘           │
  │  Separate indexes, shared compute pool                         │
  │                                                                 │
  │  Strategy C: Dedicated Collection per Tenant                    │
  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           │
  │  │ Collection T1│ │ Collection T2│ │ Collection T3│           │
  │  │ Own index    │ │ Own index    │ │ Own index    │           │
  │  │ Own replicas │ │ Own replicas │ │ Own replicas │           │
  │  └──────────────┘ └──────────────┘ └──────────────┘           │
  │  Full isolation, highest cost                                   │
  │                                                                 │
  └─────────────────────────────────────────────────────────────────┘
```

### 2.2 Detailed Comparison

| Dimension | Shared + Filter | Namespace/Partition | Dedicated Collection |
|---|---|---|---|
| **Max tenants** | 10,000+ | 1,000-10,000 | 10-100 |
| **Isolation level** | None (shared index) | Logical (separate index) | Physical (separate resources) |
| **Recall impact** | Filter reduces recall 5-30% | No cross-tenant interference | Perfect isolation |
| **Memory overhead** | O(1) index | O(T) indexes, shared pool | O(T) full deployments |
| **Scaling per tenant** | Cannot scale individually | Limited (partition resize) | Full independent scaling |
| **Noisy neighbor risk** | HIGH | MEDIUM | NONE |
| **Operational complexity** | LOW | MEDIUM | HIGH |
| **Cost per tenant** | ~$0.001-0.01/mo | ~$0.10-1.00/mo | ~$10-100/mo |
| **Tenant onboarding** | Instant (insert vectors) | Fast (create namespace) | Slow (provision infra) |

---

## 3. Metadata Filtering Approaches

### 3.1 The Filtering Problem

```
  Scenario: 1000 tenants, 100M total vectors, query for tenant_42 (has 50K vectors)

  Post-filtering:
    1. ANN search returns top-1000 candidates (from all tenants)
    2. Filter to tenant_42
    3. Maybe only 5 results survive → UNDER-FETCHING

  Pre-filtering:
    1. Filter to tenant_42's 50K vectors
    2. ANN search on 50K vectors
    3. Problem: ANN index was built on 100M vectors, not 50K
       → Index structure doesn't align with filtered subset

  In-filter (integrated):
    1. During HNSW traversal, skip non-tenant_42 nodes
    2. Expand search beam to compensate for skipped nodes
    3. Best recall, but complex implementation
```

### 3.2 Filter Strategy Decision Matrix

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  Tenant's share of total vectors (selectivity):          │
  │                                                          │
  │  > 10% ──→ Post-filtering works fine                     │
  │             (enough candidates survive filtering)        │
  │                                                          │
  │  1-10% ──→ Pre-filtering with over-fetch (3-5x)         │
  │             or in-filter search                          │
  │                                                          │
  │  < 1% ───→ MUST use dedicated partition/namespace        │
  │             Shared index + filter will fail               │
  │             (recall < 0.5 in worst case)                 │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

### 3.3 Quantifying Filter Impact on Recall

```
  Empirical model for post-filtering recall degradation:

  effective_recall ≈ base_recall * (selectivity)^α

  where:
    selectivity = tenant_vectors / total_vectors
    α ≈ 0.3-0.5 (empirical, depends on data distribution)

  Example:
    base_recall = 0.95, selectivity = 0.01 (1% of vectors)
    effective_recall ≈ 0.95 * 0.01^0.4 ≈ 0.95 * 0.06 ≈ 0.06

    → CATASTROPHIC. Only 6% recall for a small tenant.

  Mitigation: Over-fetch by 1/selectivity factor:
    Fetch top-(k / selectivity) candidates, then filter.
    For k=10, selectivity=0.01: fetch top-1000, filter to 10.
    But: O(1/selectivity) cost increase.
```

---

## 4. Per-Tenant Collections vs Shared Collections

### 4.1 Architecture Patterns

```
  Pattern 1: "Collection Per Tenant" (Enterprise SaaS)
  ───────────────────────────────────────────────────────
  ┌────────────┐  ┌────────────┐  ┌────────────┐
  │ Tenant A   │  │ Tenant B   │  │ Tenant C   │
  │ Collection │  │ Collection │  │ Collection │
  │ 10M vecs   │  │ 500K vecs  │  │ 2M vecs    │
  │ HNSW       │  │ HNSW       │  │ HNSW       │
  │ 3 replicas │  │ 1 replica  │  │ 2 replicas │
  └────────────┘  └────────────┘  └────────────┘

  Pros: Perfect isolation, per-tenant SLA, independent scaling
  Cons: O(T) collections, resource waste for small tenants,
        complex orchestration, slow tenant provisioning

  When to use: < 100 tenants, large tenants (>1M vectors each),
               strict SLA requirements, enterprise pricing tier


  Pattern 2: "Shared Collection + Tenant Field" (Consumer SaaS)
  ────────────────────────────────────────────────────────────────
  ┌─────────────────────────────────────────────────────────┐
  │                    Shared Collection                     │
  │                                                         │
  │  {vec: [...], tenant_id: "A", ...}                     │
  │  {vec: [...], tenant_id: "B", ...}                     │
  │  {vec: [...], tenant_id: "A", ...}                     │
  │  ...                                                    │
  │                                                         │
  │  Single HNSW index over all vectors                     │
  │  Filter on tenant_id at query time                      │
  └─────────────────────────────────────────────────────────┘

  Pros: Simple, efficient resource usage, instant onboarding
  Cons: Noisy neighbors, recall degradation for small tenants,
        no per-tenant scaling, shared failure domain


  Pattern 3: "Tiered Hybrid" (Recommended for most cases)
  ────────────────────────────────────────────────────────
  ┌──────────────────────────────────────────────────────┐
  │  Large Tenants (> 1M vectors)                        │
  │  → Dedicated collection per tenant                   │
  │  → Independent scaling, SLA guarantees               │
  │                                                      │
  │  Medium Tenants (10K - 1M vectors)                   │
  │  → Namespace/partition per tenant                    │
  │  → Shared compute, separate indexes                  │
  │                                                      │
  │  Small Tenants (< 10K vectors)                       │
  │  → Shared collection with metadata filter            │
  │  → Brute force within tenant is fast enough          │
  └──────────────────────────────────────────────────────┘
```

---

## 5. Noisy Neighbor Mitigation

### 5.1 Types of Noisy Neighbor Problems

| Problem | Mechanism | Impact |
|---|---|---|
| **Memory hog** | Large tenant consumes disproportionate RAM | OOM for co-located tenants |
| **QPS hog** | Tenant's query volume saturates CPU | Latency spike for all tenants |
| **Index pollution** | Large tenant's vectors dominate graph structure | Recall degradation for small tenants |
| **Compaction storm** | Large tenant's bulk write triggers compaction | I/O contention for all |
| **Write amplification** | Frequent updates by one tenant cause reindexing | Build CPU consumed |

### 5.2 Mitigation Strategies

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                   Noisy Neighbor Mitigations                     │
  │                                                                 │
  │  1. RATE LIMITING (per tenant)                                  │
  │     ┌─────────────────────────────────────┐                     │
  │     │ Tenant A: 100 QPS max, 1000 WPS max│                     │
  │     │ Tenant B: 500 QPS max, 5000 WPS max│                     │
  │     │ Enforced at query router layer      │                     │
  │     └─────────────────────────────────────┘                     │
  │                                                                 │
  │  2. RESOURCE QUOTAS                                             │
  │     ┌─────────────────────────────────────┐                     │
  │     │ Max vectors per tenant: 10M         │                     │
  │     │ Max memory per tenant:  2 GB        │                     │
  │     │ Max concurrent queries: 50          │                     │
  │     └─────────────────────────────────────┘                     │
  │                                                                 │
  │  3. FAIR SCHEDULING                                             │
  │     ┌─────────────────────────────────────┐                     │
  │     │ Weighted fair queue per tenant      │                     │
  │     │ Priority: paid > free tier          │                     │
  │     │ Deadline-based scheduling           │                     │
  │     └─────────────────────────────────────┘                     │
  │                                                                 │
  │  4. PHYSICAL ISOLATION (last resort)                            │
  │     ┌─────────────────────────────────────┐                     │
  │     │ Move noisy tenant to dedicated node │                     │
  │     │ Triggered by: sustained > 80% CPU   │                     │
  │     │ Auto-migration with zero downtime   │                     │
  │     └─────────────────────────────────────┘                     │
  │                                                                 │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 6. Tenant-Aware Routing

### 6.1 Routing Architecture

```
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  Client Request: search(tenant="T42", query_vec, k=10)      │
  │                         │                                    │
  │                         ▼                                    │
  │  ┌──────────────────────────────────────────┐               │
  │  │          Tenant Router                    │               │
  │  │                                           │               │
  │  │  1. Lookup tenant → tier mapping          │               │
  │  │     T42 → "medium" tier                   │               │
  │  │                                           │               │
  │  │  2. Lookup tier → node assignment         │               │
  │  │     "medium" → namespace on node-pool-B   │               │
  │  │                                           │               │
  │  │  3. Apply rate limit check                │               │
  │  │     T42: 45/100 QPS quota used → ALLOW    │               │
  │  │                                           │               │
  │  │  4. Route to appropriate backend          │               │
  │  └────────────────┬─────────────────────────┘               │
  │                   │                                          │
  │         ┌─────────┼──────────┐                               │
  │         ▼         ▼          ▼                               │
  │  ┌──────────┐ ┌────────┐ ┌──────────┐                      │
  │  │ Pool A   │ │ Pool B │ │ Pool C   │                      │
  │  │ (large)  │ │(medium)│ │ (small)  │                      │
  │  │ dedicated│ │ shared │ │ shared + │                      │
  │  │ per-tenant│ │ ns    │ │  filter  │                      │
  │  └──────────┘ └────────┘ └──────────┘                      │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

### 6.2 Routing Metadata Table

```sql
  -- Tenant routing metadata (stored in PostgreSQL or etcd)
  CREATE TABLE tenant_routing (
      tenant_id       TEXT PRIMARY KEY,
      tier            ENUM('small', 'medium', 'large'),
      isolation_mode  ENUM('shared', 'namespace', 'dedicated'),
      node_pool       TEXT,
      collection_name TEXT,
      namespace       TEXT,
      qps_limit       INT,
      vector_limit    BIGINT,
      created_at      TIMESTAMP,
      last_rebalanced TIMESTAMP
  );

  -- Example entries:
  -- ('T1',  'large',  'dedicated',  'pool-a', 'coll_T1',  NULL,     1000, 50M,  ...)
  -- ('T42', 'medium', 'namespace',  'pool-b', 'shared_m', 'ns_T42', 100,  5M,   ...)
  -- ('T999','small',  'shared',     'pool-c', 'shared_s', NULL,     10,   100K, ...)
```

---

## 7. Capacity Planning Per Tenant

### 7.1 Per-Tenant Sizing Formula

```
  Memory per tenant =
    num_vectors * (d * bytes_per_dim + index_overhead + metadata_bytes)

  Where:
    d = embedding dimension
    bytes_per_dim = 4 (float32), 1 (int8), or m/d (PQ)
    index_overhead = ~40 bytes (HNSW node) or ~4 bytes (IVF assignment)
    metadata_bytes = avg payload size per vector

  Example: Tenant with 1M vectors, d=768, HNSW, 100 bytes metadata
    = 1M * (768 * 4 + 40 + 100)
    = 1M * 3,212
    = 3.2 GB

  QPS capacity per tenant:
    Available_QPS = total_node_QPS * tenant_weight / sum(all_weights)
```

### 7.2 Capacity Planning Table

| Tenant Tier | Vectors | Memory | QPS Allocation | Isolation | Monthly Cost |
|---|---|---|---|---|---|
| Free | < 10K | < 50 MB | 1-5 QPS | Shared | ~$0 (amortized) |
| Starter | 10K-100K | 50-500 MB | 5-50 QPS | Shared | $5-20 |
| Pro | 100K-5M | 0.5-16 GB | 50-500 QPS | Namespace | $50-500 |
| Enterprise | 5M-100M | 16-320 GB | 500-5000 QPS | Dedicated | $500-5000 |

---

## 8. Access Control Patterns

### 8.1 Defense in Depth

```
  ┌─────────────────────────────────────────────────────────────┐
  │                 Access Control Layers                        │
  │                                                             │
  │  Layer 1: API Key → Tenant ID mapping                       │
  │  ┌───────────────────────────────────────────────────────┐  │
  │  │ API key "sk_abc123" → tenant_id "T42"                │  │
  │  │ Validated at API gateway (< 1ms overhead)             │  │
  │  └───────────────────────────────────────────────────────┘  │
  │                                                             │
  │  Layer 2: Tenant ID injected into every query               │
  │  ┌───────────────────────────────────────────────────────┐  │
  │  │ search(q, filter={tenant_id: "T42", ...user_filters}) │  │
  │  │ Tenant filter is ALWAYS added server-side, never       │  │
  │  │ trusted from client. Defense against IDOR.             │  │
  │  └───────────────────────────────────────────────────────┘  │
  │                                                             │
  │  Layer 3: Collection/namespace-level ACL                    │
  │  ┌───────────────────────────────────────────────────────┐  │
  │  │ Dedicated tenants: collection ACL enforced at DB level│  │
  │  │ Cannot query another tenant's collection even with    │  │
  │  │ forged tenant_id.                                      │  │
  │  └───────────────────────────────────────────────────────┘  │
  │                                                             │
  │  Layer 4: Audit logging                                     │
  │  ┌───────────────────────────────────────────────────────┐  │
  │  │ Every query logged: {tenant, collection, timestamp,   │  │
  │  │ result_count, latency}. Detect anomalies.             │  │
  │  └───────────────────────────────────────────────────────┘  │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### 8.2 Common Security Mistakes

| Mistake | Risk | Fix |
|---|---|---|
| Trusting client-sent tenant_id | Cross-tenant data access | Server-side injection from auth token |
| Shared collection without filter | Full data exposure | Mandatory tenant filter at query layer |
| No rate limiting per tenant | DoS by one tenant | Per-tenant token bucket |
| Vector IDs expose tenant info | Information leakage | Use opaque UUIDs |
| No audit trail | Cannot detect breaches | Log all access with tenant context |

---

## 9. Cost Allocation

### 9.1 Metering Model

```
  Per-Tenant Cost Components:
  ──────────────────────────────────────────────────────────
  1. STORAGE:   num_vectors * bytes_per_vector * $/GB/mo
  2. COMPUTE:   queries_per_month * avg_query_cost
  3. INGESTION: vectors_inserted * per_insert_cost
  4. INDEX:     reindex_frequency * index_build_cost
  5. TRANSFER:  bytes_out * $/GB

  Example metering for a SaaS vector DB:
  ┌──────────────────────────────────────────────────────┐
  │ Storage:      $0.10 per 1M vectors per month         │
  │ Read units:   $0.01 per 1K queries                   │
  │ Write units:  $0.05 per 1K inserts                   │
  │ Compute:      $0.25 per pod-hour (dedicated tier)    │
  └──────────────────────────────────────────────────────┘
```

### 9.2 Cost Attribution for Shared Infrastructure

```
  Challenge: Tenant A (1M vectors) and Tenant B (100K vectors)
  share an HNSW index on a $500/mo node. How to split cost?

  Method 1: Proportional to vectors
    A pays: 1M/(1M+100K) * $500 = $454
    B pays: 100K/(1M+100K) * $500 = $45

  Method 2: Proportional to QPS consumed
    A: 200 QPS, B: 800 QPS
    A pays: 200/1000 * $500 = $100
    B pays: 800/1000 * $500 = $400

  Method 3: Hybrid (recommended)
    Base cost (storage):  proportional to vectors
    Variable cost (compute): proportional to QPS

    A: $227 (storage) + $50 (compute) = $277
    B: $23 (storage) + $200 (compute) = $223
```

---

## 10. Scaling Patterns

### 10.1 Tenant Growth Triggers

```
  Automatic Tier Promotion:
  ─────────────────────────

  Monitor per tenant:
  ┌────────────────────────────────────────────┐
  │ IF tenant.vectors > 100K                   │
  │    AND tenant.tier == "small":             │
  │    → Promote to "medium" (namespace)       │
  │    → Migrate vectors to dedicated namespace│
  │    → Update routing table                  │
  │                                            │
  │ IF tenant.vectors > 5M                     │
  │    OR tenant.qps > 500:                    │
  │    → Promote to "large" (dedicated)        │
  │    → Provision dedicated collection        │
  │    → Background migration                  │
  │    → Atomic routing switch                 │
  └────────────────────────────────────────────┘

  Demotion (downsizing):
  ┌────────────────────────────────────────────┐
  │ IF tenant.qps < 10 for 30 days             │
  │    AND tenant.tier == "large":             │
  │    → Demote to "medium"                    │
  │    → Reclaim dedicated resources           │
  │    → Notify tenant of tier change          │
  └────────────────────────────────────────────┘
```

### 10.2 Zero-Downtime Tenant Migration

```
  Step 1: Create destination (new namespace/collection)
  Step 2: Start dual-write (writes go to both old and new)
  Step 3: Backfill historical vectors to new destination
  Step 4: Verify vector count and recall parity
  Step 5: Switch reads to new destination (atomic routing update)
  Step 6: Stop writes to old destination
  Step 7: Garbage collect old data after grace period
```

---

## 11. Failure Modes Specific to Multi-Tenancy

| Failure | Trigger | Impact | Mitigation |
|---|---|---|---|
| Tenant filter bypass | Bug in filter injection | Cross-tenant data leak | Defense-in-depth, penetration testing |
| Large tenant OOM | Tenant exceeds quota | Node crash, all co-tenants affected | Hard memory limits, proactive migration |
| Hotspot tenant | Viral usage spike | Shared resource saturation | Auto-promotion to dedicated tier |
| Stale routing | Routing table not updated after migration | Queries go to wrong backend | Health checks, routing table versioning |
| Quota exhaustion | Tenant hits vector limit | Insert failures | Graceful degradation, queue overflow |
| Reindex cascade | Multiple tenants trigger reindex simultaneously | CPU/IO starvation | Staggered reindex scheduling |

---

## 12. Interview Narrative

### How to Present This in a Staff/Principal Interview

**Opening framing (30 seconds):**
> "Multi-tenant vector search is one of the hardest design problems in this space because
> vector indexes don't naturally support isolation. Unlike a relational database where you
> add a WHERE clause, filtering within an HNSW graph changes the recall characteristics
> fundamentally. My approach is to tier tenants by size and route them to the appropriate
> isolation level — shared for small, namespaced for medium, dedicated for large."

**When asked "How do you handle 10,000 tenants in a vector database?":**

1. **Categorize tenants:** Use the power-law distribution — most tenants are small. Perhaps
   100 tenants have > 1M vectors, 1000 have 10K-1M, and 8900 have < 10K.

2. **Small tenants (8900):** Shared collection with mandatory tenant_id metadata filter.
   At < 10K vectors per tenant, even brute force within the tenant's filtered subset is fast.

3. **Medium tenants (1000):** Namespace per tenant. Separate HNSW index per namespace, but
   shared compute pool. Good isolation with reasonable overhead.

4. **Large tenants (100):** Dedicated collections with independent scaling. Full SLA control.

5. **Routing layer:** Tenant router maps API key to tenant to tier to backend. Enforces
   rate limits. Handles tier promotions transparently.

**Demonstrating E7 depth:**
- Quantify the recall degradation from metadata filtering using the selectivity formula
- Explain the IDOR risk if tenant_id comes from the client rather than the auth token
- Discuss cost allocation: how do you fairly charge shared infrastructure to tenants?
- Propose the automatic tier promotion system with monitoring triggers
- Explain zero-downtime migration between tiers

**System design follow-up — "A tenant suddenly goes viral, 100x QPS spike":**
> "First, the rate limiter absorbs the spike to protect co-tenants. Then I'd auto-promote
> the tenant to a dedicated tier — spin up a dedicated collection, dual-write during
> migration, backfill history, atomic routing switch. The key is that the routing layer
> makes this transparent to the tenant's API calls. Total migration time: ~30 minutes for
> a 1M vector tenant, zero downtime."

**Red flags to avoid:**
- Don't propose "just add a tenant_id filter" without discussing recall impact.
- Don't ignore the noisy neighbor problem — it's the #1 operational issue.
- Don't forget that collection-per-tenant doesn't scale to 10K tenants (resource overhead).
- Don't overlook access control — IDOR in vector DBs is a real vulnerability class.
