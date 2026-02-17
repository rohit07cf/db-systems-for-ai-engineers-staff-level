# Redis for LLM and AI Systems — Staff/Principal Deep Dive

## 1. Redis in the AI/ML Infrastructure Stack

```
  +------------------------------------------------------------------+
  |                     AI/ML System Architecture                     |
  +------------------------------------------------------------------+
  |                                                                    |
  |  User Request                                                      |
  |       |                                                            |
  |       v                                                            |
  |  +----+-------+     +----------------+     +------------------+   |
  |  | API Gateway |---->| Rate Limiter   |---->| Semantic Cache   |   |
  |  | (Auth/Route)|     | (Redis)        |     | (Redis + Vector) |   |
  |  +-------------+     +--------+-------+     +--------+---------+   |
  |                               |                      |             |
  |                          if allowed            cache hit?          |
  |                               |              /           \         |
  |                               v           YES             NO       |
  |                      +--------+------+     |               |       |
  |                      | Feature Store |     |               v       |
  |                      | Serving Layer |     |    +----------+---+   |
  |                      | (Redis)       |     |    | LLM Inference|   |
  |                      +--------+------+     |    | Service      |   |
  |                               |            |    +----------+---+   |
  |                               v            |               |       |
  |                      +--------+------+     |               v       |
  |                      | Model Serving |     |    +----------+---+   |
  |                      | (TF Serving / |     |    | Cache Result  |   |
  |                      |  vLLM/TGI)    |     |    | in Redis      |   |
  |                      +---------------+     |    +--------------+   |
  |                                            |               |       |
  |                                            +-------+-------+       |
  |                                                    |               |
  |                                                    v               |
  |                                            +-------+-------+       |
  |                                            | Session Store  |       |
  |                                            | (Redis)        |       |
  |                                            | Chat History   |       |
  |                                            +---------------+       |
  +------------------------------------------------------------------+
```

---

## 2. Semantic Caching for LLM Responses

### Why Semantic Cache?

```
  Traditional cache: exact key match
    "What is the capital of France?" --> "Paris"
    "what's the capital of france?"  --> MISS (different string!)

  Semantic cache: similarity-based match
    "What is the capital of France?" --> embedding --> [0.12, 0.85, ...]
    "what's the capital of france?"  --> embedding --> [0.11, 0.84, ...]
    Cosine similarity: 0.98 > threshold 0.95 --> CACHE HIT!

  LLM API call cost:
    GPT-4 class: ~$0.03-0.06 per 1K tokens
    At 1M requests/day with avg 500 tokens:
      Without cache: $15,000-30,000/day
      With 40% cache hit rate: $9,000-18,000/day
      Savings: $6,000-12,000/day ($2M-4M/year)
```

### Semantic Cache Architecture with Redis

```
  +------------------------------------------------------------------+
  |  Semantic Cache Flow                                              |
  |                                                                    |
  |  User Query: "Explain photosynthesis briefly"                     |
  |       |                                                            |
  |       v                                                            |
  |  +----+----------+                                                |
  |  | Embedding     |  query_embedding = embed("Explain photo...")   |
  |  | Model (small) |  Latency: 5-20ms (local model or API)         |
  |  +----+----------+                                                |
  |       |                                                            |
  |       v                                                            |
  |  +----+--------------------------------------------+              |
  |  | Redis Search (FT.SEARCH)                        |              |
  |  |                                                  |              |
  |  | FT.SEARCH cache_idx                             |              |
  |  |   "(@embedding:[VECTOR_RANGE $radius $blob])"   |              |
  |  |   PARAMS 4 radius 0.05 blob <query_embedding>  |              |
  |  |   SORTBY __embedding_score ASC                  |              |
  |  |   LIMIT 0 1                                     |              |
  |  +----+-----------+--------------------------------+              |
  |       |           |                                                |
  |    HIT (d<0.05)  MISS (d>=0.05)                                   |
  |       |           |                                                |
  |       v           v                                                |
  |  Return cached   Call LLM --> Store result:                       |
  |  response         HSET cache:{hash}                               |
  |  (< 1ms)           prompt "Explain photosynthesis briefly"        |
  |                     response "Photosynthesis is the process..."   |
  |                     embedding <binary_vector>                     |
  |                     timestamp 1706140800                          |
  |                     model "gpt-4"                                 |
  |                     tokens_used 250                               |
  +------------------------------------------------------------------+
```

### Redis Search Index for Semantic Cache

```
  # Create vector similarity index
  FT.CREATE cache_idx ON HASH PREFIX 1 cache:
    SCHEMA
      prompt TEXT
      response TEXT NOINDEX        # Don't index response text
      embedding VECTOR HNSW 6     # HNSW algorithm
        TYPE FLOAT32               # 32-bit floats
        DIM 1536                   # OpenAI ada-002 dimensions
        DISTANCE_METRIC COSINE     # Cosine similarity
      timestamp NUMERIC SORTABLE
      model TAG
      tokens_used NUMERIC

  # Query for similar prompts
  FT.SEARCH cache_idx
    "*=>[KNN 5 @embedding $query_vec AS score]"
    PARAMS 2 query_vec <binary_embedding>
    RETURN 3 prompt response score
    SORTBY score ASC
    LIMIT 0 1
    DIALECT 2
```

### Semantic Cache Design Decisions

| Decision               | Option A                       | Option B                         | Recommendation              |
|------------------------|--------------------------------|----------------------------------|-----------------------------|
| Similarity threshold   | 0.95 (strict)                  | 0.85 (loose)                     | 0.92-0.95 for factual, 0.85-0.90 for creative |
| Embedding model        | OpenAI ada-002 (1536d)         | Local sentence-transformers (384d)| Local for latency, API for quality |
| Index algorithm        | HNSW (approximate)             | FLAT (exact, brute force)        | HNSW for > 100K entries     |
| TTL policy             | Global TTL (24h)               | Per-model TTL (varies)           | Per-model (GPT-4: 7d, GPT-3.5: 1d) |
| Cache key              | Hash of prompt                 | Auto-generated ID                | Hash (enables exact dedup)  |
| Invalidation           | TTL only                       | TTL + model version tracking     | Track model version for correctness |

---

## 3. Feature Store Serving Layer

### Online Feature Serving with Redis

```
  ML Pipeline:
  +------------+     +-----------+     +------------+
  | Feature    |---->| Feature   |---->| Redis      |
  | Engineering|     | Transform |     | (Online    |
  | (Spark/    |     | Pipeline  |     |  Feature   |
  |  Flink)    |     |           |     |  Store)    |
  +------------+     +-----------+     +-----+------+
                                             |
                                             | sub-ms reads
                                             v
                                       +-----+------+
                                       | Model      |
                                       | Serving    |
                                       | (features +|
                                       |  inference)|
                                       +------------+

  Feature Storage Schema:
  Key: feature:{entity_type}:{entity_id}:{feature_group}
  Value: Hash with feature names as fields

  Example:
  HSET feature:user:1001:profile
    age 28
    tenure_days 365
    lifetime_value 1500.00
    last_login_ts 1706140800
    preferred_category "electronics"

  HSET feature:user:1001:realtime
    session_duration_sec 340
    pages_viewed 12
    cart_value 89.99
    clicks_last_hour 45
```

### Feature Store Performance Requirements

```
  +----------------------------------------------------------+
  | Metric                | Requirement     | Redis Achieves  |
  |-----------------------|-----------------|-----------------|
  | Read latency (p50)    | < 1ms           | ~0.2ms          |
  | Read latency (p99)    | < 5ms           | ~1-2ms          |
  | Throughput             | 100K+ reads/sec | 200K+ per node  |
  | Feature freshness     | < 1 minute      | Real-time via   |
  |                       |                 | stream consumers|
  | Feature vector size   | 50-500 features | HGETALL: ~0.5ms |
  | Entity count          | 100M+ entities  | Cluster sharding|
  +----------------------------------------------------------+

  Feature Retrieval Pattern:
  1. Model serving receives inference request for user 1001
  2. Pipeline HGETALL for each feature group:
     PIPELINE:
       HGETALL feature:user:1001:profile
       HGETALL feature:user:1001:realtime
       HGETALL feature:user:1001:embeddings
     END PIPELINE
  3. Single RTT, all features retrieved: ~0.5-1ms
  4. Concatenate into feature vector for model input
```

### Feature Freshness: Batch vs Real-Time

```
  Batch Features (daily/hourly):
  +----------+     +--------+     +-------+
  | Data     |---->| Spark  |---->| Redis |
  | Warehouse|     | Job    |     | MSET  |
  +----------+     +--------+     +-------+
  Freshness: hours                 Bulk load: PIPELINE + MSET
                                   100M features: ~10 minutes

  Real-Time Features (seconds):
  +----------+     +---------+     +-------+
  | Event    |---->| Flink / |---->| Redis |
  | Stream   |     | Stream  |     | HSET  |
  | (Kafka)  |     | Processor|    +-------+
  +----------+     +---------+
  Freshness: seconds               Per-event: HSET + EXPIRE
```

---

## 4. Rate Limiting for AI Inference Endpoints

### Why Rate Limiting is Critical for LLM APIs

```
  LLM inference is expensive:
  - GPU-seconds per request: 1-30 seconds
  - Cost per request: $0.01-$0.10
  - GPU capacity is finite and pre-provisioned

  Without rate limiting:
  - One client sends 10K requests/sec
  - GPU queue saturates in seconds
  - All other clients experience timeout
  - Cost: $1,000/sec in compute

  Rate limiting dimensions for AI:
  1. Requests per second (API calls)
  2. Tokens per minute (input + output tokens)
  3. Concurrent requests (GPU slots)
  4. Cost per hour (dollar-based limiting)
```

### Token Bucket with Redis (Lua Script)

```lua
  -- Token bucket rate limiter
  -- KEYS[1] = rate limit key
  -- ARGV[1] = max tokens (bucket size)
  -- ARGV[2] = refill rate (tokens per second)
  -- ARGV[3] = tokens requested
  -- ARGV[4] = current timestamp (microseconds)

  local key = KEYS[1]
  local max_tokens = tonumber(ARGV[1])
  local refill_rate = tonumber(ARGV[2])
  local requested = tonumber(ARGV[3])
  local now = tonumber(ARGV[4])

  local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
  local tokens = tonumber(bucket[1]) or max_tokens
  local last_refill = tonumber(bucket[2]) or now

  -- Refill tokens based on elapsed time
  local elapsed = (now - last_refill) / 1000000  -- to seconds
  tokens = math.min(max_tokens, tokens + (elapsed * refill_rate))

  local allowed = 0
  local remaining = tokens

  if tokens >= requested then
      tokens = tokens - requested
      allowed = 1
      remaining = tokens
  end

  redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
  redis.call('EXPIRE', key, math.ceil(max_tokens / refill_rate) + 1)

  return {allowed, math.floor(remaining)}
```

### Multi-Tier Rate Limiting Architecture

```
  +------------------------------------------------------------------+
  | Rate Limiting Tiers for AI API                                    |
  |                                                                    |
  | Tier 1: Per-API-Key (requests/min)                                |
  |   Key: rl:req:{api_key}:{minute}                                  |
  |   Free: 60/min | Pro: 600/min | Enterprise: 6000/min             |
  |                                                                    |
  | Tier 2: Per-API-Key (tokens/min)                                  |
  |   Key: rl:tok:{api_key}:{minute}                                  |
  |   Free: 10K/min | Pro: 100K/min | Enterprise: 1M/min             |
  |                                                                    |
  | Tier 3: Global (concurrent GPU slots)                             |
  |   Key: rl:gpu:concurrent                                          |
  |   Semaphore pattern: max 1000 concurrent inferences               |
  |                                                                    |
  | Tier 4: Per-Model (protect expensive models)                      |
  |   Key: rl:model:{model_name}:{minute}                             |
  |   GPT-4: 100/min cluster-wide | GPT-3.5: 1000/min               |
  +------------------------------------------------------------------+

  Request Flow:
  1. Check Tier 1 (per-key request rate)     --> REJECT if exceeded
  2. Estimate token count from input          --> pre-check Tier 2
  3. Check Tier 3 (acquire GPU semaphore)    --> QUEUE if full
  4. Check Tier 4 (model-specific limit)     --> REJECT if exceeded
  5. Execute inference
  6. Post-inference: update Tier 2 with actual tokens used
  7. Release Tier 3 semaphore
```

### Sliding Window Rate Limiter

```
  More accurate than fixed-window counters:

  KEYS[1] = rl:sliding:{api_key}
  Window size: 60 seconds

  Implementation using Sorted Set:
  1. ZREMRANGEBYSCORE key 0 (now - 60s)    -- Remove old entries
  2. ZCARD key                              -- Count requests in window
  3. If count < limit:
       ZADD key now member_id              -- Add this request
       EXPIRE key 60                        -- Auto-cleanup
       return ALLOWED
     Else:
       return REJECTED

  Memory: ~100 bytes per request in window
  At 1000 req/sec * 60s window = 60K entries = ~6MB per key
  Optimization: Use PFADD (HyperLogLog) for approximate counting
```

---

## 5. Session Management for Chat-Based AI

### Chat Session Architecture

```
  +------------------------------------------------------------------+
  | Chat Session Storage in Redis                                     |
  |                                                                    |
  | Session Key: chat:{session_id}                                    |
  | Type: List (ordered messages) or Stream                           |
  |                                                                    |
  | Structure (Hash-based):                                           |
  | chat:session:abc123                                                |
  |   metadata -> {"user_id": "u1001", "model": "gpt-4",            |
  |                "created": 1706140800, "system_prompt": "..."}     |
  |                                                                    |
  | chat:messages:abc123 (Stream)                                     |
  |   1706140800-0: role=user content="Hello" tokens=5                |
  |   1706140801-0: role=assistant content="Hi there!" tokens=8       |
  |   1706140810-0: role=user content="Explain ML" tokens=10          |
  |   1706140812-0: role=assistant content="Machine learning..."      |
  |                 tokens=250 finish_reason=stop                     |
  +------------------------------------------------------------------+
```

### Context Window Management

```
  Problem: LLM context windows are finite (4K-128K tokens)
  Chat history grows unbounded

  Strategy: Sliding Window with Summarization

  +----------------------------------------------+
  | Full Chat History in Redis Stream             |
  | [msg1][msg2][msg3]...[msg50][msg51]...[msg100]|
  +----------------------------------------------+
        |                                |
        v                                v
  +----------------+           +------------------+
  | Summarized     |           | Recent messages  |
  | (msgs 1-50)   |           | (msgs 51-100)    |
  | ~200 tokens    |           | ~3000 tokens     |
  +----------------+           +------------------+
        |                                |
        +------------ + -----------------+
                      |
                      v
               +------+-------+
               | LLM Context  |
               | System prompt |
               | + Summary     |
               | + Recent msgs |
               | = ~3500 tokens|
               +--------------+

  Redis Operations:
  1. XADD chat:messages:{session} * role user content "..." tokens 10
  2. XLEN chat:messages:{session}  --> check if > threshold (50)
  3. If over threshold:
     a. XRANGE chat:messages:{session} - + COUNT 50  (get old msgs)
     b. Summarize via LLM
     c. HSET chat:session:{session} summary "..."
     d. XTRIM chat:messages:{session} MAXLEN ~ 50 (trim old)
```

### Session TTL and Cleanup

```
  Session Lifecycle:
  +----------+     +-----------+     +-----------+     +----------+
  | Created  |---->| Active    |---->| Idle      |---->| Expired  |
  | (TTL=24h)|     | (TTL reset|     | (TTL      |     | (auto-   |
  |          |     |  on msg)  |     |  counting)|     |  deleted) |
  +----------+     +-----------+     +-----------+     +----------+

  TTL Strategy:
  - Active sessions: EXPIRE reset on every message (24h)
  - Idle sessions: 1h TTL after last message
  - Archived sessions: Persist to database before TTL expiry

  Cleanup with Keyspace Notifications:
  CONFIG SET notify-keyspace-events Ex  (expire events)

  Subscriber listens on __keyevent@0__:expired
  On session expiry:
  1. Check if session should be archived (long conversations)
  2. Archive to database/S3 if needed
  3. Clean up related keys (messages, metadata)
```

---

## 6. Embedding Caching Strategies

### Embedding Cache Architecture

```
  Embedding generation is expensive:
  - OpenAI ada-002: ~$0.0001 per 1K tokens, 100ms latency
  - Local model: ~5-20ms per embedding, GPU utilization cost

  Cache Strategy:
  +------+     +----------+     +----------+     +-----------+
  | Text |---->| Hash     |---->| Redis    |---->| Return    |
  |      |     | (SHA256) |     | Lookup   |     | Cached    |
  +------+     +----------+     | (hit?)   |     | Embedding |
                                +----+-----+     +-----------+
                                     |
                                  MISS
                                     |
                                     v
                                +----+-----+
                                | Generate |
                                | Embedding|
                                | (API/GPU)|
                                +----+-----+
                                     |
                                     v
                                +----+-----+     +-----------+
                                | Store in |---->| Return    |
                                | Redis    |     | Embedding |
                                +----------+     +-----------+

  Key Design:
    emb:{model_name}:{sha256(text)[:16]}

  Value: Binary FLOAT32 array (1536 dims * 4 bytes = 6KB for ada-002)

  Memory Budget:
    1M cached embeddings * 6KB = 6GB
    + key overhead (~100 bytes each) = +100MB
    Total: ~6.1GB for 1M embeddings
```

### Embedding Versioning

```
  Model versions change embeddings:
  - ada-002 vs text-embedding-3-small produce different vectors
  - Fine-tuned models produce different embeddings
  - Cached embeddings from old model are INVALID with new model

  Versioning Strategy:
  Key: emb:v{version}:{model}:{hash}

  Example:
  emb:v2:ada002:a1b2c3d4e5f6  --> [0.12, 0.85, ...]

  On model upgrade:
  1. Deploy new model version
  2. New embeddings stored with new version prefix
  3. Old cache entries expire via TTL (7 days)
  4. No cache invalidation needed (namespace separation)

  Background migration:
  SCAN 0 MATCH emb:v1:* COUNT 1000
  For each key: re-embed with new model, store as v2
```

---

## 7. Redis Streams for ML Pipeline Events

### ML Pipeline Event Architecture

```
  +------------------------------------------------------------------+
  | Redis Streams as ML Pipeline Event Bus                            |
  |                                                                    |
  | Stream: ml:events:training                                        |
  | +-----------------------------------------------------------+    |
  | | ID           | event_type     | payload                   |    |
  | |--------------|----------------|---------------------------|    |
  | | 1706140800-0 | dataset_ready  | {"dataset": "v2.3", ...}  |    |
  | | 1706140801-0 | training_start | {"model": "rec-v4", ...}  |    |
  | | 1706140900-0 | epoch_complete | {"epoch": 5, "loss": 0.3} |    |
  | | 1706141800-0 | training_done  | {"model": "rec-v4", ...}  |    |
  | | 1706141801-0 | eval_start     | {"metrics": [...]}        |    |
  | | 1706141900-0 | model_approved | {"version": "v4.1"}       |    |
  | | 1706141901-0 | deploy_start   | {"target": "canary"}      |    |
  | +-----------------------------------------------------------+    |
  |                                                                    |
  | Consumer Groups:                                                   |
  |   ml:events:training -> group:evaluator (eval service)            |
  |   ml:events:training -> group:deployer  (deployment service)      |
  |   ml:events:training -> group:monitor   (experiment tracker)      |
  +------------------------------------------------------------------+
```

### Consumer Group Pattern for ML Pipelines

```
  # Create consumer groups
  XGROUP CREATE ml:events:training evaluator $ MKSTREAM
  XGROUP CREATE ml:events:training deployer $ MKSTREAM
  XGROUP CREATE ml:events:training monitor $ MKSTREAM

  # Evaluator service reads events
  XREADGROUP GROUP evaluator eval-worker-1
    COUNT 10 BLOCK 5000
    STREAMS ml:events:training >

  # Process event, then acknowledge
  XACK ml:events:training evaluator 1706141800-0

  # Pending entries (unacknowledged) -- for failure recovery
  XPENDING ml:events:training evaluator - + 10

  # Claim stale entries (worker died without ACK)
  XAUTOCLAIM ml:events:training evaluator eval-worker-2
    60000    -- claim entries idle > 60 seconds
    0-0      -- start from beginning of PEL
```

### Redis Streams vs Kafka for ML Events

| Aspect              | Redis Streams                    | Kafka                             |
|---------------------|----------------------------------|-----------------------------------|
| Latency             | Sub-millisecond                  | 1-10ms                            |
| Throughput           | ~100K-500K msgs/sec per stream  | 1M+ msgs/sec per partition        |
| Retention           | Memory-bound (MAXLEN/MINID)      | Disk-based (days/weeks/forever)   |
| Consumer groups     | Built-in, simple                 | Built-in, more features           |
| Ordering            | Per-stream guaranteed            | Per-partition guaranteed           |
| Replay              | Full replay from any ID          | Full replay from any offset       |
| Persistence         | AOF/RDB (same as Redis)          | Disk-first (durable by design)    |
| Best for            | Real-time events, low latency    | High-throughput, long retention   |
| ML pipeline use     | Feature updates, model events    | Training data, large event logs   |

**Recommendation**: Use Redis Streams for low-latency operational events (feature updates, model serving signals, real-time metrics). Use Kafka for high-throughput data pipelines (training data, batch feature computation).

---

## 8. Real-Time Model Serving Patterns

### Model Metadata and Routing in Redis

```
  # Model registry in Redis
  HSET model:registry:recommendation-v4
    status "active"
    endpoint "http://model-serving:8501/v1/models/rec-v4"
    version "4.1.0"
    canary_percent 10
    created_at 1706140800
    features_required "user_profile,realtime,item_embeddings"
    avg_latency_ms 45
    p99_latency_ms 120
    max_batch_size 32

  # A/B test routing
  HSET ab:experiment:rec-model-v4
    control_model "recommendation-v3"
    treatment_model "recommendation-v4"
    traffic_split 90:10
    start_time 1706140800
    end_time 1706745600

  # Canary routing with Redis
  Lua script:
  local experiment = redis.call('HGETALL', 'ab:experiment:rec-model-v4')
  local rand = math.random(100)
  if rand <= 10 then
      return experiment.treatment_model
  else
      return experiment.control_model
  end
```

### Feature Flag + Model Selection

```
  Request Flow:
  +--------+     +-------+     +---------+     +----------+
  | Client |---->| Redis |---->| Select  |---->| Model    |
  |        |     | (A/B  |     | Model   |     | Server   |
  |        |     |  test)|     | Version |     | Endpoint |
  +--------+     +-------+     +---------+     +----------+
                    |
                    | Also fetch from Redis:
                    | 1. Feature flags (model-specific)
                    | 2. Feature vectors (online features)
                    | 3. Cached embeddings (if available)
                    | 4. Rate limit check
                    v
              All in single pipeline (~1ms total)
```

### Real-Time Inference Result Caching

```
  Recommendation Caching Strategy:
  +----------------------------------------------------------+
  | Cache Level      | Key Pattern            | TTL    | Hit% |
  |------------------|------------------------|--------|------|
  | Exact user+ctx   | rec:{user}:{context}   | 5min   | 15%  |
  | User preferences | rec:{user}:prefs       | 1hour  | 40%  |
  | Popular items    | rec:popular:{category}  | 10min  | 70%  |
  | Precomputed recs | rec:precomp:{user}      | 1hour  | 50%  |
  +----------------------------------------------------------+

  Tiered Cache Strategy:
  1. Check exact cache (user + context hash)
  2. Check precomputed recommendations
  3. Check popular items (fallback)
  4. Call model serving (cache miss on all tiers)

  PIPELINE:
    GET rec:u1001:ctx_abc123           # Exact match
    GET rec:precomp:u1001              # Precomputed
    ZREVRANGE rec:popular:electronics 0 19  # Popular fallback
  END PIPELINE

  One RTT, three cache levels, ~0.5ms
```

---

## 9. Vector Similarity Search with Redis

### Redis as a Vector Database

```
  Redis Search Module (RediSearch) supports:
  - HNSW index: Approximate Nearest Neighbor (fast, ~95% recall)
  - FLAT index: Exact Nearest Neighbor (brute force, 100% recall)

  Performance Characteristics:
  +----------------------------------------------------------+
  | Vectors     | HNSW Search  | FLAT Search | Memory (1536d)|
  |-------------|------------- |-------------|---------------|
  | 10K         | 0.5ms        | 2ms         | 60MB          |
  | 100K        | 1ms          | 20ms        | 600MB         |
  | 1M          | 2-5ms        | 200ms       | 6GB           |
  | 10M         | 5-15ms       | 2000ms      | 60GB          |
  +----------------------------------------------------------+

  HNSW Parameters:
  - M (connections per layer): 16 (default) -- higher = better recall, more memory
  - EF_CONSTRUCTION: 200 (default) -- higher = better index quality, slower build
  - EF_RUNTIME: 10 (default) -- higher = better search recall, slower queries
```

### Hybrid Search: Vector + Metadata Filtering

```
  # Index supporting both vector and metadata filtering
  FT.CREATE product_idx ON HASH PREFIX 1 product:
    SCHEMA
      name TEXT
      category TAG
      price NUMERIC SORTABLE
      in_stock TAG
      embedding VECTOR HNSW 6
        TYPE FLOAT32
        DIM 384
        DISTANCE_METRIC COSINE

  # Hybrid query: "red shoes under $100" + vector similarity
  FT.SEARCH product_idx
    "(@category:{shoes} @price:[0 100])=>[KNN 10 @embedding $vec AS dist]"
    PARAMS 2 vec <query_embedding>
    RETURN 4 name category price dist
    SORTBY dist ASC
    DIALECT 2

  # This filters FIRST (category=shoes, price<100)
  # Then runs KNN on the filtered subset
  # Much faster than KNN on full dataset + post-filter
```

### Redis vs Dedicated Vector Databases

| Feature              | Redis (RediSearch)        | Pinecone              | Weaviate              |
|----------------------|---------------------------|-----------------------|-----------------------|
| Latency              | Sub-ms (in-memory)        | 10-50ms (managed)     | 5-20ms                |
| Max vectors          | ~50M per cluster          | Billions (managed)    | ~100M per node        |
| Hybrid filtering     | Full-text + vector + tag  | Metadata filtering    | GraphQL + vector      |
| Persistence          | RDB/AOF (standard Redis)  | Managed (always on)   | Disk-based            |
| Multi-tenancy        | Key prefixes + ACL        | Namespaces            | Multi-tenant classes  |
| Additional features  | Full Redis capabilities   | Vector-only           | Knowledge graph       |
| Cost at scale        | Self-managed: moderate    | High (managed pricing)| Moderate              |
| Best for             | < 50M vectors + need      | Large-scale pure      | Knowledge-rich        |
|                      | Redis features (cache,    | vector search         | applications          |
|                      | sessions, rate limiting)  |                       |                       |

---

## 10. Production Architecture: AI Platform on Redis

```
  Complete AI Platform Redis Architecture:
  +------------------------------------------------------------------+
  |                                                                    |
  |  Redis Cluster 1: Cache & Sessions (allkeys-lfu, 200GB)          |
  |  +------------------------------------------------------------+  |
  |  | Semantic cache | Session store | Response cache | Embeddings|  |
  |  +------------------------------------------------------------+  |
  |                                                                    |
  |  Redis Cluster 2: Feature Store (noeviction, 500GB)               |
  |  +------------------------------------------------------------+  |
  |  | User features | Item features | Real-time features | CTR   |  |
  |  +------------------------------------------------------------+  |
  |                                                                    |
  |  Redis Cluster 3: Vector Search (noeviction, 100GB)               |
  |  +------------------------------------------------------------+  |
  |  | Product vectors | User vectors | Content vectors | HNSW idx|  |
  |  +------------------------------------------------------------+  |
  |                                                                    |
  |  Redis Instance 4: Rate Limiting & Coordination (volatile-lru)    |
  |  +------------------------------------------------------------+  |
  |  | API rate limits | GPU semaphores | Model routing | A/B flags|  |
  |  +------------------------------------------------------------+  |
  |                                                                    |
  |  Redis Streams 5: Event Bus (maxlen-trimmed, 50GB)                |
  |  +------------------------------------------------------------+  |
  |  | ML pipeline events | Feature updates | Model deploy events  |  |
  |  +------------------------------------------------------------+  |
  |                                                                    |
  +------------------------------------------------------------------+
```

---

## Interview Narrative

**When asked "How would you use Redis in an AI/ML system?" or "Design the caching layer for an LLM-powered application":**

> "Redis serves multiple critical roles in AI/ML systems, and I would use separate Redis clusters for each concern to isolate failure domains and tune configurations independently.
>
> The highest-impact use is semantic caching for LLM responses. LLM inference is expensive, often $0.03-0.06 per request for GPT-4 class models. By embedding each prompt using a lightweight model and searching for similar prompts in Redis using the RediSearch vector index with HNSW, we can achieve 30-40% cache hit rates for repetitive queries. The key design decision is the similarity threshold: 0.95 for factual queries where precision matters, and 0.85-0.90 for creative tasks where approximate matches are acceptable. At 1M daily requests, this can save millions of dollars annually in inference costs.
>
> For the feature store serving layer, Redis is ideal because online model serving requires sub-millisecond feature retrieval. I would store features as Redis hashes with a key schema like feature:{entity}:{group}, enabling pipelined HGETALL across multiple feature groups in a single RTT. The feature store cluster uses noeviction policy because features must always be available for inference; a cache miss falling back to a database adds unacceptable latency.
>
> Rate limiting for AI APIs is multi-dimensional: we limit by requests per second, tokens per minute, concurrent GPU slots, and per-model capacity. I implement this as a Lua-based token bucket in Redis, checking all tiers in a single script execution to minimize round trips. The GPU semaphore pattern is particularly important because GPU slots are the most constrained resource.
>
> For chat-based AI applications, I use Redis Streams for conversation history. Streams provide natural ordering, consumer group support for async processing, and the ability to trim old messages. The context window management pattern summarizes older messages while keeping recent ones, all stored in Redis with sub-millisecond access.
>
> The architecture uses five separate Redis deployments: a cache cluster with LFU eviction for semantic cache and embeddings, a feature store cluster with no eviction, a vector search cluster with RediSearch indexes, a rate limiting instance with volatile eviction, and Redis Streams for ML pipeline events. This separation ensures that a cache eviction storm does not impact feature serving, and a vector search spike does not affect rate limiting."
