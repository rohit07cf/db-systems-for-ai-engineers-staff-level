# Redis Scaling Patterns — Staff/Principal Deep Dive

## 1. Vertical vs Horizontal Scaling

### Decision Matrix

| Dimension             | Vertical (Scale Up)              | Horizontal (Scale Out)                |
|-----------------------|----------------------------------|---------------------------------------|
| Implementation        | Bigger machine, more RAM         | More nodes, sharding                  |
| Max practical size    | ~500GB RAM per instance          | ~10TB+ across cluster                 |
| Throughput ceiling    | ~300K ops/sec (single thread)    | Linear scaling with masters           |
| Complexity            | Low (single instance)            | High (routing, migrations, resharding)|
| Failure blast radius  | Entire dataset unavailable       | Only affected slots unavailable       |
| Cost efficiency       | Poor at extremes (non-linear $)  | Good (commodity hardware)             |
| When to choose        | Dataset < 50GB, simple ops       | Dataset > 50GB, need > 300K ops/sec   |

### Vertical Scaling Limits

```
  Single Redis Instance Performance Profile:

  Operations/sec vs Memory Size:
  +----------------------------------------------------------+
  |  ops/sec                                                  |
  |  300K |==================                                 |
  |  250K |========================                           |
  |  200K |==============================                     |
  |  150K |====================================     <-- COW   |
  |  100K |=======================================   pressure |
  |   50K |===========================================        |
  |       +--+-----+-----+------+------+------+-----+        |
  |          8GB  16GB  32GB   64GB  128GB  256GB  512GB      |
  +----------------------------------------------------------+

  Throughput degrades at large sizes due to:
  1. BGSAVE fork() time: ~10ms per GB of data
  2. COW page table duplication: O(N) pages
  3. Active defragmentation overhead grows linearly
  4. Full sync to replicas takes longer (larger RDB)
```

---

## 2. Read Replicas

### Replication Architecture

```
  +-------------------+
  | Master (writes)   |
  | Slots: 0-16383    |
  +---+-------+-------+
      |       |
      | async | async
      | repl  | repl
      v       v
  +-------+ +-------+
  |Repl A | |Repl B |     Client reads with READONLY command
  |       | |       |     Stale reads possible (repl lag)
  +-------+ +-------+
      |
      | async repl (chained)
      v
  +-------+
  |Repl C |     Chained replication reduces master load
  +-------+     But increases replication lag
```

### Replication Backlog

```
  Master maintains a circular replication backlog:
  +-----------------------------------------------------------+
  | repl-backlog-size 256mb (default 1mb -- DANGEROUSLY small) |
  |                                                            |
  | [oldest offset .... master_repl_offset ... newest write]   |
  |  ^                   ^                                     |
  |  backlog_start       current position                      |
  +-----------------------------------------------------------+

  Partial resync succeeds IF:
    replica.repl_offset >= backlog_start
    AND replica connects to same master (matching replid)

  Partial resync fails (full sync triggered) IF:
    replica.repl_offset < backlog_start  (data evicted from backlog)
    OR master changed (failover occurred, replid mismatch)

  CRITICAL: Set repl-backlog-size to handle your max expected
  disconnect duration:
    size = write_throughput_bytes_per_sec * max_disconnect_seconds
    Example: 10MB/s writes * 60s disconnect = 600MB backlog needed
```

### READONLY Command and Stale Reads

```
  # Cluster mode: replicas reject reads by default
  > GET key1
  -MOVED 12345 192.168.1.1:6379

  # Enable reading from replica
  > READONLY
  +OK
  > GET key1
  "value"    <-- may be stale by replication lag (typically <1ms)

  Replication lag sources:
  +------------------------------------------+
  | Source              | Typical Impact      |
  |---------------------|---------------------|
  | Network RTT         | 0.1-1ms (same DC)  |
  | Output buffer queue | 0-10ms (write heavy)|
  | Replica I/O thread  | 0-1ms              |
  | Replica AOF fsync   | 0-2ms (if enabled) |
  | Slow command on repl| 0-100ms+ (KEYS, etc)|
  | Network congestion  | 1-100ms            |
  +------------------------------------------+
  Total typical: 0.5-5ms same DC, 10-100ms cross-DC
```

---

## 3. Sharding Strategies

### Client-Side Sharding

```
  Application Layer:
  +-------------------------------------------+
  |  hash(key) mod N --> select Redis instance |
  +---+--------+--------+--------+------------+
      |        |        |        |
      v        v        v        v
  +------+ +------+ +------+ +------+
  |Redis0| |Redis1| |Redis2| |Redis3|
  +------+ +------+ +------+ +------+

  Pros: Simple, no extra infra, full Redis feature set per shard
  Cons: Resharding requires application-level migration
        No automatic failover
        Client must know all nodes
```

### Consistent Hashing (Client-Side)

```
  Hash Ring:
            Node A (0-90)
           /            \
          /              \
    Node D               Node B
   (270-360)            (90-180)
          \              /
           \            /
            Node C (180-270)

  Adding Node E between A and B:
  - Only keys in A's range that now map to E need migration
  - ~1/N keys move (not N-1/N as in modulo hashing)

  Virtual nodes: Each physical node maps to 100-200 points
  on the ring for uniform distribution
```

### Redis Cluster vs Proxy-Based Sharding

| Aspect               | Redis Cluster            | Proxy (twemproxy/envoy)     | Client-Side              |
|-----------------------|--------------------------|-----------------------------|--------------------------|
| Routing intelligence  | Server-side + client     | Proxy                       | Client library           |
| Protocol              | RESP + redirections      | RESP (transparent)          | RESP (standard)          |
| Multi-key operations  | Same slot only           | Same backend only           | Same shard only          |
| Failover              | Automatic (built-in)     | External (sentinel/health)  | External                 |
| Connection pooling    | Client-side              | Proxy multiplexes           | Client-side              |
| Added latency         | Redirection: ~1 RTT      | Proxy hop: ~0.1ms           | None                     |
| Operational overhead  | Cluster management       | Proxy fleet management      | Application logic        |
| Resharding            | Online (MIGRATE)         | Requires custom solution    | Application handles      |

### Proxy-Based Sharding: twemproxy and envoy

```
  +----------+    +----------+    +----------+
  | Client 1 |    | Client 2 |    | Client 3 |
  +----+-----+    +----+-----+    +----+-----+
       |               |               |
       +-------+-------+-------+-------+
               |               |
         +-----v-----+  +-----v-----+
         | twemproxy  |  | twemproxy  |   Stateless proxies
         | (pool: 4   |  | (pool: 4   |   (horizontal scaling)
         |  backends) |  |  backends) |
         +--+--+--+---+  +--+--+--+---+
            |  |  |          |  |  |
       +----+  |  +----+     |  |  |
       v       v       v     v  v  v
   +------+ +------+ +------+ +------+
   |Redis0| |Redis1| |Redis2| |Redis3|
   +------+ +------+ +------+ +------+

  twemproxy characteristics:
  - Consistent hashing (ketama)
  - Connection pooling (multiplexes N client conns to M backend conns)
  - Pipelining support (batches commands to same backend)
  - NO support for: MULTI, WATCH, PUB/SUB, blocking commands
  - Single-threaded (scale by running multiple instances)

  Envoy Redis Proxy:
  - Same connection pooling + consistent hashing
  - Supports Redis Cluster protocol (follows MOVED/ASK)
  - mTLS between proxy and backends
  - Observability (Prometheus metrics, tracing)
  - Circuit breaking per backend
```

---

## 4. Connection Pooling Strategies

### Connection Lifecycle Cost

```
  New connection: TCP handshake (1.5 RTT) + AUTH + SELECT + CLIENT SETNAME
  Estimated: 0.5-2ms per new connection

  At 10K requests/sec with 1-connection-per-request:
    10K * 1ms = 10 seconds of pure connection overhead per second
    (impossible -- connections MUST be pooled)
```

### Pool Configuration Patterns

```
  Application Server Pool Configuration:
  +----------------------------------------------------------+
  |  Pool Setting          | Recommended        | Why         |
  |------------------------|--------------------|-------------|
  | min_idle               | 5-10               | Warm start  |
  | max_total              | 20-50              | Bound memory|
  | max_idle               | 10-20              | Reduce waste|
  | connection_timeout     | 250ms              | Fail fast   |
  | socket_timeout         | 100ms              | Bound tail  |
  | test_on_borrow         | false              | Performance |
  | test_while_idle        | true (30s interval)| Health check|
  | eviction_interval      | 30s                | Clean stale |
  +----------------------------------------------------------+

  Connection Pool Sizing Formula:
    pool_size = (peak_ops_per_sec * avg_command_latency_sec) / num_app_instances
    Example: (100K ops/s * 0.0005s) / 10 instances = 5 connections/instance

  In practice: 10-50 connections per application instance is typical
  Redis maxclients default: 10000
  Budget: (num_app_instances * pool_size) + replica_connections + monitoring < maxclients
```

---

## 5. Pipeline Optimization

### Pipeline vs Single Commands

```
  Without Pipeline (N commands):
  Client  ----CMD1----->  Server
  Client  <---RESP1-----  Server
  Client  ----CMD2----->  Server
  Client  <---RESP2-----  Server
  ...
  Total time: N * RTT + N * processing_time

  With Pipeline (N commands):
  Client  ----CMD1----->
  Client  ----CMD2----->
  Client  ----CMD3----->  Server processes all
  Client  <---RESP1-----
  Client  <---RESP2-----
  Client  <---RESP3-----
  Total time: 1 * RTT + N * processing_time

  Speedup for N=100 commands with 0.5ms RTT:
    Without: 100 * 0.5ms = 50ms
    With:    1 * 0.5ms = 0.5ms  (100x improvement)
```

### Pipeline Sizing Tradeoffs

| Pipeline Size | Client Memory | Server Memory         | Latency Impact         |
|---------------|---------------|-----------------------|------------------------|
| 1 (no pipe)   | Minimal       | Minimal               | RTT per command        |
| 10-50         | Low           | Low                   | Good balance           |
| 100-500       | Moderate      | Output buffer grows   | Optimal throughput     |
| 1000+         | High          | Risk output buffer OOM| Marginal improvement   |
| 10000+        | Very high     | Dangerous             | Diminishing returns    |

**Production guideline**: Pipeline 50-200 commands. Beyond that, the output buffer grows faster than the throughput improves. Monitor `client-output-buffer-limit`.

### Cluster-Aware Pipelining

```
  Smart client groups pipeline commands by destination node:

  Commands: SET a 1, SET b 2, SET c 3, GET a, GET d

  Slot mapping:
    a -> slot 15495 -> Node C
    b -> slot 3300  -> Node A
    c -> slot 7365  -> Node B
    d -> slot 11298 -> Node B

  Grouped pipelines:
    Node A: [SET b 2]                    (1 command)
    Node B: [SET c 3, GET d]             (2 commands, pipelined)
    Node C: [SET a 1, GET a]             (2 commands, pipelined)

  Execute all three pipelines in PARALLEL across nodes
  Total time: max(RTT_A, RTT_B, RTT_C) + max(processing_A, B, C)
```

---

## 6. Lua Scripting for Atomicity

### Why Lua in Redis

```
  Problem: Read-modify-write race condition

  Thread 1:              Thread 2:
    GET counter (=10)      GET counter (=10)
    INCR to 11             INCR to 11
    SET counter 11         SET counter 11
                           Lost update! Should be 12

  Solution: Lua script executes atomically (single-threaded)

  EVAL "
    local val = redis.call('GET', KEYS[1])
    val = tonumber(val) + 1
    redis.call('SET', KEYS[1], val)
    return val
  " 1 counter
```

### Lua Script Best Practices

```
  +------------------------------------------------------------+
  | Practice                    | Reason                        |
  |-----------------------------|-------------------------------|
  | Use EVALSHA, not EVAL       | Avoid resending script body   |
  | Keep scripts < 5ms          | Blocks ALL other commands     |
  | Declare all keys in KEYS[]  | Required for cluster routing  |
  | No external calls (io/os)   | Sandboxed environment         |
  | Use redis.call not pcall    | Propagate errors to client    |
  | Cache SCRIPT LOAD on startup| Avoid first-call overhead     |
  +------------------------------------------------------------+

  Script Caching Flow:
  1. SCRIPT LOAD "local val = ..." --> returns SHA1 hash
  2. EVALSHA <sha1> 1 mykey       --> execute by hash
  3. If NOSCRIPT error:           --> fallback to EVAL
```

### Lua vs MULTI/EXEC

| Feature            | Lua Scripts              | MULTI/EXEC Transactions      |
|--------------------|--------------------------|------------------------------|
| Conditional logic  | Full (if/else/loops)     | None (blind execution)       |
| Read-then-write    | Atomic                   | Requires WATCH (optimistic)  |
| Cross-key atomicity| Yes (same slot)          | Yes (same connection)        |
| Performance        | Single call, no RTT per op| Multiple RTTs for WATCH loop|
| Error handling     | In-script logic          | All-or-nothing execution     |
| Cluster support    | All keys must be same slot| All keys must be same slot  |
| Script replication | Script body or effects   | Individual commands          |

---

## 7. Memory Optimization Techniques

### Encoding Optimization

```
  Memory savings by choosing correct encoding:

  Hash with 5 fields (field names ~10 chars, values ~20 chars):
    hashtable encoding: ~400 bytes
    listpack encoding:  ~120 bytes (3.3x smaller)

  Trigger: hash-max-listpack-entries 128 (default)
           hash-max-listpack-value 64 (default)

  Strategy: Keep hashes under 128 entries and 64-byte values
            to stay in listpack encoding
```

### Memory Optimization Patterns

```
  Pattern 1: Hash Bucketing (Instagram technique)
  -----------------------------------------------
  Instead of: SET user:1001:name "alice"   (~80 bytes overhead per key)
              SET user:1001:email "a@b.c"

  Use:        HSET user:1 1001:name "alice"  (hash bucket, listpack encoded)
              HSET user:1 1001:email "a@b.c"

  Bucket ID: user_id / 1000 = 1
  Field: user_id % 1000 concatenated with field name

  Savings: 5-10x memory reduction for billions of small values

  Pattern 2: Bit-Level Storage
  -----------------------------------------------
  Tracking 100M users' daily login:
    SETBIT logins:2024-01-15 <user_id> 1

  Memory: 100M bits = 12.5MB (vs 100M keys = ~8GB)

  Pattern 3: Sorted Set Score Packing
  -----------------------------------------------
  Pack two 32-bit values into one 64-bit score:
    score = (timestamp << 32) | priority

  Enables compound sorting in a single ZRANGEBYSCORE

  Pattern 4: Short Key Names
  -----------------------------------------------
  Production key naming impact:
    "user:session:authentication:token:1001"  = 43 bytes
    "u:s:a:t:1001"                            = 14 bytes

  At 100M keys: saves ~2.7GB just on key names
  Tradeoff: Readability vs memory (use a key registry)
```

### Memory Analysis Commands

```
  # Per-key memory analysis
  MEMORY USAGE mykey [SAMPLES count]

  # Overall memory breakdown
  INFO memory
    used_memory: 1073741824        # Allocator-reported
    used_memory_rss: 1181116006    # OS-reported (RSS)
    mem_fragmentation_ratio: 1.10  # RSS/used (healthy)
    used_memory_dataset: 998244352 # Data only (excl overhead)
    used_memory_overhead: 75497472 # Internal structures

  # Object encoding inspection
  OBJECT ENCODING mykey    # "listpack", "hashtable", "skiplist", etc.
  OBJECT FREQ mykey        # LFU access frequency (0-255 log scale)
  OBJECT IDLETIME mykey    # LRU idle seconds

  # Big key scanning (non-blocking)
  redis-cli --bigkeys --memkeys -i 0.01   # 10ms sleep between scans
```

---

## 8. Comprehensive Scaling Decision Tree

```
  Start: Need more Redis capacity
       |
       +-- Is it a READ throughput bottleneck?
       |     |
       |     YES --> Add read replicas (simplest)
       |     |       + READONLY on cluster replicas
       |     |       + Client-side caching (Redis 6+ tracking)
       |     |
       |     NO
       |
       +-- Is it a WRITE throughput bottleneck?
       |     |
       |     YES --> Shard the data
       |     |       + Redis Cluster (native)
       |     |       + Proxy-based (twemproxy/envoy)
       |     |       + Pipeline optimization first (cheaper)
       |     |
       |     NO
       |
       +-- Is it a MEMORY capacity bottleneck?
       |     |
       |     YES --> First: optimize encodings + compression
       |     |       Then: shard the data (more masters = more RAM)
       |     |       Consider: Redis on Flash (tiered storage)
       |     |
       |     NO
       |
       +-- Is it a LATENCY bottleneck?
             |
             YES --> Check slow log (SLOWLOG GET)
                     + Avoid O(N) commands on large collections
                     + Pipeline for RTT reduction
                     + Connection pooling
                     + Move to same AZ as hot path
                     + Client-side caching for hot keys
```

---

## 9. Client-Side Caching (Redis 6+ Server-Assisted)

```
  Tracking Mode (Opt-in):
  +--------+                          +---------+
  | Client | ---CLIENT TRACKING ON--> | Redis   |
  |        |                          | Server  |
  | Local  | ---GET user:1001-------> |         |
  | Cache  | <--"alice"-------------- |         |
  | [user: |                          | Tracks: |
  |  1001  |                          | client X|
  |  =     |                          | watching|
  | "alice"]|                          | user:1001|
  +--------+                          +---------+
       |                                   |
       | (another client modifies user:1001)|
       |                                   |
       | <--INVALIDATE user:1001---------- |
       |                                   |
  Local cache evicts user:1001
  Next GET fetches from server

  Broadcast Mode (Opt-out):
  - Client subscribes to key prefixes: CLIENT TRACKING ON BCAST PREFIX user:
  - Gets invalidations for ALL keys matching prefix
  - Simpler but more invalidation traffic
```

---

## Interview Narrative

**When asked "How would you scale Redis for a service doing 1M ops/sec with 500GB of data?":**

> "I would approach this in layers, starting with the cheapest optimizations before adding infrastructure complexity.
>
> First, I would profile the workload. Using SLOWLOG and redis-cli --bigkeys, I'd identify if we have hot keys, large keys, or suboptimal encodings. Often, switching from individual keys to hash bucketing using the Instagram pattern can cut memory 5-10x, and pipeline optimization can boost throughput 10-100x by amortizing RTT.
>
> For the data size of 500GB, we clearly need sharding. I would use Redis Cluster with approximately 20 master nodes, each holding about 25GB. This leaves headroom for BGSAVE copy-on-write overhead without risking OOM. Each master gets one replica for HA, so 40 nodes total.
>
> For 1M ops/sec, assuming 70% reads and 30% writes, I would enable READONLY on replicas for the read path, which effectively doubles read capacity. The 20 masters can each handle about 25-30K writes/sec, which gives us 500-600K write ops/sec, plenty of headroom for the 300K write ops.
>
> On the client side, I would deploy a smart Redis client like lettuce or redis-py with cluster mode and pipeline support. The client groups commands by destination slot, pipelines them per node, and sends to all nodes in parallel. I would use connection pools sized to about 20-30 connections per application instance, and deploy the proxy tier only if we have too many small microservices creating connection pressure.
>
> For the data model, I would design hash tags carefully to co-locate data that needs multi-key atomicity, but I would validate the access distribution to avoid hot slots. I would use Lua scripts for any read-modify-write patterns to avoid WATCH/MULTI retry loops.
>
> The key tradeoff at this scale is operational complexity. Redis Cluster requires monitoring slot distribution, replication lag, memory fragmentation per node, and having runbooks for resharding when traffic patterns shift. I would instrument everything: per-node ops/sec, memory fragmentation ratio, replication offset lag, and client connection counts."
