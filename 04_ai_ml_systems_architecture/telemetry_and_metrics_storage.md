# Telemetry and Metrics Storage for ML Systems

## Why This Matters at Staff/Principal Level

ML systems generate orders of magnitude more telemetry than traditional services. Every
prediction has latency, confidence scores, feature values, and downstream outcomes that must
be tracked. At FAANG scale — millions of predictions per second across hundreds of models —
the telemetry storage system becomes a critical piece of infrastructure. A staff engineer
must design for high-cardinality labels, efficient downsampling, cost-effective retention,
and query patterns that range from real-time alerting to retrospective A/B test analysis.
The wrong storage choice can cost millions annually or leave you blind to model degradation.

---

## ML Telemetry Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                    ML Telemetry Pipeline                              │
│                                                                      │
│  Model Serving ──► Telemetry Collector ──► Kafka ──► Consumers       │
│  (predictions)     (sidecar/SDK)          (buffer)     │             │
│                                                        │             │
│               ┌────────────────────────────────────────┤             │
│               │                │                │      │             │
│               ▼                ▼                ▼      ▼             │
│        ┌───────────┐   ┌───────────┐   ┌───────────┐ ┌──────────┐  │
│        │Real-time   │   │Time-series│   │Analytics  │ │Long-term │  │
│        │Alerting    │   │DB         │   │Warehouse  │ │Archive   │  │
│        │(Prometheus)│   │(ClickHouse│   │(BigQuery/ │ │(S3/GCS   │  │
│        │            │   │/Timescale)│   │ Snowflake)│ │ Parquet) │  │
│        └───────────┘   └───────────┘   └───────────┘ └──────────┘  │
│             │                │                │                      │
│             ▼                ▼                ▼                      │
│        PagerDuty/      Grafana           Looker/                    │
│        OpsGenie        Dashboards        Notebooks                  │
└──────────────────────────────────────────────────────────────────────┘
```

### What ML Systems Emit

| Telemetry Type | Volume | Retention | Query Pattern | Storage |
|---|---|---|---|---|
| Prediction logs | Very high (M/sec) | 30-90 days | Batch analytics | ClickHouse / S3 |
| Model latency (p50/p99) | High | 1 year | Time-series queries | Prometheus / TimescaleDB |
| Feature values at serving | Very high | 30 days | Debug, drift detection | S3 Parquet |
| Model accuracy metrics | Medium | Forever | Trend analysis | TimescaleDB / PostgreSQL |
| A/B test events | High | 6-12 months | Statistical analysis | BigQuery / ClickHouse |
| Training metrics | Medium | Forever | Experiment comparison | MLflow + PostgreSQL |
| Infrastructure metrics | High | 90 days | Capacity planning | Prometheus |
| Data quality metrics | Low | 1 year | Anomaly detection | TimescaleDB |

---

## Time-Series Database Selection

### ClickHouse vs TimescaleDB vs InfluxDB

```
┌─────────────────────────────────────────────────────────────┐
│              Decision Matrix: TSDB Selection                 │
│                                                              │
│  ClickHouse                                                  │
│  ├── Architecture: Column-oriented OLAP                      │
│  ├── Sweet spot: High-volume analytics, aggregations         │
│  ├── Cardinality: Handles high cardinality well              │
│  ├── Query: SQL (familiar, powerful)                         │
│  └── Scale: Excellent horizontal scaling                     │
│                                                              │
│  TimescaleDB                                                 │
│  ├── Architecture: PostgreSQL extension (hypertables)        │
│  ├── Sweet spot: Mixed time-series + relational queries      │
│  ├── Cardinality: Good with proper partitioning              │
│  ├── Query: Full PostgreSQL SQL + time-series functions      │
│  └── Scale: Good vertical, moderate horizontal               │
│                                                              │
│  InfluxDB                                                    │
│  ├── Architecture: Purpose-built TSDB (TSM engine)           │
│  ├── Sweet spot: Simple metrics collection, IoT              │
│  ├── Cardinality: Poor at high cardinality (known weakness)  │
│  ├── Query: Flux (proprietary) or InfluxQL                   │
│  └── Scale: Limited horizontal (OSS), better in Cloud        │
└─────────────────────────────────────────────────────────────┘
```

### Detailed Comparison

| Dimension | ClickHouse | TimescaleDB | InfluxDB |
|---|---|---|---|
| **Write throughput** | 1M+ rows/sec | 200K rows/sec | 500K points/sec |
| **Query latency (aggregation)** | 10-100ms | 50-500ms | 100ms-1s |
| **Compression ratio** | 10-20x | 5-10x | 5-15x |
| **High cardinality** | Excellent | Good | Poor (OOM risk) |
| **SQL support** | Full SQL | Full PostgreSQL | Limited (Flux/InfluxQL) |
| **JOINs** | Supported (distributed) | Full PostgreSQL JOINs | Not supported |
| **Continuous aggregates** | Materialized views | Native (TimescaleDB) | Continuous queries |
| **Operational complexity** | Medium-high (cluster) | Low (PG extension) | Low (single binary) |
| **Cost at 10TB** | $3,000-5,000/mo | $4,000-8,000/mo | $8,000-15,000/mo |
| **Best for ML** | Prediction logs, A/B analysis | Model metrics + relational | Simple monitoring |

**Staff-level recommendation:** Use ClickHouse for high-volume event-level telemetry
(prediction logs, A/B test events) and TimescaleDB for operational metrics that need
relational context (model performance over time joined with model metadata). Avoid
InfluxDB for ML systems due to high-cardinality label challenges.

---

## High-Cardinality Label Challenges

ML telemetry has inherently high cardinality: model_name x model_version x feature_set x
experiment_id x user_segment x region. This can produce millions of unique label combinations.

### The Cardinality Problem

```
Labels for a single metric "prediction_latency_ms":
  model_name:      100 models
  model_version:   5 versions each = 500
  endpoint:        3 (canary, stable, shadow) = 1,500
  region:          10 = 15,000
  user_segment:    20 = 300,000
  experiment_id:   50 active = 15,000,000

15 million unique time series for ONE metric.
At 1 sample/sec = 15M samples/sec = 1.3 trillion samples/day.
```

### Mitigation Strategies

| Strategy | Mechanism | Cardinality Reduction | Tradeoff |
|---|---|---|---|
| Label allowlisting | Only index known-useful label combos | 90%+ | Loss of ad-hoc queries |
| Pre-aggregation | Aggregate at ingestion (sum, count, histogram) | 95%+ | Loss of raw samples |
| Label value bucketing | Group rare values into "other" | 80-90% | Reduced granularity |
| Separate high-cardinality to logs | Route to ClickHouse instead of Prometheus | 99% for TSDB | Two systems to query |
| Hierarchical rollup | Store fine-grained short, rolled-up long | 70-80% after rollup | Time-delayed detail |

### Recommended Architecture for High Cardinality

```
┌──────────────────────────────────────────────────────────────┐
│          High-Cardinality Telemetry Architecture              │
│                                                              │
│  ML Predictions ──► Kafka                                    │
│                      │                                       │
│          ┌───────────┼──────────────┐                        │
│          ▼           ▼              ▼                        │
│   Prometheus     ClickHouse      S3 (Parquet)               │
│   (low-card      (high-card      (raw event                 │
│    metrics)       analytics)      archive)                   │
│                                                              │
│   Labels:        Columns:        All fields:                 │
│   model_name,    model_name,     Full prediction             │
│   region,        model_version,  log with all                │
│   status         experiment_id,  metadata                    │
│                  user_segment,                                │
│   Use: alerting  region, etc.    Use: deep-dive              │
│   Retention: 30d                 analysis                    │
│                  Use: dashboards Retention: 90d+             │
│                  A/B analysis                                │
│                  Retention: 1yr                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Downsampling Strategies

At FAANG scale, raw metric retention is prohibitively expensive. Downsampling reduces storage
costs by 90%+ while preserving analytical utility.

### Downsampling Tiers

```
┌──────────────────────────────────────────────────────────────┐
│              Downsampling Strategy                            │
│                                                              │
│  Raw (1-sec resolution)                                      │
│  ├── Retention: 48 hours                                     │
│  ├── Use: real-time debugging, incident response             │
│  └── Storage: 100% (baseline)                                │
│                                                              │
│  1-minute aggregates                                         │
│  ├── Retention: 30 days                                      │
│  ├── Store: min, max, avg, p50, p95, p99, count              │
│  ├── Use: dashboards, recent trend analysis                  │
│  └── Storage: ~2% of raw                                     │
│                                                              │
│  5-minute aggregates                                         │
│  ├── Retention: 1 year                                       │
│  ├── Store: min, max, avg, p50, p95, p99, count              │
│  ├── Use: long-term trends, capacity planning                │
│  └── Storage: ~0.4% of raw                                   │
│                                                              │
│  1-hour aggregates                                           │
│  ├── Retention: forever                                      │
│  ├── Store: min, max, avg, count                             │
│  ├── Use: historical analysis, annual reviews                │
│  └── Storage: ~0.03% of raw                                  │
└──────────────────────────────────────────────────────────────┘
```

### Implementation in TimescaleDB

```sql
-- Create continuous aggregates for automatic downsampling
CREATE MATERIALIZED VIEW model_latency_1min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 minute', ts) AS bucket,
    model_name,
    model_version,
    region,
    COUNT(*) as request_count,
    AVG(latency_ms) as avg_latency,
    percentile_agg(latency_ms) as latency_pct,  -- stores full percentile sketch
    MIN(latency_ms) as min_latency,
    MAX(latency_ms) as max_latency
FROM model_predictions
GROUP BY bucket, model_name, model_version, region;

-- Add retention policy: drop raw data after 48 hours
SELECT add_retention_policy('model_predictions', INTERVAL '48 hours');

-- Add refresh policy: keep continuous aggregate up to date
SELECT add_continuous_aggregate_policy('model_latency_1min',
    start_offset => INTERVAL '2 hours',
    end_offset => INTERVAL '1 minute',
    schedule_interval => INTERVAL '1 minute'
);
```

### Implementation in ClickHouse

```sql
-- Raw prediction log table (MergeTree with TTL)
CREATE TABLE prediction_logs (
    ts DateTime64(3),
    model_name LowCardinality(String),
    model_version String,
    experiment_id String,
    region LowCardinality(String),
    latency_ms Float32,
    confidence Float32,
    prediction String,
    features Map(String, Float64)
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (model_name, model_version, ts)
TTL ts + INTERVAL 7 DAY;

-- Materialized view for 1-minute rollups
CREATE MATERIALIZED VIEW prediction_rollup_1min
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(bucket)
ORDER BY (model_name, model_version, region, bucket)
TTL bucket + INTERVAL 1 YEAR
AS SELECT
    toStartOfMinute(ts) as bucket,
    model_name,
    model_version,
    region,
    countState() as request_count,
    avgState(latency_ms) as avg_latency,
    quantileState(0.5)(latency_ms) as p50_latency,
    quantileState(0.95)(latency_ms) as p95_latency,
    quantileState(0.99)(latency_ms) as p99_latency
FROM prediction_logs
GROUP BY bucket, model_name, model_version, region;
```

---

## Model Performance Metrics Storage

### Schema for Tracking Model Quality Over Time

```sql
-- Core model performance metrics (TimescaleDB)
CREATE TABLE model_performance (
    ts                  TIMESTAMPTZ NOT NULL,
    model_name          TEXT NOT NULL,
    model_version       TEXT NOT NULL,
    dataset_name        TEXT NOT NULL,          -- 'production', 'golden_set', 'slice_X'
    metric_name         TEXT NOT NULL,          -- 'accuracy', 'auc', 'f1', 'rmse'
    metric_value        DOUBLE PRECISION NOT NULL,
    sample_count        BIGINT,
    confidence_interval DOUBLE PRECISION[2],    -- [lower, upper]
    metadata            JSONB                   -- additional context
);

SELECT create_hypertable('model_performance', 'ts');

-- Index for common query patterns
CREATE INDEX idx_model_perf_lookup
    ON model_performance (model_name, model_version, metric_name, ts DESC);

-- Example: Track accuracy degradation over time
SELECT
    time_bucket('1 day', ts) as day,
    model_name,
    AVG(metric_value) as daily_accuracy,
    LAG(AVG(metric_value)) OVER (
        PARTITION BY model_name
        ORDER BY time_bucket('1 day', ts)
    ) as prev_day_accuracy
FROM model_performance
WHERE metric_name = 'accuracy'
    AND dataset_name = 'production'
    AND ts > now() - INTERVAL '90 days'
GROUP BY day, model_name
ORDER BY day;
```

---

## A/B Test Result Storage

### A/B Test Schema Design

```sql
-- Experiment definition
CREATE TABLE experiments (
    experiment_id       UUID PRIMARY KEY,
    experiment_name     TEXT NOT NULL,
    hypothesis          TEXT,
    model_control       TEXT NOT NULL,          -- model_name:version
    model_treatment     TEXT NOT NULL,
    traffic_split       DECIMAL(3,2) NOT NULL,  -- 0.50 = 50/50
    primary_metric      TEXT NOT NULL,          -- 'conversion_rate'
    guardrail_metrics   TEXT[],                 -- ['latency_p99', 'error_rate']
    min_sample_size     BIGINT,
    start_ts            TIMESTAMPTZ NOT NULL,
    end_ts              TIMESTAMPTZ,
    status              TEXT DEFAULT 'running', -- running/completed/stopped
    conclusion          TEXT,                   -- 'treatment_wins'/'no_difference'/'control_wins'
    created_by          TEXT NOT NULL
);

-- Per-unit (user) experiment assignments
CREATE TABLE experiment_assignments (
    experiment_id       UUID REFERENCES experiments(experiment_id),
    unit_id             TEXT NOT NULL,          -- user_id
    variant             TEXT NOT NULL,          -- 'control' or 'treatment'
    assigned_at         TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (experiment_id, unit_id)
);

-- Metric observations per unit per day (ClickHouse for volume)
CREATE TABLE experiment_metrics (
    experiment_id       UUID,
    unit_id             String,
    variant             LowCardinality(String),
    metric_name         LowCardinality(String),
    metric_value        Float64,
    ts                  DateTime64(3)
) ENGINE = MergeTree()
PARTITION BY (experiment_id, toYYYYMMDD(ts))
ORDER BY (experiment_id, variant, metric_name, unit_id, ts);

-- Pre-computed daily summaries for dashboarding
CREATE TABLE experiment_daily_summary (
    experiment_id       UUID,
    variant             LowCardinality(String),
    metric_name         LowCardinality(String),
    day                 Date,
    unit_count          UInt64,
    metric_sum          Float64,
    metric_sum_sq       Float64,              -- for variance calculation
    metric_count        UInt64
) ENGINE = SummingMergeTree()
PARTITION BY experiment_id
ORDER BY (experiment_id, variant, metric_name, day);
```

### A/B Analysis Query Pattern

```sql
-- Calculate statistical significance for an experiment
WITH stats AS (
    SELECT
        variant,
        COUNT(DISTINCT unit_id) as n,
        AVG(metric_value) as mean,
        STDDEV(metric_value) as std
    FROM experiment_metrics
    WHERE experiment_id = 'abc-123'
        AND metric_name = 'conversion_rate'
    GROUP BY variant
),
control AS (SELECT * FROM stats WHERE variant = 'control'),
treatment AS (SELECT * FROM stats WHERE variant = 'treatment')
SELECT
    treatment.mean - control.mean as lift,
    (treatment.mean - control.mean) / control.mean * 100 as lift_pct,
    (treatment.mean - control.mean) /
        SQRT(POW(control.std, 2)/control.n + POW(treatment.std, 2)/treatment.n)
        as z_score
FROM control, treatment;
-- z_score > 1.96 implies p < 0.05 (two-tailed)
```

---

## Retention Policies and Cost Modeling

### Retention Policy Matrix

| Data Type | Hot (SSD) | Warm (HDD/S3-IA) | Cold (Glacier) | Total Retention |
|---|---|---|---|---|
| Raw prediction logs | 7 days | 83 days | 275 days | 1 year |
| 1-min aggregates | 30 days | 335 days | - | 1 year |
| 5-min aggregates | 1 year | 2 years | - | 3 years |
| Hourly aggregates | Forever | - | - | Forever |
| A/B test raw events | 30 days | 5 months | 7 months | 1 year |
| Model performance | Forever | - | - | Forever |
| Training metrics | Forever | - | - | Forever |

### Cost Model (100 models, 1M predictions/sec aggregate)

| Component | Specification | Monthly Cost |
|---|---|---|
| ClickHouse cluster | 6 nodes, 32-core, 256GB, 4TB NVMe each | $18,000 |
| TimescaleDB | 2 nodes (primary + replica), 16-core, 128GB | $4,000 |
| Prometheus | 3 nodes (HA), 64GB each | $3,000 |
| Kafka cluster | 6 brokers, 1TB each, 3-day retention | $8,000 |
| S3 archival storage | 50TB, Parquet compressed | $1,150 |
| S3 Glacier | 200TB, cold archive | $800 |
| Grafana Cloud | Enterprise, 50 dashboards, 20 users | $2,000 |
| Data transfer (inter-AZ) | ~10TB/month | $1,000 |
| **Total** | | **$37,950** |

### Cost Optimization Strategies

| Strategy | Savings | Implementation Effort |
|---|---|---|
| Aggressive downsampling | 40-60% on TSDB storage | Low (configure policies) |
| ClickHouse compression (ZSTD) | 30-50% on disk | Low (table settings) |
| LowCardinality column type | 20-30% on ClickHouse | Low (schema change) |
| Move raw logs to S3 after 7 days | 60% on ClickHouse | Medium (ETL pipeline) |
| Prometheus remote write to Thanos | 50% on Prometheus | Medium (deploy Thanos) |
| Reduce label cardinality | 30-50% on Prometheus | Medium (instrumentation change) |
| Reserved instances for DBs | 30-40% on compute | Low (procurement) |

---

## Dashboarding at Scale

### Dashboard Architecture

```
┌──────────────────────────────────────────────────────────────┐
│            ML Observability Dashboard Stack                    │
│                                                              │
│  ┌──────────────────────────────────────────────────┐        │
│  │  Executive Dashboard (Looker/Tableau)             │        │
│  │  ├── Model portfolio health (red/yellow/green)    │        │
│  │  ├── Business metric impact of ML                 │        │
│  │  └── Cost overview across all ML systems          │        │
│  │  Refresh: hourly | Data source: BigQuery          │        │
│  └──────────────────────────────────────────────────┘        │
│                                                              │
│  ┌──────────────────────────────────────────────────┐        │
│  │  ML Engineer Dashboard (Grafana)                  │        │
│  │  ├── Per-model latency, throughput, error rate    │        │
│  │  ├── Feature freshness and data quality           │        │
│  │  ├── Prediction distribution drift                │        │
│  │  └── A/B test progress (sample size, significance)│        │
│  │  Refresh: 30s | Data source: Prometheus + TSDB    │        │
│  └──────────────────────────────────────────────────┘        │
│                                                              │
│  ┌──────────────────────────────────────────────────┐        │
│  │  Debug Dashboard (Grafana + ClickHouse)           │        │
│  │  ├── Individual prediction lookup                 │        │
│  │  ├── Feature value inspection at serving time     │        │
│  │  ├── Request tracing (correlation ID)             │        │
│  │  └── Model comparison (control vs treatment)      │        │
│  │  Refresh: on-demand | Data source: ClickHouse     │        │
│  └──────────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────────┘
```

---

## Interview Narrative

**When asked about telemetry and metrics storage for ML systems in a Staff/Principal interview:**

> "ML telemetry is fundamentally different from traditional service metrics in two ways:
> volume and cardinality. A model serving system at FAANG scale might produce millions of
> predictions per second, each with a dozen labels — model name, version, experiment ID,
> user segment, region. The cardinality explosion from these label combinations can bring
> a naive Prometheus setup to its knees.
>
> I'd design a tiered telemetry architecture. Prometheus handles low-cardinality operational
> metrics — the stuff you alert on: latency p99, error rate, throughput, per model and
> region. That's maybe 50,000 unique time series, well within Prometheus's comfort zone.
>
> For high-cardinality analytics — prediction logs with experiment IDs, user segments,
> individual feature values — I'd route everything to ClickHouse. Its columnar storage
> with LowCardinality optimization and ZSTD compression handles billions of rows per day
> at 10-20x compression. SQL support makes it accessible to data scientists who need to
> do ad-hoc analysis on A/B test results.
>
> Downsampling is the key cost lever. I'd keep raw 1-second data for 48 hours (incident
> debugging), 1-minute aggregates for 30 days (dashboards), and 5-minute aggregates for
> a year (trend analysis). This reduces storage by 99.6% compared to keeping raw data
> for a year. In TimescaleDB, continuous aggregates handle this automatically.
>
> For A/B test storage specifically, I'd design for the query patterns analysts actually
> run: daily metric summaries per variant that can compute z-scores incrementally. Storing
> sum and sum-of-squares per day per variant lets you calculate statistical significance
> without scanning raw events, which makes dashboard queries go from minutes to milliseconds.
>
> The total cost for a mature ML telemetry stack at FAANG scale — ClickHouse, TimescaleDB,
> Prometheus, Kafka, Grafana — runs about $35-40K per month. The biggest cost optimization
> is disciplined retention policies and pushing raw data to S3 Parquet after 7 days, which
> cuts ClickHouse storage costs by 60%."

**Key signals this demonstrates:**
- Understanding of ML-specific telemetry challenges (cardinality, volume)
- Practical multi-system architecture with clear role for each component
- Deep knowledge of TSDB tradeoffs (ClickHouse vs TimescaleDB vs InfluxDB)
- Cost modeling with specific numbers and optimization strategies
- Query pattern awareness (A/B analysis, dashboarding, debugging)
- Downsampling as a first-class architectural concern
