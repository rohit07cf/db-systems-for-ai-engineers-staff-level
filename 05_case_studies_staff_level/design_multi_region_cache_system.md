# Design a Multi-Region Caching Layer

## Staff/Principal-Level System Design Exercise

---

## 1. Problem Statement

Design a multi-region caching layer that serves 10M+ requests per second globally
across 5+ regions. The system must define a clear consistency model (not just "eventual"),
support multiple invalidation strategies (TTL, event-driven, version-based), prevent
thundering herd on cache misses, handle cold starts during region launches or failovers,
optimize cost across memory tiers, and maintain predictable failover behavior.

At staff level, this is not "put Redis in front of your database." The challenge is
reasoning about the consistency spectrum (how stale is acceptable for which data?),
the invalidation topology (who tells whom that data changed?), and the operational
reality that a misconfigured cache causes more outages than the database it protects.

---

## 2. Requirements

### 2.1 Functional Requirements

- **Multi-Region Deployment**: 5+ regions (US-East, US-West, EU-West, APAC-Tokyo,
  APAC-Singapore) with local caches in each.
- **Cache Topology**: Two-tier — local (per-region) and global (cross-region shared).
- **Consistency Model**: Configurable per cache namespace: strong, bounded staleness,
  or eventual.
- **Invalidation**: TTL-based, event-driven (pub/sub), and version-based (compare-and-swap).
- **Thundering Herd Prevention**: Single-flight for cache misses.
- **Cold Start Mitigation**: Pre-warming strategy for new regions or post-failover.
- **Negative Caching**: Cache "not found" results to prevent repeated DB lookups.

### 2.2 Non-Functional Requirements

- **Global Throughput**: 10M+ requests/sec across all regions.
- **Per-Region Throughput**: 2-3M requests/sec per region.
- **Latency**: Cache hit: p50 < 0.5ms, p99 < 2ms. Cache miss: < 50ms (DB fallback).
- **Hit Rate**: > 95% for hot data, > 80% overall.
- **Availability**: 99.99% — cache unavailability must not cascade to DB failure.
- **Memory Efficiency**: < 30% overhead vs raw data size.
- **Invalidation Latency**: Event-driven invalidation propagated across all regions
  within 2 seconds.

---

## 3. Scale Numbers

| Metric | Value |
|---|---|
| Global requests/sec | 10M+ |
| Regions | 5-7 |
| Per-region requests/sec | 2-3M |
| Unique cached keys (global) | ~5B |
| Hot keys (accessed in last hour) | ~200M |
| Avg value size | 500 bytes |
| Hot dataset size | ~100 GB per region |
| Total dataset size | ~2.5 TB (all unique keys) |
| Cache hit rate (target) | 95% (hot), 80% (overall) |
| Invalidation events/sec | ~500K |
| Cross-region replication bandwidth | ~250 MB/sec |
| DB backend capacity | ~500K reads/sec (total, all regions) |
| Write rate (cache updates) | ~1M/sec |

---

## 4. High-Level Architecture

```
                    ┌──────────────────────────────┐
                    │         DNS (GeoDNS)          │
                    └──────────────┬───────────────┘
                                   │
         ┌─────────────────────────┼────────────────────────┐
         │                         │                        │
  ┌──────▼──────┐          ┌──────▼──────┐          ┌──────▼──────┐
  │  US-East     │          │  EU-West     │          │  APAC-Tokyo  │
  │              │          │              │          │              │
  │ ┌──────────┐ │          │ ┌──────────┐ │          │ ┌──────────┐ │
  │ │ App Tier  │ │          │ │ App Tier  │ │          │ │ App Tier  │ │
  │ └────┬─────┘ │          │ └────┬─────┘ │          │ └────┬─────┘ │
  │      │       │          │      │       │          │      │       │
  │ ┌────▼─────┐ │          │ ┌────▼─────┐ │          │ ┌────▼─────┐ │
  │ │ L1 Cache │ │          │ │ L1 Cache │ │          │ │ L1 Cache │ │
  │ │(In-Proc) │ │          │ │(In-Proc) │ │          │ │(In-Proc) │ │
  │ └────┬─────┘ │          │ └────┬─────┘ │          │ └────┬─────┘ │
  │      │       │          │      │       │          │      │       │
  │ ┌────▼─────┐ │          │ ┌────▼─────┐ │          │ ┌────▼─────┐ │
  │ │ L2 Cache │ │          │ │ L2 Cache │ │          │ │ L2 Cache │ │
  │ │ (Redis   │ │          │ │ (Redis   │ │          │ │ (Redis   │ │
  │ │  Cluster)│ │          │ │  Cluster)│ │          │ │  Cluster)│ │
  │ └────┬─────┘ │          │ └────┬─────┘ │          │ └────┬─────┘ │
  │      │       │          │      │       │          │      │       │
  │ ┌────▼─────┐ │          │ ┌────▼─────┐ │          │ ┌────▼─────┐ │
  │ │ L3 Cache │ │          │ │ L3 Cache │ │          │ │ L3 Cache │ │
  │ │ (Global  │◄├──────────┤►│ (Global  │◄├──────────┤►│ (Global  │ │
  │ │  Memcache│ │  Inval.  │ │  Memcache│ │  Inval.  │ │  Memcache│ │
  │ │  /Redis) │ │  Bus     │ │  /Redis) │ │  Bus     │ │  /Redis) │ │
  │ └────┬─────┘ │          │ └────┬─────┘ │          │ └────┬─────┘ │
  │      │       │          │      │       │          │      │       │
  └──────┼──────┘          └──────┼──────┘          └──────┼──────┘
         │                        │                        │
         └────────────────────────┼────────────────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │    Source of Truth (DB)      │
                    │  (Spanner / Aurora Global /  │
                    │   DynamoDB Global Tables)    │
                    └─────────────────────────────┘
```

---

## 5. Storage Layer Design

### 5.1 Cache Topology: Three-Tier Design

**L1: In-Process Cache (Application Memory)**

```
Technology: Caffeine (Java) / Ristretto (Go) / lru-cache (Python)

Configuration:
  max_entries: 100K per instance
  max_memory: 512 MB per instance
  eviction: W-TinyLFU (Caffeine) — frequency + recency, best hit rate
  TTL: 10-60 seconds (short, to limit staleness)
  refresh_policy: Async refresh on read when TTL > 50% expired

Characteristics:
  - Latency: < 100 microseconds (memory read, no network)
  - Hit rate: 30-50% (small size, only hottest keys)
  - Invalidation: TTL only (too expensive to push invalidations to all instances)
  - Use case: Extremely hot keys (config, rate limit counters, session data)
```

**L2: Regional Cache (Redis Cluster)**

```
Technology: Redis Cluster (or KeyDB for multi-threaded performance)

Configuration per region:
  Nodes: 50-100 (r6g.2xlarge, 52 GB RAM each)
  Total memory: 2.6-5.2 TB per region
  Eviction: allkeys-lfu
  Persistence: None (cache only; data reconstructable from DB)
  Max connections: 50K per node

Characteristics:
  - Latency: p50 < 0.3ms, p99 < 1ms (same-AZ)
  - Hit rate: 80-90% (covers most of the hot working set)
  - Invalidation: Event-driven (pub/sub) + TTL
  - Use case: General-purpose caching for all read-heavy data
```

**L3: Global Cache (Cross-Region Shared Layer)**

```
Technology: Memcached (or Redis with cross-region replication)

Purpose:
  - Serves as "second chance" cache for regional cache misses
  - Stores data that is expensive to recompute from DB
  - Higher latency (cross-region) but higher hit rate than L2 alone

Configuration:
  - Deployed in 2-3 "hub" regions (US-East, EU-West, APAC-Tokyo)
  - Application routes to nearest hub on L2 miss
  - Cross-region latency: 50-150ms (acceptable for cache miss path)

Characteristics:
  - Latency: 20-80ms (cross-region network)
  - Hit rate: Additional 5-10% over L2 alone
  - Invalidation: Version-based (compare-and-swap)
  - Use case: Large objects, computed aggregates, expensive DB queries
```

**Hit rate cascade:**
```
Request → L1 (hit 40%) → L2 (hit 85% of remaining) → L3 (hit 60% of remaining) → DB
Overall: 40% + (60% * 85%) + (60% * 15% * 60%) = 40% + 51% + 5.4% = 96.4% cache hit rate
DB receives only 3.6% of requests: 10M * 3.6% = 360K/sec (within DB capacity of 500K/sec)
```

### 5.2 Consistency Model (Per-Namespace)

```
┌──────────────────────────────────────────────────────────────┐
│              Consistency Spectrum by Data Type                 │
│                                                               │
│  STRONG              BOUNDED STALENESS       EVENTUAL         │
│  ──────              ─────────────────       ────────         │
│  • User balance      • Product catalog       • User profile  │
│  • Inventory count   • Search results        • Recommendations│
│  • Auth tokens       • Leaderboards          • Analytics      │
│  • Rate limit state  • Pricing (< 5s stale)  • Content feed  │
│                                                               │
│  Read from primary   Read from cache if       Read from any   │
│  region, skip cache  age < staleness_bound    cache replica   │
│  (or read-through    Else: read-through       TTL-based       │
│  with lease)         to DB                    invalidation    │
│                                                               │
│  Latency: ~50ms      Latency: ~1ms (hit)     Latency: ~0.3ms │
│  (DB read)           ~50ms (miss)            (almost always   │
│                                               cache hit)      │
└──────────────────────────────────────────────────────────────┘
```

**Implementation: Cache-aside with staleness tags**

```python
class CacheEntry:
    value: bytes
    version: int              # Monotonic version from DB
    created_at: float         # Timestamp when cached
    source_region: str        # Which region populated this entry

class CacheClient:
    def get(self, key: str, consistency: str = "eventual",
            max_staleness_sec: float = None) -> Optional[bytes]:

        # L1 check (always, regardless of consistency)
        entry = self.l1.get(key)

        if consistency == "strong":
            # Skip all caches, go to DB
            return self.db.get(key)

        if consistency == "bounded_staleness":
            if entry and (now() - entry.created_at) < max_staleness_sec:
                return entry.value
            # Fall through to L2/L3/DB

        if consistency == "eventual":
            if entry:
                return entry.value
            # Fall through to L2/L3/DB

        # L2 check (regional Redis)
        entry = self.l2.get(key)
        if entry:
            self.l1.set(key, entry)  # Promote to L1
            if consistency == "bounded_staleness":
                if (now() - entry.created_at) < max_staleness_sec:
                    return entry.value
                # Stale; fall through to DB
            else:
                return entry.value

        # L3 check (global cache)
        entry = self.l3.get(key)
        if entry:
            self.l2.set(key, entry)  # Promote to L2
            self.l1.set(key, entry)  # Promote to L1
            return entry.value

        # Cache miss: read from DB
        value = self.db.get(key)
        entry = CacheEntry(value=value, version=self.db.version(key),
                          created_at=now(), source_region=self.region)
        self.l3.set(key, entry)
        self.l2.set(key, entry)
        self.l1.set(key, entry)
        return value
```

### 5.3 Invalidation Strategies

**Strategy 1: TTL-Based (Passive)**
```
SET key value EX {ttl_seconds}

Properties:
  - Simplest. No coordination needed.
  - Max staleness = TTL. Set TTL based on data change frequency.
  - No thundering herd protection (all replicas expire simultaneously).
  - Use for: Recommendations, analytics, content feeds.

TTL selection heuristic:
  - Data changes every X seconds → TTL = X * 2 (tolerate 2x staleness)
  - Diminishing returns: TTL of 5s vs 10s adds 2x invalidation load for 5s less staleness.
```

**Strategy 2: Event-Driven (Active)**
```
Write to DB → Publish event to Kafka → Consumers in each region invalidate local cache

Event schema:
  {
    "key": "user:12345:profile",
    "version": 43,
    "action": "invalidate",    // or "update" with new value
    "source_region": "us-east",
    "timestamp": 1706000000
  }

Consumer in each region:
  1. Receive event from Kafka
  2. DELETE key from L2 (Redis)
  3. Publish to local pub/sub for L1 invalidation (best-effort)

Properties:
  - Invalidation latency: 1-2 seconds cross-region (Kafka replication)
  - Guaranteed consistency (eventually): Every write triggers invalidation
  - Requires infrastructure: Kafka, consumers, monitoring
  - Use for: Product catalog, pricing, user settings
```

**Strategy 3: Version-Based (Compare-and-Swap)**
```
Every cache entry has a version number.
On read: Compare cached version with authoritative version.
If stale: Fetch new value.

Implementation:
  - DB stores: {key → value, version}
  - Cache stores: {key → value, version, ttl}
  - Version oracle: Lightweight service that returns latest version for a key
    (or batch of keys). Backed by a version table in Redis or Memcached.

  On read:
    cached = cache.get(key)
    latest_version = version_oracle.get_version(key)
    if cached.version == latest_version:
      return cached.value    // Still valid
    else:
      value = db.get(key)
      cache.set(key, value, version=latest_version)
      return value

Properties:
  - Strong consistency (no stale reads, if version oracle is up to date)
  - Two round-trips on cache hit (cache + version oracle)
  - Version oracle must be extremely fast (< 0.5ms) and highly available
  - Use for: User balance, inventory, auth tokens
```

### 5.4 Thundering Herd Prevention

```
Problem: Popular key expires. 10,000 concurrent requests see cache miss.
All 10,000 hit the DB simultaneously. DB overloads.

Solution 1: Single-Flight / Request Coalescing
──────────────────────────────────────────────

  In-memory lock per key (per application instance):
    lock = get_or_create_lock(key)
    if lock.try_acquire():
      # I am the "leader" — fetch from DB
      value = db.get(key)
      cache.set(key, value)
      lock.release_with_value(value)
    else:
      # Wait for leader to finish
      value = lock.wait_for_value(timeout=5s)

  With distributed lock (across instances):
    Use Redis SETNX for distributed single-flight:
      if SETNX lock:{key} {instance_id} EX 5:
        # I am the leader
        value = db.get(key)
        cache.set(key, value)
        DEL lock:{key}
      else:
        # Someone else is fetching; wait and retry cache
        sleep(50ms)
        return cache.get(key)  # Should be populated by leader

Solution 2: Stale-While-Revalidate
──────────────────────────────────

  Cache entries have two TTLs:
    - Soft TTL (e.g., 60s): After this, entry is "stale but usable"
    - Hard TTL (e.g., 300s): After this, entry is truly expired

  On read:
    entry = cache.get(key)
    if entry.age < soft_ttl:
      return entry.value                    // Fresh
    elif entry.age < hard_ttl:
      trigger_async_refresh(key)            // Refresh in background
      return entry.value                    // Return stale value (fast)
    else:
      return fetch_from_db_with_single_flight(key)  // Hard miss

  Properties:
    - Users almost never see a cache miss (stale data served during refresh)
    - Background refresh prevents thundering herd
    - Staleness bounded by hard_ttl in worst case

Solution 3: Probabilistic Early Expiration (XFetch)
────────────────────────────────────────────────────

  Instead of all keys expiring at exactly TTL:
    effective_ttl = ttl - (beta * log(random()) * recompute_time)

  Where:
    - beta = tuning parameter (typically 1.0)
    - random() = uniform [0, 1]
    - recompute_time = estimated time to fetch from DB

  Effect: Keys expire slightly BEFORE their TTL, with randomized jitter.
  At most 1-2 requests trigger a refresh, not thousands.
```

### 5.5 Cold Start Mitigation

```
Scenarios:
  1. New region launch: Zero cache data. All requests hit DB.
  2. Post-failover: Region comes back with empty cache.
  3. Cache node replacement: Portion of keyspace lost.

Strategy 1: Pre-Warming from Sibling Region
────────────────────────────────────────────

  Before routing traffic to new region:
    1. Identify hot keys from existing region (access log analysis, top-N by frequency)
    2. Bulk-copy hot keys from sibling region's L2 cache to new region's L2 cache
    3. Use Redis DUMP/RESTORE or custom bulk copy tool
    4. Target: Pre-warm 80% of hot dataset before traffic shift

  Time estimate:
    100 GB hot dataset, 1 Gbps cross-region bandwidth = ~800 seconds (~13 min)
    With 10 Gbps dedicated link: ~80 seconds

Strategy 2: Shadow Traffic Warming
──────────────────────────────────

  Before cutover:
    1. Mirror production traffic to new region (shadow mode)
    2. New region processes requests but responses are discarded
    3. Cache populates organically from real traffic patterns
    4. Monitor hit rate: cutover when > 80%

  Duration: Typically 30-60 minutes for 80% hit rate
  Advantage: Warms exactly the right keys (actual traffic pattern)

Strategy 3: Gradual Traffic Shift
─────────────────────────────────

  Do not shift 100% of traffic at once:
    t=0:    5% of traffic → new region (cache starts warming)
    t=5m:   20% of traffic (hit rate climbing)
    t=15m:  50% of traffic (hit rate > 70%)
    t=30m:  100% of traffic (hit rate > 85%)

  Monitor DB load during ramp-up. Pause if DB approaches capacity limit.
```

### 5.6 Cache Eviction and Memory Management

```
Eviction policy by tier:

L1 (In-Process):
  - W-TinyLFU: Combines LRU for recency and CountMinSketch for frequency
  - Admission filter: New entries must pass a frequency threshold to enter cache
  - This prevents "cache pollution" from one-time access patterns (scans)

L2 (Redis):
  - allkeys-lfu: Evict the least frequently used key globally
  - Why not LRU? At this scale, there are many keys accessed occasionally but
    repeatedly. LFU preserves these; LRU would evict them in favor of one-time
    burst keys.
  - Memory warning: Alert at 80% capacity. Begin more aggressive eviction at 90%.

L3 (Global):
  - allkeys-lru with large TTLs
  - This tier is for "expensive to recompute" data. LRU is appropriate because
    access patterns are less skewed (L1 and L2 already absorbed the hot keys).

Memory overhead breakdown:
  Raw value: 500 bytes avg
  Redis overhead per key: ~80 bytes (dict entry, key string, metadata)
  Total per key: ~580 bytes
  Overhead ratio: 16% (well within 30% target)

For 200M hot keys per region:
  200M * 580 bytes = ~116 GB → 3 nodes at 52 GB each (with headroom)
  Actual deployment: 50 nodes per region (for throughput, not capacity)
```

---

## 6. Detailed Component Design

### 6.1 Invalidation Bus (Cross-Region)

```
Architecture:
  ┌──────────┐    ┌─────────────┐    ┌──────────────────────┐
  │ DB Write │───▶│ CDC (Debezium│───▶│ Kafka (Global Topic) │
  └──────────┘    │ or native)  │    │ inval.cache.events   │
                  └─────────────┘    └──────────┬───────────┘
                                                │
                         ┌──────────────────────┼──────────────┐
                         │                      │              │
                  ┌──────▼──────┐  ┌────────────▼─┐  ┌────────▼───┐
                  │ US-East     │  │ EU-West      │  │ APAC-Tokyo │
                  │ Consumer    │  │ Consumer     │  │ Consumer   │
                  │             │  │              │  │            │
                  │ Invalidate  │  │ Invalidate   │  │ Invalidate │
                  │ L2 Redis    │  │ L2 Redis     │  │ L2 Redis   │
                  │ Notify L1   │  │ Notify L1    │  │ Notify L1  │
                  └─────────────┘  └──────────────┘  └────────────┘

Latency breakdown:
  DB write → CDC capture:     ~100ms
  CDC → Kafka produce:        ~50ms
  Kafka cross-region repl:    ~500-1500ms (depends on region pair)
  Consumer → Redis DELETE:    ~1ms
  ─────────────────────────────────────
  Total invalidation latency: ~1-2 seconds

For 500K invalidations/sec:
  Kafka throughput: 500K * 200 bytes (avg event) = 100 MB/sec
  Cross-region bandwidth: 100 MB/sec * 4 regions = 400 MB/sec
  This is within typical cross-region Kafka replication capacity.
```

### 6.2 Read Path (Detailed Flow)

```
1. Application receives request needing data for key K with consistency = "bounded_staleness(5s)"

2. L1 Check (in-process):
   entry = l1_cache.get(K)
   if entry and entry.age < 5s:
     METRICS: l1_hit++
     return entry.value

3. L2 Check (regional Redis):
   entry = redis.get(K)
   if entry:
     if entry.age < 5s:
       l1_cache.set(K, entry, ttl=min(5s, entry.remaining_ttl))
       METRICS: l2_hit++
       return entry.value
     else:
       METRICS: l2_stale++
       // Entry is stale but exists. Use stale-while-revalidate:
       trigger_async_refresh(K)
       return entry.value  // Return stale data now

4. L3 Check (global cache, optional):
   entry = global_cache.get(K)
   if entry:
     redis.set(K, entry, ttl=default_ttl)
     l1_cache.set(K, entry, ttl=5s)
     METRICS: l3_hit++
     return entry.value

5. DB Fallback (with single-flight):
   value = single_flight_fetch(K, lambda: db.get(K))
   entry = CacheEntry(value, version=db_version, created_at=now())
   global_cache.set(K, entry, ttl=long_ttl)
   redis.set(K, entry, ttl=default_ttl)
   l1_cache.set(K, entry, ttl=5s)
   METRICS: cache_miss++
   return value
```

### 6.3 Write Path (Cache Invalidation)

```
Two approaches — choose based on consistency requirement:

Approach A: Invalidate-on-Write (Default)
─────────────────────────────────────────
  1. Application writes to DB
  2. DB write triggers CDC event
  3. Invalidation bus deletes key from all L2 caches
  4. Next read sees cache miss → populates cache with fresh data

  Consistency gap: 1-2 seconds (time for invalidation to propagate)
  Pros: Simple. No stale data after invalidation propagates.
  Cons: Brief window of stale reads. Cache miss storm after invalidation.

Approach B: Write-Through (For hot keys)
────────────────────────────────────────
  1. Application writes to DB
  2. Application ALSO writes new value to local L2 cache immediately
  3. CDC event propagates update (not just invalidation) to remote regions
  4. Remote L2 caches updated with new value (no cache miss)

  Consistency gap: 1-2 seconds for remote regions, 0 for local region
  Pros: No cache miss after write. Better for frequently-updated hot keys.
  Cons: More complex. Risk of cache having newer data than DB on write failure.

Approach C: Write-Behind (For write-heavy data)
────────────────────────────────────────────────
  1. Application writes to cache ONLY
  2. Async background process flushes cache to DB
  3. Risk: Data loss if cache node fails before flush
  4. Only use for: Counters, view counts, analytics — data where loss is tolerable

  We generally AVOID write-behind for the caching layer. It belongs in
  specialized systems (e.g., rate limit counters).
```

---

## 7. Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| L2 Redis node failure | Partial cache misses in region | Redis Cluster auto-failover; L1 absorbs brief spike |
| Full L2 cluster failure | All cache misses in region; DB overloaded | Circuit breaker: serve stale L1 data; rate-limit DB calls; failover to L3 |
| Invalidation bus lag | Stale data served beyond SLA | Monitor lag metric; if lag > threshold, reduce L2 TTL dynamically |
| Invalidation bus failure | No invalidations; growing staleness | Fall back to TTL-only invalidation; alert ops team |
| DB overload from misses | Cascading failure | Single-flight prevents thundering herd; circuit breaker limits DB QPS |
| Cache poisoning (bad data) | Wrong values served | Version-based invalidation detects stale versions; manual purge API |
| Hot key (celebrity effect) | Single Redis node overloaded | Hot key detection + replication to multiple shards; L1 absorbs most reads |
| Memory pressure (L2) | Eviction of useful keys, hit rate drops | Proactive eviction monitoring; auto-scale Redis nodes; alert on hit rate drop |
| Network partition (region) | Region's cache diverges | Invalidation bus uses Kafka (durable, replayable); catch up after partition heals |
| Cold start (new node) | Burst of cache misses | Gradual traffic shift; pre-warming from peer nodes |

### Cascading Failure Protection

```
The most dangerous failure mode:
  Cache down → All traffic hits DB → DB overloaded → DB down → Total outage

Protection layers:

1. Circuit Breaker on DB:
   if db_error_rate > 50% over 10s:
     OPEN circuit → return cached stale data or default values
     Retry after 30s

2. Admission Control:
   if db_qps > 80% of capacity:
     Reject lowest-priority cache-miss requests
     Serve stale data instead of fresh

3. Graceful Degradation:
   if l2_cache unavailable:
     L1 TTL extended from 10s to 300s (serve increasingly stale data)
     Non-critical features disabled (no personalization, default recommendations)
     Critical paths (auth, checkout) bypass cache, direct to DB with rate limit

4. Pre-provisioned Headroom:
   DB capacity provisioned for 2x normal cache-miss rate
   Normal: 360K/sec to DB. Provisioned: 750K/sec.
   This absorbs cache degradation events without intervention.
```

---

## 8. Cost Analysis

| Component | Monthly Cost Estimate |
|---|---|
| L1 cache (application memory, included in compute) | $0 (marginal) |
| L2 Redis Cluster (5 regions, 50 nodes each, r6g.2xlarge) | $750K |
| L3 Global Cache (3 hubs, 20 nodes each) | $180K |
| Kafka (invalidation bus, 5 regions) | $100K |
| Cross-region data transfer (invalidation events) | $50K |
| Monitoring and observability | $30K |
| **Total** | **~$1.1M/month** |

**Cost per request**: $1.1M / (10M/sec * 2.6M sec/month) = $0.000042/request.

**Alternative: No cache (all DB reads)**:
- 10M/sec on a managed DB (e.g., DynamoDB on-demand): ~$13/sec = ~$34M/month.
- Cache saves: $34M - $1.1M = ~$33M/month. **30x cost reduction.**

**Cost optimization levers:**
1. Reduce L2 node count if hit rate > 97% (over-provisioned).
2. Use spot/preemptible instances for L3 (global cache is a "nice to have").
3. Compress values > 1 KB in Redis (ZSTD, ~3x compression). Reduces memory needs.
4. Use Redis on Graviton (ARM) instances: ~20% cheaper than x86 for same performance.

---

## 9. Evolution Path (Handling 10x Growth)

### 9.1 From 10M to 100M Requests/sec

- **L1 scales with app instances** (no additional cache infra needed).
- **L2**: 10x Redis nodes per region (500 nodes/region). At this point, consider
  KeyDB (multi-threaded Redis fork) to reduce node count by 5-8x.
- **Invalidation bus**: Kafka handles 5M events/sec easily. Cross-region bandwidth
  is the bottleneck: 1 GB/sec. May need dedicated network links.
- **DB backend**: Even at 96% hit rate, 4% of 100M = 4M/sec to DB. Need a DB that
  handles this (DynamoDB auto-scaling, or Vitess-sharded MySQL).

### 9.2 Adding New Regions (Latency Optimization)

- Current: 5 regions. Growth: 10-15 regions (add South America, Africa, more APAC).
- Each new region adds an L2 cluster and an invalidation bus consumer.
- Pre-warming strategy is critical: Use shadow traffic from GeoDNS before cutover.
- L3 hub topology may need adjustment (3 hubs serving 15 regions; some regions
  may be 200ms+ from nearest hub).

### 9.3 Intelligent Caching (ML-Driven)

- Use access pattern ML to predict which keys will be requested next.
- Pre-fetch predicted keys from DB before they are requested.
- Reduce cache miss rate from 4% to < 1%.
- This is the "10x improvement on top of 10x improvement" that staff engineers propose.

### 9.4 Tiered L2 (Memory + SSD)

- Not all cached data needs DRAM. Large values (> 5 KB) with moderate access
  frequency can live on NVMe SSD with acceptable latency (< 5ms).
- Redis on Flash (Enterprise feature) or custom SSD-backed cache.
- Cost impact: SSD is 10x cheaper than DRAM per GB. For 80% of data on SSD:
  L2 cost drops from $750K to ~$200K/month.

### 9.5 Edge Caching (CDN Integration)

- For public, cacheable data (product pages, API responses), push cache to CDN edge.
- 200+ edge locations vs 5 regions. Latency drops from 1ms (L2) to 0.1ms (edge).
- Challenge: Invalidation to 200+ edge nodes. Use CDN purge APIs with priority queues.
- This is the natural evolution when per-region caching is no longer sufficient
  for latency-sensitive use cases.

---

## 10. Interview Narrative

### How to Present This in 45 Minutes

**Minutes 0-5: Frame the problem.** "Multi-region caching is fundamentally about
three things: hit rate (determines DB load), staleness (determines data correctness),
and failure handling (determines whether a cache failure cascades into a total outage).
I will design for 10M req/sec across 5 regions with configurable consistency per
data type."

**Minutes 5-15: Topology and consistency.** Draw the three-tier architecture (L1/L2/L3).
Walk through the consistency spectrum: strong (skip cache), bounded staleness (check
age), eventual (always serve from cache). Show the hit rate cascade math: how L1 at
40% + L2 at 85% + L3 at 60% yields 96.4% overall, keeping DB load at 360K/sec.
This demonstrates quantitative reasoning.

**Minutes 15-25: Invalidation strategies.** Compare TTL, event-driven, and version-based
invalidation. For each, state the staleness bound, the infrastructure cost, and the
appropriate use case. The event-driven invalidation bus (DB -> CDC -> Kafka -> consumers)
is the centerpiece. Walk through the latency budget (1-2 seconds cross-region).

**Minutes 25-35: Thundering herd and cold start.** These are the operational challenges
that separate theoretical designs from production-ready ones. Walk through single-flight
with distributed locks, stale-while-revalidate, and probabilistic early expiration.
For cold start, explain shadow traffic warming and gradual traffic shift. These show
that you have operated cache infrastructure at scale.

**Minutes 35-45: Failure modes and cost.** The cascading failure scenario (cache down ->
DB overloaded -> total outage) is the most important failure mode. Walk through the
four protection layers: circuit breaker, admission control, graceful degradation,
and pre-provisioned headroom. End with the cost analysis: caching saves $33M/month
vs direct DB access. The ROI justifies the complexity.

### Key Phrases That Signal Staff-Level Thinking

- "Consistency is not binary. We offer three consistency levels per cache namespace
  because a user's account balance requires strong consistency while their
  recommendation feed tolerates minutes of staleness. One policy for all data is
  either too expensive or too stale."
- "The most dangerous failure is not a cache miss — it is a cache failure that
  cascades into a database failure. We provision the DB for 2x normal miss rate
  and use a circuit breaker to prevent unbounded DB load during cache degradation."
- "Stale-while-revalidate eliminates thundering herd for hot keys by serving the
  old value while refreshing in the background. Users experience zero-latency
  reads with bounded staleness rather than a latency spike on every TTL expiry."
- "At 10M req/sec, the cache saves $33M/month vs direct DB access. But the cache
  also introduces failure modes that did not exist before. The ROI is only real if
  the failure handling is robust enough to prevent cache-induced outages."
- "Cross-region invalidation via Kafka gives us durability and replayability. If a
  region's consumer falls behind, it catches up from Kafka's retention buffer. With
  pub/sub, those messages would be lost, and the region would serve stale data
  until TTL expiry."
