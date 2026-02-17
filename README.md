# Database Systems for AI Engineers — Staff/Principal Level

> A principal engineer's handbook for database architecture at FAANG scale.
> Designed for E6/E7 interviews. No fluff. Pure architectural signal.

```
Target: 100M+ users | Multi-region | Strict SLOs | AI/ML workloads
Level:  Staff Engineer / Principal Architect
Focus:  Tradeoffs, failure modes, cost models, scale limits
```

---

## Staff-Level Database Decision Matrix

| Requirement | Best Fit | Why | Watch Out For |
|---|---|---|---|
| Sub-ms key lookup | Redis Cluster | In-memory, single-threaded event loop | Memory cost at TB scale |
| Billion-scale vector search | Dedicated Vector DB (Milvus/Weaviate) | Purpose-built ANN indexes | Index rebuild cost, memory pressure |
| Multi-region strong consistency | CockroachDB / Spanner | Distributed SQL with serializable isolation | Cross-region latency (150-300ms) |
| Multi-region eventual consistency | DynamoDB Global Tables / Cassandra | Tunable consistency, fast local writes | Conflict resolution complexity |
| Time-series telemetry | ClickHouse / TimescaleDB | Columnar compression, fast aggregation | Write amplification at high cardinality |
| Feature store (online) | Redis + DynamoDB | Low-latency point lookups | Feature freshness vs cost |
| Feature store (offline) | Delta Lake / Hudi on S3 | Cheap storage, time-travel queries | Query latency (seconds) |
| RAG document store | PostgreSQL + pgvector (small), Dedicated vector DB (large) | Operational simplicity vs scale | pgvector breaks at ~10M vectors |
| Model metadata / registry | PostgreSQL | ACID, relational queries, mature tooling | Schema evolution at org scale |
| Real-time event streaming | Kafka | Durable, ordered, replayable | Operational cost, partition rebalance |

---

## Redis vs Kafka vs DynamoDB Comparison

| Dimension | Redis | Kafka | DynamoDB |
|---|---|---|---|
| **Data model** | Key-value, streams, sets, sorted sets | Append-only log (topics/partitions) | Key-value / document |
| **Latency (p99)** | <1ms (single node) | 5-50ms (produce), 10-100ms (consume) | 5-10ms (single region) |
| **Durability** | Optional (RDB/AOF) | Durable (replicated log) | Always durable (3 AZs) |
| **Throughput** | 100K-1M ops/sec/node | 1M+ msgs/sec/cluster | Unlimited (on-demand) |
| **Scaling model** | Cluster (16384 hash slots) | Partition-based | On-demand or provisioned WCU/RCU |
| **Multi-region** | Active-active (CRDT, Enterprise) | MirrorMaker 2 / Cluster Linking | Global Tables (eventual) |
| **Consistency** | Eventual (cluster), strong (single) | Per-partition ordering | Eventually consistent or strong reads |
| **Cost at scale** | $$$$ (memory-bound) | $$ (disk-based) | $$$ (WCU/RCU pricing) |
| **Best for** | Caching, rate limiting, sessions, leaderboards | Event streaming, CDC, log aggregation | Serverless apps, session stores, metadata |
| **Worst for** | Large datasets (>RAM) | Low-latency point lookups | Complex queries, analytics |

---

## Vector Database Comparison Table

| Dimension | Pinecone | Milvus | Weaviate | Qdrant | pgvector |
|---|---|---|---|---|---|
| **Deployment** | Managed only | Self-hosted / Zilliz Cloud | Self-hosted / Cloud | Self-hosted / Cloud | PostgreSQL extension |
| **Max scale** | Billions | Billions | 100M+ | 100M+ | ~10M practical |
| **Index types** | Proprietary | HNSW, IVF, DiskANN | HNSW | HNSW | IVFFlat, HNSW |
| **Hybrid search** | Metadata filtering | Attribute + vector | BM25 + vector | Payload filtering | SQL + vector |
| **Multi-tenancy** | Namespaces | Partitions/Collections | Classes/Tenants | Collections/Payloads | Schemas/Row-level |
| **Disk-based index** | No (all in-memory) | DiskANN support | No | Memmap support | Disk-native |
| **GPU acceleration** | Unknown | Yes (Knowhere) | No | No | No |
| **Operational complexity** | Low (managed) | High | Medium | Medium | Low (it's Postgres) |
| **Cost per 1M vectors (768d)** | ~$70/mo | ~$30-50/mo (self-hosted) | ~$40-60/mo | ~$30-50/mo | ~$10-20/mo (but slower) |
| **When to choose** | Team wants zero ops | Need max flexibility at billion scale | GraphQL-native, hybrid search | Rust performance, simple API | Already on Postgres, <10M vectors |

---

## Sharding Decision Framework

```
START
  |
  +-- Data fits on single node? --> YES --> Don't shard. Use read replicas.
  |                                         (Sharding is operational debt)
  |
  +-- NO
       |
       +-- Write-heavy? --> Hash-based sharding
       |     |                (even distribution)
       |     |
       |     +-- Hot key problem? --> Add key-prefix salting
       |                               or local aggregation
       |
       +-- Range queries needed? --> Range-based sharding
       |     |                        (locality preserved)
       |     |
       |     +-- Hot range problem? --> Composite key
       |                                (tenant_id + timestamp)
       |
       +-- Multi-tenant? --> Shard per tenant (if few large tenants)
       |                      or hash within shared cluster (many small)
       |
       +-- Geographic locality? --> Geo-sharding
                                     (data stays near users)
```

**Shard Key Selection Principles:**

| Principle | Why | Anti-pattern |
|---|---|---|
| High cardinality | Even distribution across shards | Sharding by boolean field |
| Query alignment | Avoid scatter-gather | Shard by user_id but query by timestamp |
| Write distribution | No hot shards | Sequential IDs as shard key |
| Immutable | Avoid cross-shard migrations | Sharding by mutable status field |
| Growth-aware | Rebalancing cost is real | Shard by date (newest shard is always hot) |

---

## Multi-Region Patterns Summary

| Pattern | Consistency | Latency | Complexity | Use Case |
|---|---|---|---|---|
| **Active-Passive** | Strong (primary region) | High for writes (cross-region) | Low | Disaster recovery, compliance |
| **Active-Active (eventual)** | Eventual | Low (local writes) | High (conflict resolution) | Global user-facing apps |
| **Active-Active (CRDT)** | Eventual (convergent) | Low (local writes) | Medium | Counters, sets, flags |
| **Consensus-based (Spanner/CRDB)** | Strong (serializable) | High (cross-region quorum) | Medium | Financial transactions |
| **Read-local, Write-global** | Stale reads OK | Low reads, high writes | Medium | Content delivery, catalogs |
| **Follow-the-sun** | Strong (regional leader) | Low during active hours | Medium | Region-specific workloads |

```
                    +---------------+
    US-East --------| Conflict      |-------- EU-West
    (Leader)        | Resolution    |         (Leader)
        |           +---------------+             |
        |                                         |
   +---------+                               +---------+
   | Replica |                               | Replica |
   | US-West |                               | AP-East |
   +---------+                               +---------+

   Active-Active with regional leaders
   Conflict resolution: LWW, vector clocks, or app-level merge
```

---

## AI Storage Blueprint at 100M+ Users

```
+----------------------------------------------------------+
|                    Client Layer (100M+)                    |
+----------------------------+-----------------------------+
                             |
+----------------------------v-----------------------------+
|                   API Gateway / LB                        |
+-------+-----------+-----------+-----------+--------------+
        |           |           |           |
   +----v---+ +-----v----+ +---v---+ +-----v----+
   | Chat   | | Search   | | Rec   | |  RAG     |
   |Service | | Service  | |Engine | | Service  |
   +---+----+ +----+-----+ +--+----+ +----+-----+
       |           |           |           |
+------v-----------v-----------v-----------v---------------+
|                  Storage Layer                            |
|                                                          |
|  +---------+  +----------+  +----------+  +----------+   |
|  |  Redis  |  | Vector   |  | Feature  |  | Object   |   |
|  | Cluster |  |   DB     |  |  Store   |  |  Store   |   |
|  |(caching,|  |(Milvus/  |  |(Redis +  |  |  (S3)    |   |
|  | rate    |  | Pinecone)|  |DynamoDB) |  |          |   |
|  | limit)  |  |          |  |          |  |          |   |
|  +----+----+  +----+-----+  +----+-----+  +----+-----+   |
|       |            |             |              |         |
|  +----v------------v-------------v--------------v-----+   |
|  |              PostgreSQL / DynamoDB                  |   |
|  |        (metadata, model registry, audit log)        |   |
|  +-------------------------+---------------------------+   |
|                            |                              |
|  +-------------------------v---------------------------+   |
|  |           Kafka (CDC, events, telemetry)            |   |
|  +-------------------------+---------------------------+   |
|                            |                              |
|  +-------------------------v---------------------------+   |
|  |     ClickHouse / TimescaleDB (metrics, analytics)   |   |
|  +--------------------------+--------------------------+   |
+----------------------------------------------------------+
```

**Key Design Decisions:**

| Component | Choice | Why |
|---|---|---|
| Cache | Redis Cluster (3+ regions) | Sub-ms reads, rate limiting, session store |
| Vector search | Dedicated vector DB | Billion-scale ANN, not feasible in pgvector |
| Feature store (online) | Redis + DynamoDB | Low-latency serving, TTL-based freshness |
| Feature store (offline) | S3 + Delta Lake | Cost-effective historical features |
| Metadata | PostgreSQL | ACID, relational, mature ecosystem |
| Events | Kafka | Durable, ordered, replayable streams |
| Telemetry | ClickHouse | Columnar, fast aggregation, compression |
| Blobs | S3 | Cheap, durable, scalable |

---

## 30-Day Staff Interview Study Roadmap

### Week 1: Foundations + Redis Mastery

| Day | Topic | File |
|---|---|---|
| 1 | How staff engineers think about databases | `00_architect_mindset/how_staff_engineers_think_about_databases.md` |
| 2 | Tradeoff frameworks + cost/latency/consistency | `00_architect_mindset/tradeoff_frameworks.md` + `cost_vs_latency_vs_consistency.md` |
| 3 | Redis internal architecture | `01_redis_deep_dive/redis_internal_architecture.md` |
| 4 | Redis Cluster slot model + scaling | `01_redis_deep_dive/redis_cluster_slot_model.md` + `redis_scaling_patterns.md` |
| 5 | Redis failure modes + multi-region | `01_redis_deep_dive/redis_failure_modes.md` + `redis_multi_region_design.md` |
| 6 | Redis for LLM/AI systems | `01_redis_deep_dive/redis_for_llm_and_ai_systems.md` |
| 7 | Review: Redis cheat sheet + practice narration | `cheat_sheets/redis_cluster_summary.md` |

### Week 2: Vector Databases at Scale

| Day | Topic | File |
|---|---|---|
| 8 | ANN search fundamentals | `02_vector_db_deep_dive/ann_search_fundamentals.md` |
| 9 | HNSW vs IVF vs Flat | `02_vector_db_deep_dive/hnsw_vs_ivf_vs_flat.md` |
| 10 | Billion-scale vector architecture | `02_vector_db_deep_dive/billion_scale_vector_architecture.md` |
| 11 | Multi-tenant vector design | `02_vector_db_deep_dive/multi_tenant_vector_design.md` |
| 12 | Reindexing, compaction, failure modes | `02_vector_db_deep_dive/vector_reindexing_and_compaction.md` + `vector_db_failure_modes.md` |
| 13 | Vector DB vs pgvector tradeoffs | `02_vector_db_deep_dive/vector_db_vs_pgvector_tradeoffs.md` |
| 14 | Review: Vector DB cheat sheet + practice | `cheat_sheets/vector_db_summary.md` |

### Week 3: Scaling + Multi-Region

| Day | Topic | File |
|---|---|---|
| 15 | Shard key evolution | `03_scaling_databases/shard_key_evolution.md` |
| 16 | Repartitioning without downtime | `03_scaling_databases/repartitioning_without_downtime.md` |
| 17 | Hot key mitigation + read/write imbalance | `03_scaling_databases/hot_key_mitigation.md` + `read_write_imbalance.md` |
| 18 | Multi-region consistency patterns | `03_scaling_databases/multi_region_consistency_patterns.md` |
| 19 | Disaster recovery playbooks | `03_scaling_databases/disaster_recovery_playbooks.md` |
| 20 | Design: 10x scale thinking | `00_architect_mindset/designing_for_10x_scale.md` |
| 21 | Review: Sharding + multi-region cheat sheets | `cheat_sheets/shard_key_decision_framework.md` + `multi_region_patterns.md` |

### Week 4: AI/ML Systems + Case Studies

| Day | Topic | File |
|---|---|---|
| 22 | RAG storage at scale | `04_ai_ml_systems_architecture/rag_storage_at_scale.md` |
| 23 | Embedding lifecycle + feature store | `04_ai_ml_systems_architecture/embedding_lifecycle_management.md` + `feature_store_at_scale.md` |
| 24 | Telemetry storage + model registry | `04_ai_ml_systems_architecture/telemetry_and_metrics_storage.md` + `model_registry_and_metadata_design.md` |
| 25 | Case study: Global chat storage | `05_case_studies_staff_level/design_global_chat_storage.md` |
| 26 | Case study: LLM serving platform | `05_case_studies_staff_level/design_llm_serving_platform_storage.md` |
| 27 | Case study: Billion-scale vector search | `05_case_studies_staff_level/design_billion_scale_vector_search.md` |
| 28 | Case study: Real-time feature store | `05_case_studies_staff_level/design_real_time_feature_store.md` |
| 29 | Case study: Multi-region cache | `05_case_studies_staff_level/design_multi_region_cache_system.md` |
| 30 | Full review: All cheat sheets + mock narration | All `cheat_sheets/` |

---

## Repository Structure

```
db-systems-for-ai-engineers-staff-level/
├── README.md
├── 00_architect_mindset/
│   ├── how_staff_engineers_think_about_databases.md
│   ├── tradeoff_frameworks.md
│   ├── cost_vs_latency_vs_consistency.md
│   └── designing_for_10x_scale.md
├── 01_redis_deep_dive/
│   ├── redis_internal_architecture.md
│   ├── redis_cluster_slot_model.md
│   ├── redis_scaling_patterns.md
│   ├── redis_failure_modes.md
│   ├── redis_multi_region_design.md
│   └── redis_for_llm_and_ai_systems.md
├── 02_vector_db_deep_dive/
│   ├── ann_search_fundamentals.md
│   ├── hnsw_vs_ivf_vs_flat.md
│   ├── billion_scale_vector_architecture.md
│   ├── multi_tenant_vector_design.md
│   ├── vector_reindexing_and_compaction.md
│   ├── vector_db_failure_modes.md
│   └── vector_db_vs_pgvector_tradeoffs.md
├── 03_scaling_databases/
│   ├── shard_key_evolution.md
│   ├── repartitioning_without_downtime.md
│   ├── hot_key_mitigation.md
│   ├── read_write_imbalance.md
│   ├── multi_region_consistency_patterns.md
│   └── disaster_recovery_playbooks.md
├── 04_ai_ml_systems_architecture/
│   ├── rag_storage_at_scale.md
│   ├── embedding_lifecycle_management.md
│   ├── feature_store_at_scale.md
│   ├── telemetry_and_metrics_storage.md
│   └── model_registry_and_metadata_design.md
├── 05_case_studies_staff_level/
│   ├── design_global_chat_storage.md
│   ├── design_llm_serving_platform_storage.md
│   ├── design_billion_scale_vector_search.md
│   ├── design_real_time_feature_store.md
│   └── design_multi_region_cache_system.md
└── cheat_sheets/
    ├── redis_cluster_summary.md
    ├── vector_db_summary.md
    ├── shard_key_decision_framework.md
    ├── multi_region_patterns.md
    └── ai_storage_summary.md
```

---

## How to Use This Repo

1. **Sequential study**: Follow the 30-day roadmap above
2. **Topic deep-dive**: Jump to any section based on interview focus
3. **Interview prep**: Read cheat sheets the morning of your interview
4. **Mock practice**: Use the "Interview Narrative" section in each file to practice verbal explanations
5. **Case study practice**: Work through `05_case_studies_staff_level/` as timed design exercises (45 min each)

---

> *"The difference between a senior engineer and a staff engineer is not knowing more systems — it's knowing which system NOT to use, and being able to articulate why at the whiteboard."*
