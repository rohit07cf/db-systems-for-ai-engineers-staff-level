# Design Storage for an LLM Serving Platform (OpenAI API Scale)

## Staff/Principal-Level System Design Exercise

---

## 1. Problem Statement

Design the complete storage subsystem for a large-scale LLM serving platform
handling 1M+ API calls per minute. The platform serves multiple model families
(GPT-class, embedding, image generation), supports multi-turn conversations,
enforces rate limits and billing, caches semantically similar prompts, logs all
interactions for safety/compliance, and routes requests to the optimal model
backend.

At staff level, the challenge is not "how to store logs" — it is reasoning about
the tension between low-latency serving (<200ms TTFT), compliance requirements
(store everything), cost optimization (semantic caching saves millions in GPU
compute), and the unique data shapes of LLM workloads (variable-length sequences,
streaming completions, token-level billing granularity).

---

## 2. Requirements

### 2.1 Functional Requirements

- **Prompt/Completion Logging**: Every request/response pair stored immutably for audit.
- **Token Usage Tracking**: Per-request token counts (prompt + completion) for billing.
- **Rate Limiting**: Per-API-key, per-org, per-model, and global rate enforcement.
- **Model Routing Metadata**: Model versions, endpoint health, capacity, and routing rules.
- **Conversation Context**: Multi-turn state management with configurable context windows.
- **Semantic Cache**: Cache responses for semantically similar prompts to reduce GPU cost.
- **Content Moderation Logs**: Store moderation decisions, flagged content, and appeals.
- **Billing Data**: Real-time metering, invoicing, and usage dashboards.

### 2.2 Non-Functional Requirements

- **Throughput**: 1M+ API calls/min = ~17K requests/sec sustained.
- **Latency**: Storage operations must not add > 5ms to the critical path.
- **Durability**: Zero data loss for billing records (financial compliance).
- **Availability**: 99.99% for the serving path; 99.9% acceptable for analytics.
- **Retention**: Logs retained 90 days (default), billing data 7 years.
- **Compliance**: SOC2, GDPR (data deletion), CCPA, and emerging AI regulations.

---

## 3. Scale Numbers

| Metric | Value |
|---|---|
| API calls/minute | 1M+ |
| API calls/sec (sustained) | ~17K |
| API calls/sec (peak, 5x) | ~85K |
| Avg prompt tokens | 500 |
| Avg completion tokens | 300 |
| Avg request payload | ~4 KB (prompt + completion text) |
| Total log data/day | ~5.8 TB |
| Unique API keys | ~5M |
| Active organizations | ~500K |
| Models served | ~50 (across families) |
| Semantic cache hit rate (target) | 15-25% |
| Token events for billing/day | ~14B tokens |
| Conversations with context | ~200M active |
| Moderation checks/day | ~1.5B |

---

## 4. High-Level Architecture

```
                    ┌─────────────────────────────┐
                    │       API Gateway / LB       │
                    └──────────┬──────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
   ┌──────▼──────┐    ┌───────▼──────┐    ┌───────▼──────┐
   │ Rate Limiter │    │   Semantic   │    │   Request    │
   │ (Redis       │    │   Cache      │    │   Router     │
   │  Cluster)    │    │   Lookup     │    │              │
   └──────┬──────┘    └───────┬──────┘    └───────┬──────┘
          │                   │                    │
          │            ┌──────▼──────┐     ┌──────▼──────┐
          │            │  Cache Hit? │     │ Model Pool  │
          │            │  Yes → Return│     │ (GPU Fleet) │
          │            │  No → Route │     └──────┬──────┘
          │            └─────────────┘            │
          │                                       │
   ┌──────▼───────────────────────────────────────▼──────┐
   │                   Async Write Path                    │
   │                   (Kafka Cluster)                     │
   └──┬──────────┬──────────┬──────────┬────────────┬────┘
      │          │          │          │            │
 ┌────▼───┐ ┌───▼────┐ ┌───▼────┐ ┌───▼─────┐ ┌───▼─────┐
 │ Log    │ │ Token  │ │ Billing│ │ Content │ │ Cache   │
 │ Store  │ │ Meter  │ │ Agg    │ │ Mod Log │ │ Writer  │
 │(S3+CK) │ │(CK/TS) │ │(Pg/CK)│ │ (CK)   │ │(Redis+  │
 │        │ │        │ │        │ │         │ │ Milvus) │
 └────────┘ └────────┘ └────────┘ └─────────┘ └─────────┘

 CK = ClickHouse, Pg = PostgreSQL, TS = TimescaleDB
```

---

## 5. Storage Layer Design

### 5.1 Prompt/Completion Log Storage

**Two-tier approach:**

**Tier 1: Hot logs in ClickHouse (0-30 days)**

```sql
CREATE TABLE api_logs (
    log_id          UUID,
    org_id          UInt64,
    api_key_id      UInt64,
    request_ts      DateTime64(3),
    model_id        LowCardinality(String),
    model_version   LowCardinality(String),
    prompt_hash     FixedString(32),      -- SHA-256 for dedup/cache key
    prompt_tokens   UInt32,
    completion_tokens UInt32,
    total_tokens    UInt32,
    latency_ms      UInt32,
    ttft_ms         UInt32,               -- Time to first token
    status_code     UInt16,
    finish_reason   LowCardinality(String),
    prompt_text     String CODEC(ZSTD(3)),
    completion_text String CODEC(ZSTD(3)),
    metadata        String CODEC(ZSTD(3)),-- JSON: temperature, top_p, etc.
    region          LowCardinality(String),
    INDEX idx_prompt_hash prompt_hash TYPE bloom_filter GRANULARITY 4,
    INDEX idx_org org_id TYPE minmax GRANULARITY 4
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(request_ts)
ORDER BY (org_id, model_id, request_ts)
TTL request_ts + INTERVAL 30 DAY TO VOLUME 'cold';
```

**Why ClickHouse?**
- Column-oriented: Queries like "total tokens by org by model by day" scan only
  the needed columns, achieving 10-100x compression on repetitive fields.
- ZSTD compression on prompt/completion text: ~5x compression ratio. 5.8 TB/day
  compresses to ~1.2 TB/day on disk.
- Materialized views for pre-aggregated metrics (no need to scan raw logs for dashboards).

**Tier 2: Cold logs in S3 Parquet (30+ days)**
- Daily export from ClickHouse to Parquet files partitioned by `(org_id, date)`.
- Queryable via Trino/Presto for compliance audits.
- Cost: $0.023/GB/month (S3 Standard) vs $0.25/GB/month (ClickHouse on NVMe).

### 5.2 Token Usage Tracking and Real-Time Metering

```sql
-- ClickHouse materialized view for real-time aggregation
CREATE MATERIALIZED VIEW token_usage_minutely
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMMDD(minute)
ORDER BY (org_id, api_key_id, model_id, minute)
AS SELECT
    org_id,
    api_key_id,
    model_id,
    toStartOfMinute(request_ts) AS minute,
    sum(prompt_tokens) AS prompt_tokens_sum,
    sum(completion_tokens) AS completion_tokens_sum,
    count() AS request_count,
    sum(latency_ms) AS total_latency_ms
FROM api_logs
GROUP BY org_id, api_key_id, model_id, minute;
```

**Why SummingMergeTree?**
- Automatically merges rows with the same ORDER BY key, summing numeric columns.
- A query for "tokens used by org X in the last hour" scans at most 60 pre-aggregated
  rows instead of millions of raw log rows.
- Latency for billing dashboards: < 50ms even for high-volume orgs.

### 5.3 Rate Limiting Storage (Redis Cluster)

```
-- Sliding window rate limiting using Redis sorted sets
ZADD ratelimit:{api_key}:{model} {timestamp_ms} {request_id}
ZREMRANGEBYSCORE ratelimit:{api_key}:{model} 0 {timestamp_ms - window_ms}
ZCARD ratelimit:{api_key}:{model}
-- If ZCARD > limit, reject with 429

-- Token-based rate limiting (more nuanced)
-- Track tokens consumed in rolling window
INCRBY tokens:{org_id}:{minute_bucket} {tokens_consumed}
EXPIRE tokens:{org_id}:{minute_bucket} 120

-- Tiered limits stored in a config hash
HGETALL rate_config:{org_id}
-- Returns: { "rpm": "10000", "tpm": "2000000", "concurrent": "500" }
```

**Design decisions:**
- **Request-level AND token-level limits**: A single request can consume 32K tokens
  (a full context window). Request count alone is insufficient.
- **Hierarchical enforcement**: Global -> Org -> API Key -> Model.
  Check from most specific to least specific; first limit hit triggers 429.
- **Graceful degradation**: If Redis is unavailable, fall back to local in-memory
  rate limiting (pessimistic: lower limits, per-instance). Never fail open at the
  rate limiter — a burst of unmetered requests can bankrupt GPU capacity.

### 5.4 Model Routing Metadata (PostgreSQL + In-Memory Cache)

```sql
CREATE TABLE model_registry (
    model_id        TEXT PRIMARY KEY,
    model_family    TEXT NOT NULL,          -- gpt-4, claude, etc.
    model_version   TEXT NOT NULL,
    context_window  INT NOT NULL,
    max_output      INT NOT NULL,
    input_price     NUMERIC(10, 6),        -- per 1K tokens
    output_price    NUMERIC(10, 6),
    endpoints       JSONB,                 -- [{"region": "us-east", "url": "...", "capacity": 1000}]
    capabilities    JSONB,                 -- {"vision": true, "function_calling": true}
    status          TEXT DEFAULT 'active', -- active, deprecated, shadow
    created_at      TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ
);

CREATE TABLE routing_rules (
    rule_id         SERIAL PRIMARY KEY,
    org_id          BIGINT,
    model_pattern   TEXT,                  -- "gpt-4*" or specific model_id
    routing_policy  JSONB,                 -- {"strategy": "least_loaded", "fallback": "gpt-4-mini"}
    priority        INT,
    active          BOOLEAN DEFAULT true
);
```

**Why PostgreSQL?**
- Model registry is low-volume (50 models, updated rarely) but requires
  transactional correctness (do not route to a model that was just deactivated).
- JSONB for flexible endpoint and capability schemas that evolve frequently.
- Replicated to each region's in-memory cache (refresh every 5s via CDC).

### 5.5 Conversation Context Management

```
-- Conversations stored in Redis (active) + DynamoDB (durable)

-- Redis: Hot conversation context (last 1 hour of activity)
SET conv:{conversation_id} {serialized_messages} EX 3600

-- DynamoDB: Durable conversation store
Table: conversations
  PK: org_id
  SK: conversation_id
  Attributes:
    created_at, updated_at, model_id, system_prompt,
    message_count, total_tokens, metadata

Table: conversation_messages
  PK: conversation_id
  SK: message_idx (int, sequential)
  Attributes:
    role (system/user/assistant/tool),
    content (text, compressed),
    token_count,
    timestamp,
    finish_reason
```

**Context window management:**
- When conversation exceeds model's context window, apply truncation strategy:
  1. Always keep system prompt.
  2. Keep last N messages that fit within `context_window - max_output - system_prompt_tokens`.
  3. Optionally: summarize dropped messages via a smaller model and prepend summary.
- **Token counting on write**: Pre-compute and store `token_count` per message
  (using tiktoken). Avoids re-tokenizing the full conversation on every request.

### 5.6 Semantic Cache (Vector Store + Redis)

```
Architecture:
  1. On request: Compute embedding of prompt using a small/fast embedding model.
  2. Query vector index for nearest neighbor within cosine similarity > 0.98.
  3. If hit: Check that model, temperature, and system prompt match exactly.
  4. If all match: Return cached completion. Log as cache hit.
  5. If miss: Route to GPU, store result in cache with embedding.

Storage:
  -- Vector index: Milvus/Qdrant cluster
  Collection: semantic_cache
    Fields:
      cache_key (UUID), prompt_embedding (float32[1536]), model_id,
      temperature, system_prompt_hash, created_at

  -- Cache values: Redis
  GET cache_val:{cache_key}
  -- Returns: serialized completion, token counts, finish_reason
  -- TTL: 1 hour (configurable per org)
```

**Why separate vector index from value store?**
- Vector similarity search is compute-intensive (ANN). Milvus/Qdrant are optimized
  for this with HNSW/IVF indices.
- The actual cached completions are large (KBs). Storing them in the vector DB
  would bloat the index and slow down search.
- Redis provides sub-millisecond value retrieval after the vector lookup.

**Cache economics:**
- Embedding model cost: ~$0.0001 per request.
- GPU inference cost (cache miss): ~$0.01-0.10 per request (depending on model).
- At 20% cache hit rate on 1M req/min: saves ~200K GPU inferences/min.
- Annual savings estimate: $50-200M in GPU compute.

### 5.7 Content Moderation Log Storage

```sql
-- ClickHouse table for moderation decisions
CREATE TABLE moderation_logs (
    request_id      UUID,
    org_id          UInt64,
    request_ts      DateTime64(3),
    moderation_ts   DateTime64(3),
    input_flagged   Bool,
    output_flagged  Bool,
    categories      Array(LowCardinality(String)),  -- ['violence', 'sexual']
    category_scores Map(String, Float32),
    action_taken    LowCardinality(String),         -- 'allowed', 'blocked', 'warned'
    model_id        LowCardinality(String),
    appeal_status   LowCardinality(String),         -- null, 'pending', 'upheld', 'overturned'
    reviewer_id     Nullable(UInt64),
    review_notes    Nullable(String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(request_ts)
ORDER BY (org_id, request_ts)
TTL request_ts + INTERVAL 2 YEAR;
```

**Why keep moderation logs separate from API logs?**
- Different access patterns: API logs queried by org for billing; moderation logs
  queried by safety team across all orgs for trend analysis.
- Different retention: Moderation logs kept 2+ years for regulatory compliance;
  API logs may be deleted per GDPR request.
- Different access controls: Moderation logs contain sensitive decisions; access
  restricted to trust & safety team with audit trail.

### 5.8 Billing Data Pipeline

```
Request → Kafka → Token Meter (ClickHouse MV) → Billing Aggregator → PostgreSQL

PostgreSQL billing tables:
  usage_daily:    org_id, date, model_id, prompt_tokens, completion_tokens, cost
  invoices:       org_id, period_start, period_end, total_cost, status, stripe_id
  credit_ledger:  org_id, txn_id, amount, type (credit/debit), balance, timestamp
```

**Double-entry bookkeeping for token billing:**
- Every token consumed creates a debit entry in the credit_ledger.
- Every payment/credit purchase creates a credit entry.
- Running balance = SUM(credits) - SUM(debits).
- This model provides auditability and makes billing disputes trivially resolvable.

**Reconciliation pipeline:**
- Hourly: Compare ClickHouse aggregated token counts with PostgreSQL billing records.
- Flag discrepancies > 0.1% for manual review.
- This catches dropped Kafka messages, double-counting, or billing logic bugs.

---

## 6. Detailed Component Design

### 6.1 Request Critical Path (Latency-Sensitive)

```
Request arrives (t=0)
  │
  ├── [1] Rate limit check (Redis)           ~1ms
  │
  ├── [2] Semantic cache lookup               ~5ms
  │     ├── Embed prompt                      ~3ms
  │     └── Vector similarity search          ~2ms
  │
  ├── [3a] Cache HIT → Return cached result   ~1ms
  │    Total: ~7ms overhead
  │
  └── [3b] Cache MISS → Route to GPU          ~50-5000ms (model dependent)
           │
           ├── [4] Produce to Kafka (async)   ~0ms on critical path
           │       (fire-and-forget with acks=1)
           │
           └── [5] Stream response to client
```

**Critical insight**: The storage layer MUST NOT be on the critical path for
inference requests. All durable writes happen asynchronously via Kafka.
The only synchronous storage operations are:
1. Rate limit check (Redis, 1ms).
2. Semantic cache lookup (Redis + vector DB, 5ms).
3. Conversation context fetch (Redis, 1ms — preloaded for multi-turn).

### 6.2 Streaming Completion Storage

LLM responses arrive as a stream of tokens. Storage challenges:
- Cannot compute total token count until stream completes.
- Client may disconnect mid-stream (partial completion).
- Must store the final assembled text, not individual tokens.

**Solution:**
1. Accumulate tokens in a streaming buffer (in-memory, per-request).
2. On stream completion (or disconnect), compute final token count.
3. Produce a single log event to Kafka with complete request + response.
4. If client disconnects: log `finish_reason = "client_disconnect"`,
   bill for tokens generated up to that point.

---

## 7. Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Redis cluster failure | Rate limits unenforced; cache misses | Local fallback rate limiter (conservative); route all to GPU (cost spike) |
| Kafka broker failure | Logs not persisted; billing delayed | Producer retries with exponential backoff; local disk buffer as WAL |
| ClickHouse node failure | Dashboard queries fail | 2x replication; queries auto-route to replica |
| Semantic cache poisoning | Wrong responses served | Exact match on model + temperature + system_prompt_hash; cache TTL; org-scoped cache isolation |
| Vector DB latency spike | Cache lookup exceeds budget | Circuit breaker: skip cache if lookup > 10ms; proceed to GPU |
| Billing data loss | Revenue leakage | WAL to local disk before Kafka produce; reconciliation pipeline catches gaps |
| DynamoDB throttle | Conversation context unavailable | DAX cache layer; exponential backoff; pre-provision for known peaks |
| Model registry stale | Requests routed to deprecated model | Health checks every 5s; immediate push invalidation on model status change |
| Log storage full | Compliance violation | Alert on 80% capacity; auto-expand; S3 tier absorbs overflow |
| Token count mismatch | Billing disputes | Server-side re-tokenization for billing (canonical); client counts are advisory only |

---

## 8. Cost Analysis

| Component | Monthly Cost Estimate |
|---|---|
| ClickHouse cluster (hot logs, 3 replicas) | $180K |
| S3 (cold logs, 90-day retention) | $50K |
| Redis cluster (rate limits + cache values) | $120K |
| Vector DB (semantic cache, Milvus) | $80K |
| Kafka cluster | $100K |
| PostgreSQL (billing, routing) | $20K |
| DynamoDB (conversations) | $150K |
| Network / cross-region | $60K |
| **Total storage infrastructure** | **~$760K/month** |

**Compare to GPU inference cost**: At 1M req/min with avg $0.01/req GPU cost,
inference costs ~$14.4M/month. Storage is ~5% of total infrastructure cost.

**Semantic cache ROI**: 20% hit rate saves ~$2.9M/month in GPU costs. The entire
semantic cache infrastructure (Redis + Milvus) costs $200K/month.
ROI = 14.5x. This is the single highest-leverage optimization.

---

## 9. Evolution Path (Handling 10x Growth)

### 9.1 From 1M to 10M Requests/Min

- **Rate limiting**: Shard Redis by org_id hash. 10x nodes.
- **Logging**: ClickHouse scales linearly with nodes. Add shards.
- **Kafka**: Increase partition count. Consider tiered storage (Kafka + S3) to
  reduce broker disk costs.
- **Semantic cache**: Cache hit rate improves with more traffic (more prompt overlap).
  May need to shard Milvus collection by model_id.

### 9.2 Multi-Model, Multi-Modality Expansion

- **Image/audio inputs**: Log storage shifts from text to binary. Move to S3-first
  with ClickHouse storing metadata + S3 references.
- **Different billing models**: Image models bill per image, not per token. Extend
  the billing schema with a `billing_unit` field (tokens, images, seconds, characters).
- **Routing complexity**: Model registry grows. Add a dedicated routing service
  with ML-based model selection (cost vs quality optimization).

### 9.3 Real-Time Fine-Tuning Data Pipeline

- Customer requests become training data (with consent).
- Add a CDC stream from ClickHouse to a feature store.
- Schema evolution: Add `training_eligible`, `quality_score`, `human_feedback`
  columns to API logs.
- This turns the logging infrastructure into a data flywheel.

### 9.4 Enterprise Data Isolation

- Large enterprise customers demand dedicated storage.
- **Approach**: Logical isolation (separate ClickHouse databases, separate Redis
  key prefixes) for most customers. Physical isolation (separate clusters) for
  top-tier enterprise contracts.
- Storage routing layer maps `org_id -> storage_cluster_id`.

---

## 10. Interview Narrative

### How to Present This in 45 Minutes

**Minutes 0-5: Frame the problem.** "This is fundamentally a high-throughput
logging and metering system with a real-time serving component (rate limiting,
caching). The key insight is that storage must NEVER be on the inference critical
path. All durable writes are async."

**Minutes 5-15: Critical path design.** Draw the request flow. Show that only
Redis (rate limit) and vector DB (semantic cache) are synchronous. Explain why
Kafka decouples the write path. This demonstrates that you think about latency
budgets, not just correctness.

**Minutes 15-25: Storage technology choices.** Walk through ClickHouse for logs
(column-oriented for analytics), Redis for rate limiting (sub-ms reads), PostgreSQL
for billing (transactional consistency), vector DB for semantic cache (ANN search).
Each choice should be justified by the access pattern, not by personal preference.

**Minutes 25-35: Semantic cache deep dive.** This is the star of the design. Explain
the embedding + ANN + exact-match-guard pipeline. Walk through the cache economics
($2.9M/month savings at 20% hit rate). Discuss cache invalidation (TTL-based,
model version change, org-specific isolation). This shows business acumen.

**Minutes 35-45: Billing correctness and failure modes.** Double-entry bookkeeping
for token billing. Reconciliation pipeline. What happens when Kafka drops messages
(answer: WAL buffer + reconciliation catches it). This shows you think about the
boring-but-critical path that determines whether the company gets paid.

### Key Phrases That Signal Staff-Level Thinking

- "The storage layer is not the bottleneck — GPU inference is. Our job is to stay
  off the critical path while guaranteeing zero data loss for billing."
- "Semantic caching has a 14.5x ROI. At this scale, the cache infrastructure pays
  for itself in 2 days of GPU savings."
- "I use double-entry bookkeeping for token billing because financial data requires
  auditability, not just correctness. Every token consumed has a corresponding
  ledger entry."
- "The moderation logs are stored separately from API logs because they have
  different access patterns, retention requirements, and access control models.
  Mixing them violates the principle of separation of concerns at the data layer."
- "When Redis is down, we fail closed on rate limiting (reject requests) but fail
  open on caching (skip cache, route to GPU). The cost asymmetry dictates the
  failure mode: a billing error is worse than a cache miss."
