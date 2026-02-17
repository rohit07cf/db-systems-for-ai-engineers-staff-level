# Design a Real-Time Feature Store for ML Model Serving

## Staff/Principal-Level System Design Exercise

---

## 1. Problem Statement

Design a real-time feature store that serves 500K feature lookups per second with
p99 latency < 5ms, supporting both online (real-time serving) and offline (batch
training) workloads. The system must guarantee point-in-time correctness for
training data, support feature freshness ranging from milliseconds (streaming) to
hours (batch), enable backfill without downtime, and serve as the shared feature
platform for hundreds of ML teams across the organization.

At staff level, the core tension is between serving latency (which demands
denormalized, pre-computed features in memory) and feature correctness (which
demands careful handling of time, late-arriving data, and the online/offline skew
that silently degrades model quality). This is the system that determines whether
your ML models work in production or only in notebooks.

---

## 2. Requirements

### 2.1 Functional Requirements

- **Online Serving**: Low-latency feature vector retrieval by entity key (user_id,
  item_id, session_id) for real-time inference.
- **Offline Serving**: Point-in-time correct feature retrieval for training dataset
  generation (avoid future data leakage).
- **Feature Computation**: Support batch (Spark), streaming (Flink/Spark Streaming),
  and on-demand (request-time) feature transformations.
- **Feature Registry**: Central catalog of all features with metadata (owner, schema,
  SLA, freshness, lineage).
- **Backfill**: Recompute historical feature values without impacting online serving.
- **Multi-Team Sharing**: Feature definitions reusable across teams/models with
  access controls and discoverability.
- **Monitoring**: Feature drift detection, staleness alerts, serving latency dashboards.

### 2.2 Non-Functional Requirements

- **Online Latency**: p50 < 1ms, p99 < 5ms for single-entity lookup.
- **Online Throughput**: 500K lookups/sec (batch gets of 50-200 features per entity).
- **Offline Throughput**: Generate 1B training rows/hour.
- **Feature Freshness**: Streaming features: < 1 minute. Batch features: < 1 hour.
- **Point-in-Time Correctness**: Zero future data leakage in training sets.
- **Availability**: 99.99% for online serving (on the ML inference critical path).
- **Feature Count**: 50K+ registered features across the organization.
- **Entity Count**: 2B+ entities (users, items, sessions, etc.).

---

## 3. Scale Numbers

| Metric | Value |
|---|---|
| Feature lookups/sec (online) | 500K |
| Avg features per lookup | 100 |
| Individual feature reads/sec | 50M |
| Avg feature value size | 50 bytes |
| Online store data throughput | 2.5 GB/sec read |
| Unique entities (online, hot) | ~500M |
| Total entities (offline, all time) | 2B+ |
| Registered feature definitions | 50K |
| Feature groups (tables) | ~2K |
| Streaming feature updates/sec | 200K |
| Batch feature jobs/day | ~500 |
| Training dataset generation/day | ~100 jobs, ~10B rows total |
| Online store size (hot features) | ~5 TB |
| Offline store size (all history) | ~500 TB |
| ML teams using the platform | ~200 |
| Models in production | ~2000 |

---

## 4. High-Level Architecture

```
  Feature Producers                      Feature Consumers
  ─────────────────                      ─────────────────
  ┌─────────────┐                        ┌──────────────┐
  │ Batch Jobs   │                        │ Model Serving│
  │ (Spark)      │──┐                ┌───│ (Online)     │
  └─────────────┘  │                │   └──────────────┘
  ┌─────────────┐  │  ┌──────────┐  │   ┌──────────────┐
  │ Stream Jobs  │──┼─▶│ Feature  │──┼───│ Training     │
  │ (Flink)      │  │  │ Platform │  │   │ Pipeline     │
  └─────────────┘  │  │ (Core)   │  │   │ (Offline)    │
  ┌─────────────┐  │  └──────────┘  │   └──────────────┘
  │ On-Demand   │──┘       │        │   ┌──────────────┐
  │ Transforms  │          │        └───│ Feature      │
  └─────────────┘          │            │ Monitoring   │
                           │            └──────────────┘
                           ▼
          ┌────────────────────────────────────┐
          │         Feature Platform Core       │
          │                                     │
          │  ┌──────────┐    ┌──────────────┐  │
          │  │ Feature   │    │ Computation  │  │
          │  │ Registry  │    │ Engine       │  │
          │  │ (Metadata)│    │ (Spark/Flink)│  │
          │  └──────────┘    └──────┬───────┘  │
          │                         │          │
          │  ┌──────────────────────▼───────┐  │
          │  │       Materialization Layer   │  │
          │  │  (Write to online + offline)  │  │
          │  └────┬─────────────────┬───────┘  │
          │       │                 │          │
          │  ┌────▼─────┐   ┌──────▼──────┐   │
          │  │  Online   │   │  Offline    │   │
          │  │  Store    │   │  Store      │   │
          │  │ (Redis/   │   │ (S3 +       │   │
          │  │  DynamoDB)│   │  Delta Lake)│   │
          │  └──────────┘   └─────────────┘   │
          └────────────────────────────────────┘
```

---

## 5. Storage Layer Design

### 5.1 Online Store (Serving Path)

**Primary: Redis Cluster (for < 2ms p99)**

```
Key format:
  feature_store:{entity_type}:{entity_id}:{feature_group}

Value format (MessagePack serialized):
  {
    "feature_a": 0.85,
    "feature_b": [1, 0, 3, 7],    // embedding or list feature
    "feature_c": "category_x",
    "_ts": 1706000000,             // materialization timestamp
    "_version": 42                 // monotonic version for staleness detection
  }

Example:
  GET feature_store:user:12345:purchase_stats
  → { "avg_order_value": 47.50, "order_count_30d": 12, "ltv_score": 0.73, ... }
```

**Why Redis Cluster?**
- Sub-millisecond reads (p99 < 1ms for GET on a well-sized cluster).
- Hash-based sharding across 200+ nodes.
- Pipeline support: Fetch 100 features in a single round-trip (~1ms).
- Pub/sub for real-time feature update notifications (used by monitoring).

**Capacity planning:**
- 500M hot entities * 100 features * 50 bytes = ~2.5 TB.
- With Redis overhead (1.5x): ~3.75 TB.
- On r6g.2xlarge (52 GB RAM) nodes: ~80 nodes. With RF=1 (Redis Cluster handles
  failover via replicas): ~160 nodes (80 primary + 80 replica).

**Alternative: DynamoDB (for > 10ms p99 budget)**
- Use for features that tolerate slightly higher latency (< 10ms).
- Better cost profile for large, sparse feature spaces (pay per read, not per GB).
- DAX (DynamoDB Accelerator) as a read-through cache brings p99 to ~3-5ms.

### 5.2 Offline Store (Training Path)

**Primary: Delta Lake on S3**

```
-- Delta table layout
s3://feature-store/offline/
  └── feature_group=purchase_stats/
       ├── _delta_log/
       │    ├── 00000000000000000000.json
       │    ├── 00000000000000000001.json
       │    └── ...
       ├── part-00000-*.parquet    (data files, partitioned by date)
       ├── part-00001-*.parquet
       └── ...

Schema:
  entity_id:      STRING
  event_timestamp: TIMESTAMP    -- When the feature value was valid
  created_timestamp: TIMESTAMP  -- When it was written to the store
  feature_a:      DOUBLE
  feature_b:      ARRAY<INT>
  feature_c:      STRING
```

**Why Delta Lake?**
- **Time travel**: Query features as of any historical timestamp. Essential for
  point-in-time correctness.
- **ACID transactions**: Concurrent writes from batch and streaming jobs without
  corruption.
- **Schema evolution**: Add new features to a group without breaking existing
  consumers. Consumers that do not need the new column simply ignore it.
- **Compaction**: Small streaming files are automatically compacted into larger
  Parquet files for efficient batch reads.

### 5.3 Point-in-Time Correctness (The Hard Problem)

```
The "future leak" problem:

Timeline:  ─────────────────────────────────────────────────
                T1          T2          T3         T_train
                │           │           │           │
Feature A:     v1          v2 ─────────────────────
Feature B:     ────────────             v3 ────────
Label:                                              LABEL

For a training example at time T2:
  Correct:   Feature A = v2, Feature B = (value at T1, as T2 has no update)
  WRONG:     Feature B = v3 (this is future data - not available at T2)

Point-in-time join:
  For each (entity, label_timestamp):
    For each feature:
      SELECT value WHERE event_timestamp <= label_timestamp
      ORDER BY event_timestamp DESC
      LIMIT 1
```

**Implementation:**

```python
# Point-in-time join using Delta Lake + Spark
def get_training_features(entity_df, feature_tables, timestamp_col):
    """
    entity_df: DataFrame with (entity_id, label_timestamp, label)
    feature_tables: List of Delta tables to join
    """
    result = entity_df
    for ft in feature_tables:
        # Read feature history from Delta Lake
        features = spark.read.format("delta").load(ft.path)

        # AS-OF join: for each entity, get latest feature before label_timestamp
        features_versioned = features.withWatermark("event_timestamp", "0 seconds")

        result = result.join(
            features_versioned,
            on=["entity_id"],
            how="left"
        ).filter(
            col("event_timestamp") <= col(timestamp_col)
        ).withColumn(
            "rn", row_number().over(
                Window.partitionBy("entity_id", timestamp_col)
                      .orderBy(col("event_timestamp").desc())
            )
        ).filter(col("rn") == 1).drop("rn", "event_timestamp")

    return result
```

**Performance at scale:**
- 1B training rows with 100 features from 20 feature groups.
- Naive: 20 point-in-time joins over the full history = hours.
- Optimized: Partition feature tables by date. For a training window of 90 days,
  only read 90 date partitions per feature group. Uses Delta Lake's data skipping.
- Further optimized: Pre-compute "feature snapshots" at regular intervals (daily).
  Point-in-time join snaps to the nearest snapshot, then applies delta updates.
  Reduces data scanned by 10-50x.

### 5.4 Feature Computation Pipeline

**Three computation modes:**

```
┌───────────────────────────────────────────────────────────┐
│                  Feature Computation Modes                  │
│                                                             │
│  BATCH (Spark)                                             │
│  ─────────────                                             │
│  Frequency: Hourly / Daily                                 │
│  Input: Data warehouse tables, logs                        │
│  Output: Writes full snapshot to online + offline store    │
│  Example: user_purchase_stats_30d (aggregated from orders) │
│                                                             │
│  STREAMING (Flink)                                         │
│  ─────────────────                                         │
│  Frequency: Continuous (sub-minute freshness)              │
│  Input: Kafka topics (events)                              │
│  Output: Incremental updates to online + offline store     │
│  Example: user_session_count_5min (sliding window)         │
│                                                             │
│  ON-DEMAND (Request-time)                                  │
│  ─────────────────────────                                 │
│  Frequency: Per request                                    │
│  Input: Request context + other features                   │
│  Output: Computed at serving time, not stored              │
│  Example: time_since_last_purchase (current_time - ts)     │
│                                                             │
└───────────────────────────────────────────────────────────┘
```

**Streaming pipeline detail (Flink):**

```
Kafka (events) → Flink Job → Feature Values → Dual Write
                                                  │
                                    ┌─────────────┼──────────────┐
                                    │             │              │
                              Redis (online)  Kafka (changelog)  Delta Lake
                                                  │              (offline,
                                              consumed by        via Kafka
                                              monitoring         Connect)
```

- Flink maintains windowed aggregations (e.g., count of clicks in last 5 minutes).
- On window trigger, feature value is written to Redis (online) and Kafka (for
  offline materialization and monitoring).
- **Exactly-once semantics**: Flink checkpoints + Kafka transactions ensure no
  duplicate or missing feature updates.

### 5.5 Backfill Without Downtime

```
Backfill scenario: Feature definition changed (e.g., new normalization logic).
Need to recompute all historical values + update online store.

Strategy: Blue/Green Feature Versions

1. Register new feature version (v2) in registry.
   v1 continues serving. v2 is "staging."

2. Run batch backfill job for v2:
   - Reads raw data from data warehouse
   - Computes features with new logic
   - Writes to offline store with version=v2
   - Writes to a SEPARATE Redis key namespace: feature_store:v2:{entity}:{group}

3. Validate v2:
   - Compare v1 vs v2 distributions (statistical tests)
   - Run shadow inference: model scores with v1 features vs v2 features
   - Alert if unexpected divergence

4. Promote v2:
   - Update feature registry: v2 is now "active"
   - Serving layer reads from v2 keys
   - Deprecate v1 keys (TTL expiry after 24h grace period)

5. Streaming jobs switch to new computation logic (Flink job restart with new config).

Zero downtime: v1 serves throughout. v2 only takes over after validation.
```

### 5.6 Feature Registry (PostgreSQL)

```sql
CREATE TABLE feature_groups (
    group_id        SERIAL PRIMARY KEY,
    group_name      TEXT UNIQUE NOT NULL,
    entity_type     TEXT NOT NULL,         -- 'user', 'item', 'session'
    description     TEXT,
    owner_team      TEXT NOT NULL,
    online_store    TEXT DEFAULT 'redis',  -- 'redis', 'dynamodb', 'none'
    offline_store   TEXT DEFAULT 'delta',
    freshness_sla   INTERVAL,             -- max acceptable staleness
    created_at      TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ
);

CREATE TABLE features (
    feature_id      SERIAL PRIMARY KEY,
    group_id        INT REFERENCES feature_groups(group_id),
    feature_name    TEXT NOT NULL,
    data_type       TEXT NOT NULL,         -- 'float64', 'int64', 'string', 'embedding[768]'
    description     TEXT,
    computation     JSONB,                 -- { "type": "batch", "sql": "...", "schedule": "hourly" }
    version         INT DEFAULT 1,
    status          TEXT DEFAULT 'active', -- 'active', 'deprecated', 'staging'
    tags            TEXT[],
    UNIQUE(group_id, feature_name, version)
);

CREATE TABLE feature_lineage (
    feature_id      INT REFERENCES features(feature_id),
    source_table    TEXT NOT NULL,
    transformation  TEXT,                  -- Description of the transformation
    PRIMARY KEY (feature_id, source_table)
);

CREATE TABLE feature_consumers (
    feature_id      INT REFERENCES features(feature_id),
    model_name      TEXT NOT NULL,
    model_version   TEXT,
    team            TEXT NOT NULL,
    PRIMARY KEY (feature_id, model_name, model_version)
);
```

**Why a registry matters:**
- **Discoverability**: 200 teams, 50K features. Without a registry, teams reimplement
  the same features with slight differences, causing training/serving skew.
- **Impact analysis**: Before deprecating a feature, query `feature_consumers` to
  know which models break.
- **SLA tracking**: Compare actual freshness vs `freshness_sla`. Alert when a
  batch job is late or a streaming job is lagging.

---

## 6. Detailed Component Design

### 6.1 Online Serving Path (Critical Path)

```
Model Serving Container
  │
  ├── 1. Receive inference request with entity_id
  │
  ├── 2. Feature Store SDK: get_online_features(entity_id, feature_list)
  │       │
  │       ├── a. Group features by feature_group (minimize round-trips)
  │       ├── b. For each group, issue Redis MGET (pipelined)
  │       │       Pipeline: [GET key1, GET key2, ...] → single round-trip
  │       ├── c. Deserialize MessagePack values
  │       ├── d. Check staleness: if _ts < (now - freshness_sla), emit warning metric
  │       └── e. For on-demand features: compute from other features + request context
  │
  ├── 3. Assemble feature vector [f1, f2, ..., f100]
  │
  └── 4. Pass to model for inference

Latency breakdown:
  - SDK overhead (grouping, serialization):   ~0.2ms
  - Redis MGET (100 features, pipelined):     ~0.8ms
  - Deserialization:                          ~0.3ms
  - On-demand computation:                    ~0.2ms
  - Total:                                    ~1.5ms (p50), ~4ms (p99)
```

**Optimization: Feature vector pre-computation.**
- For models with a fixed feature set, pre-compute and store the entire feature
  vector as a single blob: `feature_store:model:fraud_v3:user:12345 → [binary blob]`.
- Single Redis GET instead of MGET across multiple groups.
- Reduces p99 from ~4ms to ~1.5ms.
- Tradeoff: Higher write amplification (must update the blob when ANY constituent
  feature changes).

### 6.2 Offline Serving Path (Training Data Generation)

```
Training Pipeline (Spark):
  1. Load entity DataFrame with labels:
     (user_id, event_timestamp, label)

  2. For each feature group in the model's feature list:
     a. Read Delta table with time travel:
        spark.read.format("delta")
             .option("timestampAsOf", training_cutoff)
             .load(feature_group_path)
     b. Point-in-time join (as described in section 5.3)

  3. Output: Parquet dataset with (entity_id, features..., label)
     Ready for model training.

Performance:
  - 1B rows, 100 features from 20 groups
  - With date partitioning and snapshot optimization: ~30 minutes on 100-node Spark cluster
  - Without optimization: ~4 hours
```

### 6.3 Feature Freshness Monitoring

```
For each feature group:
  - Expected freshness: from registry (e.g., "5 minutes" for streaming, "1 hour" for batch)
  - Actual freshness: max(_ts) in online store for a sample of entities

  Monitoring query (runs every minute):
    For each feature_group:
      SAMPLE 1000 random entity keys
      GET feature_store:{entity_type}:{sampled_id}:{group}
      Extract _ts from each value
      freshness = now() - max(_ts)

      IF freshness > freshness_sla:
        ALERT: "Feature group {group} is stale. Expected: {sla}. Actual: {freshness}"

      Emit metrics:
        feature_freshness_seconds{group="{group}"} = freshness
        feature_staleness_violations{group="{group}"} = count(freshness > sla)
```

---

## 7. Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Redis node failure | Feature lookups fail for affected shard | Redis Cluster auto-failover to replica (< 5s); SDK retries |
| Redis full cluster outage | All online serving fails | Fallback to DynamoDB (higher latency); circuit breaker |
| Stale features (batch job late) | Model uses outdated features; degraded quality | Staleness alert; model can use default values for stale features |
| Streaming pipeline lag | Streaming features not fresh | Flink checkpoint recovery; auto-restart; alert on lag > SLA |
| Backfill job failure | v2 features incomplete | Backfill is idempotent; restart from last checkpoint; v1 continues serving |
| Feature schema mismatch | Model receives wrong types | Schema validation in SDK; registry enforces types; integration tests |
| Training/serving skew | Model performs differently in prod vs training | Feature logging at serving time; offline comparison with training features |
| Delta Lake corruption | Offline store unreadable | Delta Lake transaction log provides recovery; S3 versioning as backup |
| Hot entity (celebrity user) | Redis hotspot | Replicate hot keys to multiple shards; local SDK cache with short TTL |
| Thundering herd on backfill | Redis overwhelmed by bulk writes | Rate-limited backfill writer; write during off-peak; gradual rollout |

### Training/Serving Skew (The Silent Killer)

This deserves special attention because it is the most common ML production failure:

```
Sources of skew:
  1. Time skew: Training uses batch features (hourly). Serving uses streaming (real-time).
     The model trained on hourly aggregates but serves with minute-level aggregates.

  2. Code skew: Training uses Python/Spark. Serving uses Java/C++ SDK.
     Subtle differences in rounding, null handling, or type casting.

  3. Data skew: Training uses historical data. Serving uses live data.
     Feature distributions shift over time (concept drift).

Detection:
  - Log the feature vector at serving time (to Kafka, async).
  - Compare served features vs what the offline store would have returned for the
    same entity at the same timestamp.
  - Alert when distribution divergence (KL, KS test) exceeds threshold.

Prevention:
  - Use the SAME computation code for batch and streaming (unified SDK).
  - Feature store SDK is the single source of truth for feature retrieval in both
    online and offline contexts. Teams NEVER compute features ad-hoc.
```

---

## 8. Cost Analysis

| Component | Monthly Cost Estimate |
|---|---|
| Redis Cluster (160 nodes, r6g.2xlarge) | $280K |
| DynamoDB (secondary online store) | $50K |
| S3 (offline store, 500 TB) | $12K |
| Spark (batch compute, on-demand) | $150K |
| Flink (streaming, always-on) | $80K |
| Kafka (feature events) | $40K |
| PostgreSQL (registry) | $5K |
| Monitoring infrastructure | $20K |
| Network | $30K |
| **Total** | **~$667K/month** |

**Cost per feature lookup**: $667K / (500K/sec * 2.6M sec/month) = $0.0005/lookup.
**Cost per training row**: $667K * 0.3 (offline share) / 10B rows/month = $0.00002/row.

**Key insight**: Redis dominates cost (42%). The optimization of pre-computing
feature vectors (section 6.1) reduces the number of Redis keys by 10x but increases
write amplification. The right tradeoff depends on read/write ratio — at 500K reads/sec
vs 200K writes/sec, pre-computation wins.

---

## 9. Evolution Path (Handling 10x Growth)

### 9.1 From 500K to 5M Lookups/sec

- **Redis scaling**: Add nodes linearly. Redis Cluster handles 10x with 10x nodes.
- **SDK optimization**: Batch feature lookups across multiple entities in a single
  request (e.g., batch recommendation for 100 items). Use Redis pipelines.
- **Local caching**: For features that change slowly (e.g., user demographics),
  cache in the model serving container with 60s TTL. Reduces Redis load by 30-50%.

### 9.2 From 50K to 500K Features

- **Registry performance**: PostgreSQL handles this easily (500K rows is trivial).
- **Online store**: Feature groups proliferate. Need namespace management and
  automated cleanup of unused features (track last read time per feature).
- **Offline store**: 500K features across 2K groups. Delta Lake handles this, but
  point-in-time joins become expensive. Pre-computed snapshots are essential.

### 9.3 Real-Time Feature Computation (On-the-Fly)

- Move from "pre-compute and store" to "compute at request time" for certain features.
- Example: Instead of storing `user_click_count_5min` in Redis (requires streaming
  pipeline), query a real-time OLAP engine (Apache Druid, StarRocks) at serving time.
- Tradeoff: Higher serving latency (~10-20ms) but eliminates the streaming pipeline
  complexity for these features. Viable for features with latency budget > 10ms.

### 9.4 Federated Feature Store

- Multiple business units want independent feature stores but need cross-team sharing.
- Solution: Federated registry. Each team owns their feature groups but registers
  them in a global catalog. Cross-team access via API gateway with RBAC.
- Data stays in team-owned storage; the registry provides a unified discovery layer.

### 9.5 GPU Feature Serving

- For embedding features (768d vectors), retrieval at 5M/sec requires significant
  memory bandwidth.
- GPU-based serving (features stored in GPU HBM) can provide 10x throughput vs
  CPU + Redis for dense vector features.
- Use case: Real-time recommendation where the feature vector IS the item embedding.

---

## 10. Interview Narrative

### How to Present This in 45 Minutes

**Minutes 0-5: Frame the problem.** "A feature store is the bridge between ML
training and serving. The core challenges are: (1) serving latency < 5ms on the
inference critical path, (2) point-in-time correctness to prevent future data
leakage in training, and (3) feature freshness guarantees across batch and
streaming pipelines. I will design for 500K lookups/sec serving 200 teams."

**Minutes 5-15: Online/offline split.** Draw the dual-store architecture. Explain
why Redis for online (sub-ms reads) and Delta Lake for offline (time travel,
ACID, schema evolution). Walk through the key schema and the pipelined MGET
optimization. This shows you understand that ML serving latency budgets are
measured in microseconds, not milliseconds.

**Minutes 15-25: Point-in-time correctness.** This is the hardest section and the
one that most candidates skip. Draw the timeline diagram showing the future leak
problem. Walk through the point-in-time join implementation. Explain the snapshot
optimization that makes it 10x faster. This is THE differentiator for staff level.

**Minutes 25-35: Computation pipeline and backfill.** Show the three computation
modes (batch, streaming, on-demand). Explain the blue/green backfill strategy.
Discuss training/serving skew detection. These topics show that you have operated
ML systems in production, not just designed them on a whiteboard.

**Minutes 35-45: Multi-team sharing, cost, and evolution.** Feature registry with
lineage and impact analysis. Cost breakdown (Redis dominates). Evolution to
federated feature stores and GPU-based serving. End with the monitoring strategy
that catches feature staleness before the model degrades.

### Key Phrases That Signal Staff-Level Thinking

- "Point-in-time correctness is non-negotiable. A feature store that leaks future
  data into training sets will produce models that look great offline and fail in
  production. This is the most common ML production bug and the hardest to detect."
- "Training/serving skew is the silent killer. We prevent it by enforcing that the
  feature store SDK is the ONLY way to retrieve features — in both training and
  serving. No ad-hoc feature computation."
- "The blue/green backfill strategy ensures zero downtime. v1 continues serving
  while v2 is computed, validated, and promoted. This is critical when 2000 models
  depend on this infrastructure."
- "Redis pipelined MGET for 100 features completes in < 1ms — a single network
  round-trip. This is why we group features by feature_group and use a binary
  serialization format (MessagePack, not JSON)."
- "Feature freshness SLAs are feature-specific, not system-wide. A real-time
  fraud feature needs sub-minute freshness. A user demographic feature can be
  hours old. The system must support both without forcing all features into the
  more expensive streaming path."
