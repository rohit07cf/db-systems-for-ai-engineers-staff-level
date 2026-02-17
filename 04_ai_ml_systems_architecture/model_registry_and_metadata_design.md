# Model Registry and Metadata Design

## Why This Matters at Staff/Principal Level

A model registry is the system of record for everything related to ML models — what was
trained, how it was trained, what data it used, where it's deployed, and whether it's
approved for production. At FAANG scale with thousands of models and tens of thousands of
experiments, the registry becomes critical infrastructure that enables reproducibility,
governance, compliance, and operational safety. A staff engineer must design a registry
that scales to the organization's ML velocity while enforcing the guardrails that prevent
bad models from reaching production.

---

## Model Registry Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Model Registry Architecture                       │
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐       │
│  │  Experiment   │    │  Model       │    │  Artifact        │       │
│  │  Tracker      │───►│  Registry    │───►│  Store           │       │
│  │  (MLflow/W&B) │    │  (metadata)  │    │  (S3/GCS)        │       │
│  └──────────────┘    └──────────────┘    └──────────────────┘       │
│         │                   │                      │                 │
│         ▼                   ▼                      ▼                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐       │
│  │  Lineage      │    │  Deployment  │    │  Model Card      │       │
│  │  Graph         │    │  Manager     │    │  Generator       │       │
│  │  (data + code) │    │  (CI/CD)     │    │  (compliance)    │       │
│  └──────────────┘    └──────────────┘    └──────────────────┘       │
│                             │                                        │
│                             ▼                                        │
│                      ┌──────────────┐                                │
│                      │  Approval    │                                │
│                      │  Workflow    │                                │
│                      │  (human-in-  │                                │
│                      │   the-loop)  │                                │
│                      └──────────────┘                                │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Core Data Model

### Entity Relationship Diagram

```
┌─────────────┐       ┌─────────────────┐       ┌─────────────────┐
│  Project    │ 1───M │  Experiment     │ 1───M │  Run            │
│  ─────────  │       │  ──────────     │       │  ─────          │
│  project_id │       │  experiment_id  │       │  run_id         │
│  name       │       │  project_id(FK) │       │  experiment_id  │
│  team       │       │  name           │       │  status         │
│  created_by │       │  hypothesis     │       │  start_time     │
└─────────────┘       │  created_at     │       │  end_time       │
                      └─────────────────┘       │  git_commit     │
                                                │  parameters     │
                                                │  metrics        │
                                                └────────┬────────┘
                                                         │
                                                    1────┤───M
                                                         │
┌─────────────────┐       ┌─────────────────┐    ┌──────┴────────┐
│  Deployment     │ M───1 │  Model Version  │ 1──│  Artifact     │
│  ──────────     │       │  ─────────────  │    │  ────────     │
│  deployment_id  │       │  version_id     │    │  artifact_id  │
│  version_id(FK) │       │  run_id(FK)     │    │  run_id(FK)   │
│  environment    │       │  model_name     │    │  artifact_type│
│  traffic_pct    │       │  stage          │    │  storage_uri  │
│  created_at     │       │  registered_at  │    │  size_bytes   │
│  created_by     │       │  approved_by    │    │  checksum     │
└─────────────────┘       │  model_card_id  │    └───────────────┘
                          └─────────────────┘
```

---

## PostgreSQL Schema Design

### Core Tables

```sql
-- Projects: top-level organizational unit
CREATE TABLE projects (
    project_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(128) NOT NULL UNIQUE,
    description     TEXT,
    team            VARCHAR(64) NOT NULL,
    created_by      VARCHAR(64) NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT now(),
    settings        JSONB DEFAULT '{}'::jsonb     -- project-level config
);

-- Experiments: a hypothesis or feature being tested
CREATE TABLE experiments (
    experiment_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(project_id),
    name            VARCHAR(256) NOT NULL,
    description     TEXT,
    hypothesis      TEXT,
    tags            TEXT[],
    created_by      VARCHAR(64) NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT now(),
    archived_at     TIMESTAMPTZ,
    UNIQUE(project_id, name)
);

-- Runs: a single training/evaluation execution
CREATE TABLE runs (
    run_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiments(experiment_id),
    status          VARCHAR(16) NOT NULL DEFAULT 'running',
                    -- running, completed, failed, killed
    start_time      TIMESTAMPTZ NOT NULL DEFAULT now(),
    end_time        TIMESTAMPTZ,
    git_repo        TEXT,
    git_commit      VARCHAR(40),
    git_branch      VARCHAR(128),
    docker_image    TEXT,
    entry_point     TEXT,                          -- script path
    parameters      JSONB NOT NULL DEFAULT '{}',   -- hyperparameters
    hardware_spec   JSONB,                         -- GPU type, count, memory
    created_by      VARCHAR(64) NOT NULL,
    tags            TEXT[]
);

CREATE INDEX idx_runs_experiment ON runs(experiment_id, start_time DESC);
CREATE INDEX idx_runs_status ON runs(status) WHERE status = 'running';

-- Run metrics: time-series of training metrics
CREATE TABLE run_metrics (
    metric_id       BIGSERIAL,
    run_id          UUID NOT NULL REFERENCES runs(run_id),
    key             VARCHAR(128) NOT NULL,         -- 'train_loss', 'val_accuracy'
    value           DOUBLE PRECISION NOT NULL,
    step            BIGINT,                        -- training step
    timestamp       TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (run_id, key, metric_id)
) PARTITION BY HASH(run_id);

-- Create 16 partitions for write throughput
CREATE TABLE run_metrics_p0 PARTITION OF run_metrics FOR VALUES WITH (MODULUS 16, REMAINDER 0);
CREATE TABLE run_metrics_p1 PARTITION OF run_metrics FOR VALUES WITH (MODULUS 16, REMAINDER 1);
-- ... (p2 through p15)

CREATE INDEX idx_metrics_lookup ON run_metrics(run_id, key, step);

-- Artifacts: files produced by runs
CREATE TABLE artifacts (
    artifact_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID NOT NULL REFERENCES runs(run_id),
    artifact_type   VARCHAR(32) NOT NULL,
                    -- 'model', 'checkpoint', 'dataset_ref', 'config', 'log'
    name            VARCHAR(256) NOT NULL,
    storage_uri     TEXT NOT NULL,                  -- s3://bucket/path
    size_bytes      BIGINT,
    checksum_sha256 VARCHAR(64),
    content_type    VARCHAR(128),
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ DEFAULT now(),
    UNIQUE(run_id, artifact_type, name)
);

CREATE INDEX idx_artifacts_run ON artifacts(run_id);
```

### Model Versioning and Stage Management

```sql
-- Registered models: named model families
CREATE TABLE registered_models (
    model_name      VARCHAR(256) PRIMARY KEY,
    description     TEXT,
    team            VARCHAR(64) NOT NULL,
    created_by      VARCHAR(64) NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT now(),
    last_updated    TIMESTAMPTZ DEFAULT now(),
    tags            TEXT[]
);

-- Model versions: specific trained instances
CREATE TABLE model_versions (
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_name      VARCHAR(256) NOT NULL REFERENCES registered_models(model_name),
    version_number  INT NOT NULL,
    run_id          UUID REFERENCES runs(run_id),
    stage           VARCHAR(32) NOT NULL DEFAULT 'none',
                    -- none, staging, canary, production, archived
    artifact_uri    TEXT NOT NULL,                  -- s3://models/name/v3/
    model_format    VARCHAR(32),                    -- 'pytorch', 'onnx', 'tensorflow', 'sklearn'
    model_size_mb   INT,
    input_schema    JSONB,                          -- expected input format
    output_schema   JSONB,                          -- output format
    metrics         JSONB NOT NULL DEFAULT '{}',    -- final evaluation metrics
    registered_at   TIMESTAMPTZ DEFAULT now(),
    registered_by   VARCHAR(64) NOT NULL,
    approved_by     VARCHAR(64),
    approved_at     TIMESTAMPTZ,
    model_card_id   UUID,
    UNIQUE(model_name, version_number)
);

CREATE INDEX idx_mv_stage ON model_versions(model_name, stage);
CREATE INDEX idx_mv_run ON model_versions(run_id);

-- Stage transition log: audit trail
CREATE TABLE stage_transitions (
    transition_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      UUID NOT NULL REFERENCES model_versions(version_id),
    from_stage      VARCHAR(32),
    to_stage        VARCHAR(32) NOT NULL,
    transitioned_by VARCHAR(64) NOT NULL,
    reason          TEXT,
    automated       BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_transitions_version ON stage_transitions(version_id, created_at);
```

---

## Lineage Tracking

### Data and Code Lineage Schema

```sql
-- Dataset references: what data was used for training
CREATE TABLE dataset_references (
    dataset_ref_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID NOT NULL REFERENCES runs(run_id),
    dataset_name    VARCHAR(256) NOT NULL,
    dataset_version VARCHAR(64),
    storage_uri     TEXT NOT NULL,
    row_count       BIGINT,
    schema_hash     VARCHAR(64),                   -- detect schema changes
    date_range      TSTZRANGE,                     -- temporal coverage
    filters_applied JSONB,                         -- any data filtering
    role            VARCHAR(32) NOT NULL,           -- 'training', 'validation', 'test'
    created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_dataset_run ON dataset_references(run_id);
CREATE INDEX idx_dataset_name ON dataset_references(dataset_name, dataset_version);

-- Feature references: which features from feature store
CREATE TABLE feature_references (
    feature_ref_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID NOT NULL REFERENCES runs(run_id),
    feature_group   VARCHAR(128) NOT NULL,
    feature_names   TEXT[] NOT NULL,
    feature_store   VARCHAR(64),                   -- 'feast', 'tecton', 'internal'
    point_in_time   BOOLEAN DEFAULT TRUE,
    as_of_timestamp TIMESTAMPTZ
);

-- Dependency graph: model-to-model dependencies
CREATE TABLE model_dependencies (
    dependency_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_version_id UUID NOT NULL REFERENCES model_versions(version_id),
    depends_on_model VARCHAR(256) NOT NULL,
    depends_on_version INT NOT NULL,
    dependency_type VARCHAR(32) NOT NULL,           -- 'embedding_model', 'preprocessor',
                                                    -- 'teacher_model', 'ensemble_member'
    created_at      TIMESTAMPTZ DEFAULT now()
);
```

### Lineage Visualization

```
┌─────────────────────────────────────────────────────────────┐
│                    Model Lineage Graph                        │
│                                                              │
│  Training Data              Feature Store         Code       │
│  ────────────              ─────────────         ────       │
│  ┌──────────┐              ┌──────────┐         git:abc123  │
│  │ClickLog  │              │user_feats│              │      │
│  │2024-01-* │              │  v3.2    │              │      │
│  └────┬─────┘              └────┬─────┘              │      │
│       │                         │                    │      │
│       └──────────┬──────────────┘                    │      │
│                  ▼                                    │      │
│           ┌──────────────┐                           │      │
│           │   Run #4521   │◄─────────────────────────┘      │
│           │  params: {    │                                  │
│           │   lr: 0.001,  │                                  │
│           │   epochs: 10  │                                  │
│           │  }            │                                  │
│           └──────┬───────┘                                  │
│                  │                                           │
│                  ▼                                           │
│         ┌────────────────┐       ┌─────────────────┐        │
│         │ rec_model v7   │──────►│  Deployment      │        │
│         │ stage: prod    │       │  env: prod-us-e1 │        │
│         │ auc: 0.847     │       │  traffic: 100%   │        │
│         │ approved: yes  │       │  since: 01/15    │        │
│         └────────────────┘       └─────────────────┘        │
│                  │                                           │
│                  ▼                                           │
│         depends on:                                          │
│         ┌────────────────┐                                  │
│         │ text-embed v2  │ (embedding model dependency)     │
│         │ stage: prod    │                                  │
│         └────────────────┘                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Deployment Metadata

```sql
-- Deployment tracking
CREATE TABLE deployments (
    deployment_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      UUID NOT NULL REFERENCES model_versions(version_id),
    environment     VARCHAR(32) NOT NULL,          -- 'staging', 'canary', 'production'
    region          VARCHAR(32),                    -- 'us-east-1', 'eu-west-1'
    endpoint_uri    TEXT,                            -- serving endpoint URL
    traffic_pct     DECIMAL(5,2) DEFAULT 100.00,
    replicas        INT,
    hardware        VARCHAR(64),                    -- 'gpu-t4-1', 'cpu-c5.2xl'
    serving_config  JSONB,                          -- batch size, timeout, etc.
    status          VARCHAR(16) DEFAULT 'deploying',
                    -- deploying, active, draining, terminated
    deployed_by     VARCHAR(64) NOT NULL,
    deployed_at     TIMESTAMPTZ DEFAULT now(),
    terminated_at   TIMESTAMPTZ,
    termination_reason TEXT
);

CREATE INDEX idx_deploy_active
    ON deployments(version_id, environment)
    WHERE status = 'active';

-- Deployment health snapshots
CREATE TABLE deployment_health (
    health_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deployment_id   UUID NOT NULL REFERENCES deployments(deployment_id),
    ts              TIMESTAMPTZ NOT NULL DEFAULT now(),
    qps             DOUBLE PRECISION,
    latency_p50_ms  DOUBLE PRECISION,
    latency_p99_ms  DOUBLE PRECISION,
    error_rate      DOUBLE PRECISION,
    gpu_utilization DOUBLE PRECISION,
    memory_usage_mb INT
);

-- Partition by time for efficient cleanup
SELECT create_hypertable('deployment_health', 'ts',
    chunk_time_interval => INTERVAL '1 day');
```

---

## Approval Workflows

### Workflow State Machine

```
┌─────────────────────────────────────────────────────────────┐
│              Model Approval Workflow                          │
│                                                              │
│  ┌──────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐ │
│  │ None │───►│ Staging  │───►│  Canary   │───►│Production│ │
│  └──────┘    └──────────┘    └───────────┘    └──────────┘ │
│                   │               │                │        │
│                   │               │                │        │
│                   ▼               ▼                ▼        │
│              ┌──────────┐   ┌──────────┐    ┌──────────┐   │
│              │ Rejected │   │ Rolled   │    │ Archived │   │
│              │          │   │ Back     │    │          │   │
│              └──────────┘   └──────────┘    └──────────┘   │
│                                                             │
│  Transition Requirements:                                   │
│  None ──► Staging:                                          │
│    • Automated: eval metrics pass thresholds                │
│    • Automated: model card generated                        │
│    • Automated: bias/fairness checks pass                   │
│                                                             │
│  Staging ──► Canary:                                        │
│    • Automated: integration tests pass                      │
│    • Automated: latency/throughput benchmarks pass           │
│    • Human: ML engineer approval                            │
│                                                             │
│  Canary ──► Production:                                     │
│    • Automated: canary metrics (no degradation in 24h)      │
│    • Human: team lead approval                              │
│    • Human: for sensitive models, privacy/legal review       │
└─────────────────────────────────────────────────────────────┘
```

### Approval Schema

```sql
CREATE TABLE approval_requests (
    approval_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      UUID NOT NULL REFERENCES model_versions(version_id),
    requested_stage VARCHAR(32) NOT NULL,
    requested_by    VARCHAR(64) NOT NULL,
    requested_at    TIMESTAMPTZ DEFAULT now(),
    status          VARCHAR(16) DEFAULT 'pending',
                    -- pending, approved, rejected, expired
    reviewed_by     VARCHAR(64),
    reviewed_at     TIMESTAMPTZ,
    review_comments TEXT,
    auto_checks     JSONB NOT NULL DEFAULT '{}',   -- automated gate results
    expires_at      TIMESTAMPTZ                     -- auto-expire if not reviewed
);

CREATE TABLE approval_checks (
    check_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    approval_id     UUID NOT NULL REFERENCES approval_requests(approval_id),
    check_name      VARCHAR(128) NOT NULL,
    check_type      VARCHAR(32) NOT NULL,           -- 'automated', 'manual'
    status          VARCHAR(16) NOT NULL,            -- 'passed', 'failed', 'skipped'
    details         JSONB,
    executed_at     TIMESTAMPTZ DEFAULT now()
);
```

---

## Model Card Generation

### Model Card Schema

```sql
CREATE TABLE model_cards (
    card_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id      UUID NOT NULL REFERENCES model_versions(version_id),
    -- Model Details
    model_type      VARCHAR(64),                    -- 'classification', 'ranking', 'regression'
    framework       VARCHAR(32),                    -- 'pytorch', 'tensorflow'
    architecture    TEXT,                            -- 'transformer-base', 'gradient-boosted-trees'
    -- Intended Use
    primary_use     TEXT NOT NULL,
    out_of_scope    TEXT,
    -- Training Data
    training_data_description TEXT NOT NULL,
    training_data_size BIGINT,
    -- Evaluation
    eval_metrics    JSONB NOT NULL,                 -- { "accuracy": 0.94, "auc": 0.87 }
    eval_slices     JSONB,                          -- per-segment performance
    -- Fairness & Bias
    fairness_metrics JSONB,                         -- demographic parity, equalized odds
    known_biases    TEXT,
    mitigation_steps TEXT,
    -- Limitations
    known_limitations TEXT,
    -- Compute
    training_compute JSONB,                         -- GPU hours, cost, carbon
    inference_compute JSONB,                        -- latency, throughput, cost/query
    -- Metadata
    generated_at    TIMESTAMPTZ DEFAULT now(),
    generated_by    VARCHAR(16) DEFAULT 'automated',
    reviewed_by     VARCHAR(64),
    status          VARCHAR(16) DEFAULT 'draft'     -- draft, published
);
```

### Model Card Example (Generated)

```
┌──────────────────────────────────────────────────────────────┐
│                    MODEL CARD                                 │
│  Model: recommendation_ranker v7                             │
│  Generated: 2024-01-20                                       │
│                                                              │
│  PURPOSE: Rank candidate items for personalized              │
│  recommendations in the home feed.                           │
│                                                              │
│  ARCHITECTURE: Two-tower transformer with cross-attention    │
│  PARAMETERS: 125M                                            │
│  FRAMEWORK: PyTorch 2.1                                      │
│                                                              │
│  TRAINING DATA:                                              │
│  • 90 days of user interaction logs (Oct-Dec 2023)           │
│  • 2.1B training examples, 50M validation                    │
│  • Features from user_engagement_v3, item_embedding_v2       │
│                                                              │
│  EVALUATION METRICS:                                         │
│  ┌──────────────┬──────────┬──────────┬───────────┐          │
│  │ Metric       │ Overall  │ New Users│ Power Users│          │
│  ├──────────────┼──────────┼──────────┼───────────┤          │
│  │ nDCG@10      │ 0.412    │ 0.318    │ 0.487     │          │
│  │ MRR          │ 0.389    │ 0.291    │ 0.452     │          │
│  │ Click-through│ 12.4%    │ 8.1%     │ 15.7%     │          │
│  └──────────────┴──────────┴──────────┴───────────┘          │
│                                                              │
│  FAIRNESS:                                                   │
│  • Demographic parity across age groups: 0.97 (>0.95 pass)  │
│  • No significant performance gap across regions             │
│  • Known gap: lower nDCG for users with <5 interactions      │
│                                                              │
│  COMPUTE:                                                    │
│  • Training: 256 A100-hours ($10,240), ~500 kg CO2e          │
│  • Inference: 2.1ms p50, 8.3ms p99 on T4 GPU                │
│  • Cost: $0.0001 per prediction                              │
│                                                              │
│  LIMITATIONS:                                                │
│  • Cold-start: poor ranking for items with <100 impressions  │
│  • Recency bias: over-weights recent interactions            │
│  • Not suitable for: safety-critical recommendations         │
└──────────────────────────────────────────────────────────────┘
```

---

## Scaling to Thousands of Models/Experiments

### Performance Considerations

| Scale | Runs | Metrics Rows | Artifacts | DB Strategy |
|---|---|---|---|---|
| Small (<100 models) | 10K | 100M | 50K | Single PostgreSQL |
| Medium (100-1000) | 100K | 1B | 500K | PostgreSQL + read replicas |
| Large (1000-5000) | 1M | 10B | 5M | Partitioned PG + TimescaleDB for metrics |
| FAANG (5000+) | 10M+ | 100B+ | 50M+ | Sharded PG + ClickHouse for metrics + S3 |

### Scaling Strategy

```
┌──────────────────────────────────────────────────────────────┐
│           Registry at FAANG Scale                             │
│                                                              │
│  Metadata (PostgreSQL, sharded by project)                   │
│  ├── projects, experiments, runs, model_versions             │
│  ├── Shard key: project_id (team-level isolation)            │
│  ├── ~10M rows per shard, 16-32 shards                       │
│  └── Read replicas per shard for query load                  │
│                                                              │
│  Run Metrics (ClickHouse)                                    │
│  ├── Billions of metric data points                          │
│  ├── Columnar storage, excellent compression                 │
│  ├── Sub-second aggregation queries                          │
│  └── Materialized views for common dashboards               │
│                                                              │
│  Artifacts (S3 + metadata in PostgreSQL)                     │
│  ├── Models: s3://model-artifacts/{model_name}/{version}/    │
│  ├── Lifecycle rules: move to IA after 90 days              │
│  ├── Versioned buckets for rollback safety                   │
│  └── Cross-region replication for DR                         │
│                                                              │
│  Search/Discovery (Elasticsearch)                            │
│  ├── Index experiment descriptions, tags, parameters         │
│  ├── Full-text search for finding related experiments        │
│  └── Faceted filtering by team, model type, date range       │
└──────────────────────────────────────────────────────────────┘
```

### Query Optimization Patterns

```sql
-- Problem: "Show me the best run for each experiment in my project"
-- Naive approach scans all runs. At 10M runs, this is slow.

-- Optimized: materialized view refreshed hourly
CREATE MATERIALIZED VIEW best_runs_per_experiment AS
SELECT DISTINCT ON (r.experiment_id)
    r.experiment_id,
    r.run_id,
    r.start_time,
    r.parameters,
    rm.value as primary_metric_value
FROM runs r
JOIN run_metrics rm ON r.run_id = rm.run_id
    AND rm.key = (
        SELECT e.primary_metric FROM experiments e
        WHERE e.experiment_id = r.experiment_id
    )
WHERE r.status = 'completed'
ORDER BY r.experiment_id, rm.value DESC;

CREATE UNIQUE INDEX idx_best_runs ON best_runs_per_experiment(experiment_id);
REFRESH MATERIALIZED VIEW CONCURRENTLY best_runs_per_experiment;

-- Problem: "Find all models in production that depend on embedding-model v1"
-- Critical for impact analysis during model migrations
SELECT
    mv.model_name,
    mv.version_number,
    d.environment,
    d.traffic_pct
FROM model_dependencies md
JOIN model_versions mv ON md.model_version_id = mv.version_id
JOIN deployments d ON d.version_id = mv.version_id
WHERE md.depends_on_model = 'text-embedding-model'
    AND md.depends_on_version = 1
    AND d.status = 'active';
```

---

## Cost of Model Registry at Scale

| Component | Specification | Monthly Cost |
|---|---|---|
| PostgreSQL (sharded, 4 shards) | db.r6g.2xlarge x 8 (primary + replica) | $12,000 |
| ClickHouse (metrics) | 3 nodes, 16-core, 128GB | $9,000 |
| Elasticsearch (search) | 3 nodes, 16-core, 64GB | $5,000 |
| S3 (artifacts, 100TB) | Standard + IA lifecycle | $2,500 |
| API servers (registry service) | 8 pods, c5.xlarge | $3,000 |
| CI/CD infrastructure | Approval workflows, testing | $2,000 |
| **Total** | | **$33,500** |

**Key cost insight:** The artifacts (S3) are the largest physical volume but cheapest component.
The compute databases (PostgreSQL, ClickHouse) are the expensive part. Aggressive archiving of
old experiment metrics to S3 Parquet can save 40-50% on ClickHouse costs.

---

## Interview Narrative

**When asked about model registry design in a Staff/Principal interview:**

> "I'd design the model registry as the single source of truth for the entire ML lifecycle,
> built on PostgreSQL for metadata with ClickHouse for high-volume metrics storage.
>
> The core data model has five key entities: projects group related work by team, experiments
> represent hypotheses, runs capture individual training executions with their parameters and
> code version, model versions are the promoted artifacts from successful runs, and
> deployments track where each version is serving traffic.
>
> The most critical design decisions are around lineage and governance. Every model version
> must reference its training data, feature store versions, and upstream model dependencies.
> This lineage graph is what enables impact analysis — when the embedding model team wants
> to deprecate v1, we can instantly find every production model that depends on it and
> coordinate the migration.
>
> For the approval workflow, I'd implement a stage-based system: None, Staging, Canary,
> Production. Each transition has automated gates — evaluation metrics must exceed thresholds,
> bias checks must pass, latency benchmarks must meet SLOs — plus human approval gates for
> the critical transitions. The canary-to-production transition requires both automated
> verification of no metric degradation during the canary period and explicit team lead sign-off.
>
> Model cards get auto-generated at registration time with training data descriptions,
> evaluation metrics broken down by population slices, fairness metrics, and compute cost.
> This isn't just good practice — it's a compliance requirement at FAANG for models that
> affect user-facing decisions.
>
> At FAANG scale with 5000+ models and millions of runs, the main scaling challenge is the
> metrics table — billions of rows of training loss curves and evaluation metrics. I'd move
> this to ClickHouse with TTL-based compaction, keeping detailed metrics for 90 days and
> only final evaluation metrics forever. The metadata tables stay in PostgreSQL sharded by
> project, which keeps each shard manageable at around 10M rows.
>
> For discovery, I'd index experiment metadata in Elasticsearch so data scientists can search
> across the organization for related work — 'who else has trained a ranking model on
> click data in the last 6 months?' This is what transforms a registry from a team tool
> into an organizational knowledge base."

**Key signals this demonstrates:**
- Complete mental model of ML lifecycle entities and their relationships
- Governance mindset (approval workflows, lineage, model cards)
- Practical scaling strategy (PostgreSQL sharding, ClickHouse for metrics)
- Organizational thinking (cross-team discovery, impact analysis)
- Compliance awareness (fairness checks, audit trails)
- Cost-consciousness with specific optimization strategies
