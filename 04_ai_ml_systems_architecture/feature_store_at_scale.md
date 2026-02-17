# Feature Store Architecture at Scale

## Why This Matters at Staff/Principal Level

Feature stores are the bridge between data engineering and ML serving. At FAANG scale,
thousands of models consume millions of features, with freshness SLOs ranging from
milliseconds to hours. The staff engineer's challenge is designing a unified feature
platform that guarantees point-in-time correctness for training, sub-10ms serving for
inference, feature reuse across hundreds of teams, and cost efficiency at petabyte scale.
Getting this wrong causes training/serving skew — the #1 silent killer of ML system quality.

---

## Feature Store Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Feature Store Architecture                        │
│                                                                      │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐   │
│  │  Feature      │    │  Feature         │    │  Feature         │   │
│  │  Definitions  │───►│  Computation     │───►│  Storage         │   │
│  │  (Registry)   │    │  (Batch/Stream)  │    │  (Online/Offline)│   │
│  └──────────────┘    └──────────────────┘    └──────────────────┘   │
│         │                     │                        │             │
│         ▼                     ▼                        ▼             │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐   │
│  │  Feature      │    │  Monitoring &    │    │  Feature         │   │
│  │  Discovery    │    │  Quality Gates   │    │  Serving API     │   │
│  │  (Catalog)    │    │  (Freshness/Drift)│   │  (Online/Offline)│   │
│  └──────────────┘    └──────────────────┘    └──────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Online vs Offline Feature Stores

### Dual-Store Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  Training Pipeline                    Serving Pipeline          │
│  (Offline Path)                       (Online Path)             │
│                                                                │
│  ┌──────────────────┐                ┌──────────────────┐      │
│  │  Offline Store    │                │  Online Store     │      │
│  │  ─────────────── │                │  ─────────────── │      │
│  │  S3 + Delta Lake │                │  Redis Cluster    │      │
│  │  or Iceberg      │                │  + DynamoDB       │      │
│  │                  │                │                   │      │
│  │  Traits:         │                │  Traits:          │      │
│  │  • Petabyte-scale│                │  • Sub-10ms reads │      │
│  │  • Time-travel   │                │  • Entity-keyed   │      │
│  │  • Columnar      │                │  • Latest values  │      │
│  │  • Cheap storage │                │  • High throughput│      │
│  │  • Slow reads    │                │  • Expensive      │      │
│  └────────┬─────────┘                └────────┬─────────┘      │
│           │                                   │                │
│           ▼                                   ▼                │
│  Point-in-time joins                 get_online_features(      │
│  for training data                     entity_keys,            │
│  generation                            feature_refs            │
│                                      )                         │
└────────────────────────────────────────────────────────────────┘

Materialization: Offline ──► Online (scheduled or streaming)
```

### Storage Layer Comparison

| Dimension | Online Store | Offline Store |
|---|---|---|
| **Primary store** | Redis Cluster / DynamoDB | S3 + Delta Lake / Iceberg |
| **Data model** | Key-value (entity_key -> features) | Columnar (entity, timestamp, features) |
| **Read latency** | 1-10ms | Seconds to minutes |
| **Write pattern** | Upsert latest value | Append-only |
| **Time-travel** | No (latest only) | Yes (point-in-time queries) |
| **Cost (1TB)** | $10,000-30,000/mo | $23/mo (S3) |
| **Retention** | Days to weeks | Years |
| **Query pattern** | Lookup by entity key | Scan/filter by time range |
| **Typical size** | 10-100GB (hot features) | 10-100TB (full history) |

---

## Point-in-Time Correctness

The most critical correctness requirement in feature stores. Without it, training data
contains future information ("label leakage"), producing models that perform well offline
but fail in production.

### The Problem

```
Timeline:
  t1          t2          t3          t4
  │           │           │           │
  ▼           ▼           ▼           ▼
  Feature     Feature     Label       Training
  update      update      observed    dataset
  (v1)        (v2)                    generated

WRONG: Training row at t3 uses feature v2 (latest value)
       This is correct only if feature was v2 at label time.

RIGHT: Training row at t3 must use feature value as-of t3.
       Point-in-time join ensures temporal consistency.
```

### Point-in-Time Join Implementation

```sql
-- Offline store: features with timestamps
-- For each training example, join the most recent feature
-- value that existed BEFORE the label timestamp.

SELECT
    t.entity_id,
    t.label_timestamp,
    t.label,
    f.feature_value
FROM training_labels t
ASOF JOIN feature_history f
    ON t.entity_id = f.entity_id
    AND f.event_timestamp <= t.label_timestamp
ORDER BY f.event_timestamp DESC
LIMIT 1 PER MATCH;

-- In Spark (Delta Lake):
-- Window function approach for point-in-time correctness
SELECT entity_id, label_timestamp, label, feature_value
FROM (
    SELECT
        l.entity_id,
        l.label_timestamp,
        l.label,
        f.feature_value,
        ROW_NUMBER() OVER (
            PARTITION BY l.entity_id, l.label_timestamp
            ORDER BY f.event_timestamp DESC
        ) as rn
    FROM labels l
    JOIN features f
        ON l.entity_id = f.entity_id
        AND f.event_timestamp <= l.label_timestamp
) ranked
WHERE rn = 1;
```

---

## Feature Freshness SLOs

| Feature Tier | Freshness SLO | Computation | Online Store | Example |
|---|---|---|---|---|
| Real-time | <1 second | Streaming (Flink/Kafka) | Redis | User's last click |
| Near-real-time | <5 minutes | Micro-batch (Spark Streaming) | Redis | Session aggregates |
| Hourly | <1 hour | Scheduled batch (Airflow) | DynamoDB | Hourly engagement stats |
| Daily | <24 hours | Nightly batch (Spark) | DynamoDB | User profile features |
| Static | Updated manually | Ad-hoc | DynamoDB | Country GDP, category taxonomy |

### Freshness Monitoring

```
┌──────────────────────────────────────────────────────────┐
│              Feature Freshness Monitor                    │
│                                                           │
│  For each feature group:                                  │
│  ├── Track: last_materialized_at                          │
│  ├── Track: last_event_timestamp in source                │
│  ├── Compute: materialization_lag =                       │
│  │            now() - last_materialized_at                │
│  ├── Compute: data_lag =                                  │
│  │            last_materialized_at - last_event_timestamp │
│  └── Alert if materialization_lag > SLO threshold         │
│                                                           │
│  Dashboard:                                               │
│  Feature Group          SLO      Current Lag    Status    │
│  ─────────────────────────────────────────────────────    │
│  user_click_features    1s       0.3s           OK        │
│  session_aggregates     5min     3.2min         OK        │
│  daily_user_profile     24h      18h            OK        │
│  product_embeddings     1h       4.5h           BREACH    │
└──────────────────────────────────────────────────────────┘
```

---

## Feature Computation: Batch vs Streaming

### Computation Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                Feature Computation Pipelines                      │
│                                                                   │
│  ┌─────────────────────────────────┐                              │
│  │  Batch Pipeline (Spark/Airflow) │                              │
│  │  ──────────────────────────────│                              │
│  │  Data Lake ──► Transform ──► Offline Store (Delta Lake)       │
│  │                    │                                           │
│  │                    └──► Materialize ──► Online Store (Redis)   │
│  │  Schedule: hourly/daily                                       │
│  │  Pros: Simple, debuggable, cheap                              │
│  │  Cons: High latency, backfill complexity                      │
│  └─────────────────────────────────┘                              │
│                                                                   │
│  ┌─────────────────────────────────┐                              │
│  │  Streaming Pipeline (Flink)     │                              │
│  │  ──────────────────────────────│                              │
│  │  Kafka ──► Transform ──► Online Store (Redis)                 │
│  │                │                                               │
│  │                └──► Archive ──► Offline Store (Delta Lake)     │
│  │  Latency: sub-second                                          │
│  │  Pros: Fresh features, real-time                              │
│  │  Cons: Complex, expensive, hard to debug                      │
│  └─────────────────────────────────┘                              │
│                                                                   │
│  ┌─────────────────────────────────┐                              │
│  │  Lambda Architecture (Hybrid)   │                              │
│  │  ──────────────────────────────│                              │
│  │  Streaming: fast path for freshness                           │
│  │  Batch: slow path for correctness (overwrites)                │
│  │  Merge: online store reads latest from either path            │
│  │  Tradeoff: operational complexity for correctness + freshness │
│  └─────────────────────────────────┘                              │
└──────────────────────────────────────────────────────────────────┘
```

---

## Backfill Patterns

Backfill is the process of computing historical feature values for new features or correcting
existing features. At FAANG scale, this can mean processing petabytes of historical data.

### Backfill Strategies

| Strategy | Duration | Cost | Correctness | Use Case |
|---|---|---|---|---|
| Full recompute | Days-weeks | Very high | Perfect | Schema changes, bugs |
| Incremental | Hours-days | Medium | High | Appending new features |
| Snapshot-based | Hours | Low | Moderate | Feature derived from snapshots |
| Proxy backfill | Minutes | Minimal | Low | Use correlated existing feature |

### Backfill Architecture

```
┌──────────────────────────────────────────────────────────┐
│              Backfill Pipeline                            │
│                                                           │
│  New Feature Definition                                   │
│  ├── Validate: feature SQL/transform logic                │
│  ├── Estimate: cost (data scanned, compute hours)         │
│  ├── Approve: cost > $X requires tech lead approval       │
│  └── Execute:                                             │
│      ├── Replay historical events through transform       │
│      ├── Write to offline store with original timestamps  │
│      ├── Validate: spot-check against known values        │
│      └── Materialize to online store (latest values only) │
│                                                           │
│  Safety:                                                  │
│  ├── Backfill writes to staging table first               │
│  ├── Quality gates check distribution, null rates         │
│  └── Promote to production after validation               │
└──────────────────────────────────────────────────────────┘
```

---

## Storage Layer Deep Dive

### Online Store: Redis + DynamoDB

```
┌──────────────────────────────────────────────────────────┐
│           Online Feature Store Layout                     │
│                                                           │
│  Redis Cluster (hot path):                                │
│  ├── Key format: {feature_group}:{entity_id}             │
│  ├── Value: Protobuf-serialized feature vector           │
│  ├── TTL: 7 days (auto-expire stale entities)            │
│  ├── Cluster: 20 nodes, 512GB total                      │
│  └── Throughput: 500K reads/sec, p99 < 2ms               │
│                                                           │
│  DynamoDB (warm path, overflow):                          │
│  ├── Partition key: entity_id                             │
│  ├── Sort key: feature_group#version                     │
│  ├── On-demand capacity mode                              │
│  ├── DAX cache for hot keys                               │
│  └── Throughput: 100K reads/sec, p99 < 10ms              │
│                                                           │
│  Read Path:                                               │
│  Request ──► Redis (try first) ──► miss? ──► DynamoDB    │
│                                                           │
│  Write Path:                                              │
│  Materializer ──► Write to both Redis + DynamoDB         │
└──────────────────────────────────────────────────────────┘
```

### Offline Store: S3 + Delta Lake

```
s3://feature-store/
├── feature_group=user_profile/
│   ├── _delta_log/
│   ├── year=2024/month=01/
│   │   ├── part-00000.parquet
│   │   └── part-00001.parquet
│   └── year=2024/month=02/
│       └── ...
├── feature_group=product_embeddings/
│   ├── _delta_log/
│   └── ...
└── feature_group=session_aggregates/
    └── ...

Schema per feature group (Delta Lake):
┌──────────────┬──────────────┬─────────────────────────────┐
│ entity_id    │ event_time   │ feature_1 │ feature_2 │ ... │
│ (STRING)     │ (TIMESTAMP)  │ (DOUBLE)  │ (ARRAY)   │     │
├──────────────┼──────────────┼───────────┼───────────┼─────┤
│ user_123     │ 2024-01-15.. │ 0.85      │ [1,2,3]   │     │
│ user_123     │ 2024-01-16.. │ 0.87      │ [1,2,4]   │     │
│ user_456     │ 2024-01-15.. │ 0.32      │ [5,6,7]   │     │
└──────────────┴──────────────┴───────────┴───────────┴─────┘
```

---

## Feature Sharing Across Teams

### Feature Discovery and Governance

```
┌──────────────────────────────────────────────────────────────┐
│              Feature Registry / Catalog                       │
│                                                              │
│  Feature Group: user_engagement_daily                        │
│  ├── Owner: Recommendations Team                             │
│  ├── Description: Daily aggregated user engagement metrics   │
│  ├── Entity: user_id                                         │
│  ├── Freshness SLO: 24 hours                                │
│  ├── Features:                                               │
│  │   ├── daily_active_minutes (FLOAT) - avg session time     │
│  │   ├── content_interactions (INT) - clicks + likes + shares│
│  │   └── last_login_days_ago (INT) - recency signal          │
│  ├── Consumers: [rec_model_v3, churn_predictor, ads_ranking] │
│  ├── Data Quality:                                           │
│  │   ├── Null rate: <0.1%                                    │
│  │   ├── Value range: [0, 1440] for active_minutes           │
│  │   └── Coverage: 99.7% of active users                     │
│  └── Cost: $1,200/month (compute + storage)                  │
└──────────────────────────────────────────────────────────────┘
```

### Cross-Team Feature Reuse Economics

| Metric | Without Feature Store | With Feature Store |
|---|---|---|
| Features per model | 50-200 (team-specific) | 500-2000 (shared catalog) |
| Feature development time | 2-4 weeks per feature | 1-2 days (reuse existing) |
| Duplicate features | 40-60% across teams | <5% with dedup governance |
| Training/serving skew incidents | Monthly | Rare (unified serving path) |
| Onboarding new model | 2-3 months | 2-3 weeks |

---

## Platform Comparison: Feast vs Tecton vs Internal Solutions

| Capability | Feast (OSS) | Tecton | FAANG Internal |
|---|---|---|---|
| Online serving | Redis, DynamoDB | Managed (DynamoDB) | Custom (Redis + Memcached) |
| Offline store | BigQuery, Redshift, S3 | Managed (Databricks/S3) | Custom (Delta Lake on S3) |
| Streaming features | Limited (push-based) | Native (Rift engine) | Flink pipelines |
| Point-in-time joins | Yes (Spark-based) | Yes (optimized) | Yes (custom Spark jobs) |
| Feature registry | Basic | Rich UI + API | Custom catalog service |
| Monitoring | Basic | Built-in | Custom Prometheus + Grafana |
| Scale | 100s of features | 1000s of features | 10,000s+ features |
| Cost | Infrastructure only | $50-200K/year license | $1-5M/year (infra + eng) |
| Operational burden | High (self-managed) | Low (managed) | Medium (dedicated team) |

---

## Cost Optimization

### Cost Breakdown at Scale (10,000 features, 100M entities)

| Component | Monthly Cost | Optimization |
|---|---|---|
| Online store (Redis, 20 nodes) | $25,000 | Compress values, TTL stale entities |
| Online store (DynamoDB, overflow) | $8,000 | On-demand pricing, DAX caching |
| Offline store (S3, 50TB) | $1,150 | Lifecycle policies, Glacier for old data |
| Compute - batch (Spark, daily) | $15,000 | Spot instances, partition pruning |
| Compute - streaming (Flink, 24/7) | $20,000 | Right-size, only for real-time features |
| Metadata DB (PostgreSQL RDS) | $2,000 | Reserved instances |
| Monitoring (Prometheus + Grafana) | $3,000 | Downsample old metrics |
| **Total** | **$74,150** | |

### Key Cost Levers

1. **Feature TTL management** — Expire features for inactive entities. At FAANG, 30% of entity
   keys haven't been queried in 30 days. Evicting them saves $7-10K/month on Redis.

2. **Tiered computation** — Only 10-15% of features need real-time streaming. Moving the rest
   to hourly or daily batch saves 60-80% on compute.

3. **Compression** — Protobuf serialization of feature vectors saves 40-60% vs JSON. At 100M
   entities with 500 features each, this saves $5-8K/month on Redis.

4. **Shared computation** — If 5 teams each compute "user_daily_active_minutes" independently,
   consolidating saves 80% of redundant compute.

---

## Interview Narrative

**When asked about feature store design in a Staff/Principal interview:**

> "I'd start by separating the problem into two distinct stores with very different
> requirements. The offline store needs to support point-in-time correct joins for training
> data generation — this is non-negotiable because training/serving skew is the most common
> and most insidious bug in production ML. I'd use Delta Lake on S3 partitioned by entity
> and time, which gives us time-travel queries and ACID semantics for backfills.
>
> The online store needs sub-10ms latency for inference-time feature lookups. I'd use a
> Redis cluster as the hot tier for the most frequently accessed features, with DynamoDB
> as a warm tier for overflow. The key schema is `{feature_group}:{entity_id}` with
> Protobuf-serialized feature vectors as values.
>
> The hard part is keeping them in sync. Feature computation runs in two modes: batch
> pipelines in Spark for daily/hourly features (90% of features, cheap), and Flink streaming
> for the 10% that need sub-minute freshness. Both paths write to the offline store with
> proper timestamps, and a materialization job pushes the latest values to the online store.
>
> For feature sharing across teams, I'd invest heavily in a feature registry — a catalog
> where every feature has an owner, description, freshness SLO, quality metrics, and list
> of consumers. This is what turns a feature store from a team-level tool into an
> organizational platform. Without it, you get 40-60% feature duplication across teams.
>
> For backfills, I'd require that every feature definition be expressed as a deterministic
> transformation that can be replayed over historical data. New features write to a staging
> table first, pass quality gates (null rate, distribution checks), then get promoted.
>
> On cost — the online store (Redis) is usually the biggest line item. TTL management for
> inactive entities, Protobuf compression, and tiering between Redis and DynamoDB typically
> save 30-40%. For compute, the key insight is that very few features actually need
> real-time streaming — being disciplined about freshness SLOs and pushing most features
> to batch saves 60-80% on compute costs."

**Key signals this demonstrates:**
- Deep understanding of training/serving skew and point-in-time correctness
- Practical dual-store architecture with clear rationale
- Cost-consciousness with specific optimization strategies
- Organizational thinking (feature sharing, governance, catalog)
- Experience with real failure modes (stale features, backfill bugs, skew)
