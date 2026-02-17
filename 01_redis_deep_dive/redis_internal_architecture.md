# Redis Internal Architecture — Staff/Principal Deep Dive

## 1. Single-Threaded Event Loop Model

Redis processes all commands on a **single main thread** using an event-driven, non-blocking I/O model built on top of `ae` (A simple event library). This is the most misunderstood aspect of Redis at senior levels.

### Why Single-Threaded Wins

| Factor                  | Single-Threaded Redis         | Hypothetical Multi-Threaded    |
|-------------------------|-------------------------------|--------------------------------|
| Context switch overhead | Zero                          | ~1-5 us per switch             |
| Lock contention         | None                          | Mutex on every data structure  |
| Cache line invalidation | Minimal (single core affine)  | Frequent cross-core flushes    |
| Command atomicity       | Free (sequential execution)   | Requires explicit locking      |
| Latency predictability  | p99 ~ p50                     | p99 >> p50 (lock waits)        |
| Throughput ceiling       | ~100K-300K ops/sec            | Higher but diminishing returns |

### Event Loop Architecture

```
  +---------------------------------------------------------+
  |                    ae Event Loop                         |
  |                                                         |
  |  +------------+    +-------------+    +---------------+ |
  |  | File Events|    | Time Events |    | Before Sleep  | |
  |  | (I/O ready)|    | (cron jobs) |    | (flush AOF)   | |
  |  +-----+------+    +------+------+    +-------+-------+ |
  |        |                  |                    |         |
  |        v                  v                    v         |
  |  +--------------------------------------------------+   |
  |  |              aeProcessEvents()                    |   |
  |  |  1. Calculate nearest timer deadline              |   |
  |  |  2. aeApiPoll(epoll/kqueue) with timeout          |   |
  |  |  3. Process fired file events (reads/writes)      |   |
  |  |  4. Process expired time events (serverCron)      |   |
  |  +--------------------------------------------------+   |
  +---------------------------------------------------------+
```

### I/O Threads (Redis 6.0+)

Redis 6.0 introduced `io-threads` for **read/write parallelism** while keeping command execution single-threaded.

```
  Client Connections
       |  |  |  |
       v  v  v  v
  +-------------------+
  | I/O Thread Pool   |    Phase 1: READ (parallel)
  | T1  T2  T3  T4   |    - Read socket data
  +--------+----------+    - Parse RESP protocol
           |
           v
  +-------------------+
  | Main Thread       |    Phase 2: EXECUTE (serial)
  | Command Execution |    - All commands run here
  +--------+----------+    - Single-threaded guarantees
           |
           v
  +-------------------+
  | I/O Thread Pool   |    Phase 3: WRITE (parallel)
  | T1  T2  T3  T4   |    - Serialize responses
  +-------------------+    - Write to sockets
```

**Key insight for interviews**: I/O threads do NOT execute commands. They only handle socket reads and response writes. The execution model remains single-threaded, preserving atomicity without locks.

**Configuration**: `io-threads 4` and `io-threads-do-reads yes` — typically set to number of cores minus 1, capped at 8.

---

## 2. Memory Allocator: jemalloc

Redis uses **jemalloc** (not glibc malloc) as its default allocator. This is critical for understanding Redis memory behavior.

### Why jemalloc Over glibc malloc

| Property            | jemalloc                     | glibc malloc (ptmalloc2)      |
|---------------------|------------------------------|-------------------------------|
| Fragmentation       | Low (~1.1x overhead)         | High (up to 2-4x)            |
| Thread scalability  | Per-thread arenas            | Global locks on arenas        |
| Small allocations   | Slab allocator (size classes) | Best-fit with splitting       |
| Memory return to OS | `MADV_DONTNEED` aggressive   | Rarely returns pages          |
| Redis memory ratio  | `used_memory / used_memory_rss` ~0.85-0.95 | Often ~0.5-0.7 |

### Memory Fragmentation Ratio

```
fragmentation_ratio = used_memory_rss / used_memory

  < 1.0  : Swapping (CRITICAL — immediate action needed)
  1.0-1.5: Healthy
  > 1.5  : Significant fragmentation (consider activedefrag)
  > 2.0  : Severe fragmentation (restart with RDB may be needed)
```

Redis 4.0+ includes **active defragmentation** (`activedefrag yes`) that runs incrementally during idle cycles, using jemalloc's `je_malloc_stats` to identify fragmented pages.

---

## 3. Internal Data Structures

### Simple Dynamic Strings (SDS)

Redis does NOT use C strings. SDS provides O(1) length, binary safety, and buffer pre-allocation.

```
  struct sdshdr {
      uint32_t len;      // used bytes (O(1) strlen)
      uint32_t alloc;    // allocated bytes (for pre-alloc)
      unsigned char flags; // type (sdshdr5/8/16/32/64)
      char buf[];        // actual data (binary safe, null terminated)
  }

  Memory layout:
  +------+-------+-------+------------------------+----+
  | len  | alloc | flags | buf (binary safe data) | \0 |
  +------+-------+-------+------------------------+----+
  ^                       ^
  header                  pointer returned to caller
```

**Space optimization**: Redis uses `sdshdr5`, `sdshdr8`, `sdshdr16`, `sdshdr32`, `sdshdr64` variants with progressively larger header fields to minimize overhead for small strings.

### Encoding Transitions

```
  String values:
    int (if fits in long) --> embstr (<=44 bytes) --> raw (>44 bytes)

  List:
    listpack (<=128 elements, <=64 bytes each) --> quicklist

  Set:
    intset (all integers, <=512 elements) --> hashtable
    listpack (<=128 elements, <=64 bytes each) --> hashtable

  Sorted Set:
    listpack (<=128 elements, <=64 bytes each) --> skiplist + hashtable

  Hash:
    listpack (<=128 entries, <=64 bytes each) --> hashtable
```

**Interview critical**: The thresholds above are configurable via `*-max-*-entries` and `*-max-*-value` configs. Tuning these affects memory by 2-10x for workloads with many small objects.

### Quicklist (List encoding since Redis 3.2)

```
  quicklist: doubly-linked list of ziplists (now listpacks)

  +--------+     +--------+     +--------+
  | node 1 |<--->| node 2 |<--->| node 3 |
  +--------+     +--------+     +--------+
  |listpack|     |listpack|     |listpack|
  |[a,b,c] |     |[d,e,f] |     |[g,h]   |
  +--------+     +--------+     +--------+

  - Each node stores a compressed listpack
  - list-max-listpack-size controls entries per node
  - list-compress-depth controls LZF compression of middle nodes
  - O(1) push/pop at both ends, O(N) for middle access
```

### Skip List (Sorted Set backbone)

```
  Level 4: HEAD ---------------------------------> 90 --> NIL
  Level 3: HEAD -----------> 30 ----------------> 90 --> NIL
  Level 2: HEAD --> 10 ----> 30 ------> 60 -----> 90 --> NIL
  Level 1: HEAD --> 10 -> 20 -> 30 -> 50 -> 60 -> 90 --> NIL

  - Probabilistic balancing (p=0.25 for level promotion)
  - Expected O(log N) search, insert, delete
  - Range queries via forward pointers at level 1
  - Each node stores: element, score, backward pointer, level array
  - Dual indexed: skiplist (by score) + hashtable (by element)
```

---

## 4. Persistence: RDB vs AOF

### Comparison Matrix

| Dimension           | RDB Snapshots                     | AOF (Append Only File)                |
|---------------------|-----------------------------------|---------------------------------------|
| Mechanism           | Point-in-time fork + serialize    | Log every write command               |
| Data loss window    | Up to `save` interval (minutes)   | `everysec`: ~1s, `always`: ~0        |
| Recovery speed      | Fast (binary load)                | Slow (replay all commands)            |
| File size           | Compact (compressed binary)       | Large (command log, even with rewrite)|
| fork() cost         | High (COW page table copy)        | Low (background rewrite only)         |
| I/O pattern         | Burst write during BGSAVE         | Continuous append (fsync policy)      |
| CPU impact          | Compression during save           | Rewrite compaction periodically       |
| Recommended         | Backups, disaster recovery        | Durability-critical workloads         |

### RDB Fork and Copy-on-Write

```
  BGSAVE triggered
       |
       v
  +----------+       fork()        +----------+
  | Parent   | ------------------> | Child     |
  | (serving)|                     | (saving)  |
  +----+-----+                     +-----+----+
       |                                 |
       | Writes modify pages             | Reads original pages
       | (COW triggers page copy)        | Serializes to dump.rdb
       |                                 |
       v                                 v
  [Modified pages]               [Original frozen snapshot]
       |                                 |
       |                                 +---> dump.rdb written
       |                                       (atomic rename)
       v
  Continues serving with
  increased RSS (COW overhead)
```

**Critical production concern**: A 64GB Redis with 50% write rate during BGSAVE can temporarily use ~96GB RSS (original 64GB + ~32GB COW pages). This is the #1 cause of OOM kills in production Redis.

### AOF Rewrite and Mixed Persistence (Redis 7+)

```
  aof-use-rdb-preamble yes (default since Redis 7)

  AOF File Structure:
  +----------------------------------+
  | RDB binary preamble (compact     |
  | snapshot of data at rewrite time)|
  +----------------------------------+
  | AOF commands appended after      |
  | rewrite (incremental changes)    |
  +----------------------------------+

  Recovery: Load RDB section (fast) -> Replay AOF tail (minimal)
```

---

## 5. Internal Command Flow

```
  Client sends: SET user:1001 '{"name":"alice"}' EX 3600

  +------------------------------------------------------------------+
  | 1. SOCKET READ (I/O thread or main thread)                       |
  |    - epoll_wait fires readable event                             |
  |    - Read bytes into client query buffer (querybuf)              |
  +----------------------------+-------------------------------------+
                               |
  +----------------------------v-------------------------------------+
  | 2. PROTOCOL PARSING                                              |
  |    - RESP3 protocol: *5\r\n$3\r\nSET\r\n$9\r\nuser:1001\r\n...  |
  |    - Parse into argc/argv: ["SET","user:1001","{...}","EX","3600"]|
  +----------------------------+-------------------------------------+
                               |
  +----------------------------v-------------------------------------+
  | 3. COMMAND LOOKUP                                                |
  |    - Hash lookup in server.commands dict                         |
  |    - Find setCommand with flags: write, denyoom, key@1           |
  +----------------------------+-------------------------------------+
                               |
  +----------------------------v-------------------------------------+
  | 4. PRE-EXECUTION CHECKS                                         |
  |    - ACL check (Redis 6+)                                       |
  |    - OOM check (maxmemory)   --> may trigger eviction            |
  |    - Command flag validation (readonly replica? blocked client?) |
  |    - Key slot check (cluster mode)                               |
  +----------------------------+-------------------------------------+
                               |
  +----------------------------v-------------------------------------+
  | 5. COMMAND EXECUTION                                             |
  |    - setCommand() -> setGenericCommand()                         |
  |    - lookupKeyWrite() on db->dict                                |
  |    - dbAdd() or dbOverwrite()                                    |
  |    - setExpire() in db->expires dict                             |
  +----------------------------+-------------------------------------+
                               |
  +----------------------------v-------------------------------------+
  | 6. POST-EXECUTION                                                |
  |    - Propagate to AOF buffer (if AOF enabled)                    |
  |    - Propagate to replicas (replication backlog)                 |
  |    - Key-space notifications (if enabled)                        |
  |    - Update server statistics (ops/sec, memory)                  |
  |    - Write +OK response to client output buffer                  |
  +------------------------------------------------------------------+
```

---

## 6. Memory Management and Eviction

### Eviction Policies

| Policy              | Scope     | Algorithm       | Use Case                          |
|---------------------|-----------|-----------------|-----------------------------------|
| noeviction          | N/A       | Return OOM error| When data loss is unacceptable    |
| allkeys-lru         | All keys  | Approx LRU      | General cache                     |
| volatile-lru        | TTL keys  | Approx LRU      | Cache with persistent keys mixed  |
| allkeys-lfu         | All keys  | Approx LFU      | Frequency-skewed workloads        |
| volatile-lfu        | TTL keys  | Approx LFU      | Hot/cold with TTL separation      |
| allkeys-random      | All keys  | Random           | Uniform access patterns           |
| volatile-random     | TTL keys  | Random           | Simple TTL-based cache            |
| volatile-ttl        | TTL keys  | Nearest TTL      | Time-sensitive data               |

### Approximate LRU/LFU Implementation

Redis does NOT maintain a true LRU linked list (too expensive: 16 bytes/key for prev/next pointers). Instead:

```
  Each key object stores a 24-bit LRU clock (or 8-bit log frequency + 16-bit decay time for LFU)

  Eviction sample:
  1. Randomly sample `maxmemory-samples` keys (default 5)
  2. Among sampled keys, evict the one with oldest LRU clock (or lowest LFU counter)
  3. Repeat until under maxmemory

  Accuracy vs Cost:
    samples=5  : ~91% accuracy vs true LRU
    samples=10 : ~98% accuracy vs true LRU
    samples=20 : ~99.5% accuracy (diminishing returns)
```

### How Redis Achieves Sub-millisecond Latency

```
  Latency Budget Breakdown (SET command, ~0.1ms total):
  +------------------------------------------------------+
  | Component             | Time       | Notes            |
  |---------------------- |------------|------------------|
  | epoll_wait wakeup     | ~5 us      | Kernel overhead  |
  | RESP parse            | ~2 us      | In-memory parse  |
  | Command lookup        | ~0.5 us    | Hash table O(1)  |
  | Dict lookup/insert    | ~1-3 us    | Hash table O(1)  |
  | AOF buffer append     | ~1 us      | Memory append    |
  | Replication propagate | ~1 us      | Backlog write    |
  | Response write        | ~2-5 us    | Socket buffer    |
  |                       |            |                  |
  | TOTAL                 | ~12-17 us  | Well under 1ms   |
  +------------------------------------------------------+
```

**Why it stays fast**:
1. All data in memory -- no disk I/O in the command path
2. Single-threaded -- no lock acquisition/release overhead
3. Efficient data structures -- O(1) hash lookups, O(log N) sorted ops
4. epoll/kqueue -- O(1) event notification (not O(N) select/poll)
5. Pre-allocated output buffers -- avoid malloc in hot path
6. Pipeline support -- amortize syscall overhead across multiple commands

---

## 7. Key Configuration Knobs for Production

```
  # Memory
  maxmemory 48gb                    # Leave headroom for COW during BGSAVE
  maxmemory-policy allkeys-lfu      # Best for most cache workloads
  maxmemory-samples 10              # Better eviction accuracy

  # Persistence
  save ""                           # Disable RDB if using AOF
  appendonly yes
  appendfsync everysec              # Balance durability/performance
  aof-use-rdb-preamble yes          # Fast recovery + durability

  # Performance
  io-threads 4                      # Match available cores
  io-threads-do-reads yes
  hz 10                             # Server cron frequency (10-100)
  dynamic-hz yes                    # Adaptive cron frequency

  # Danger zone
  latency-monitor-threshold 5       # Track commands > 5ms
  slowlog-log-slower-than 5000      # Log commands > 5ms (in microseconds)
```

---

## Interview Narrative

**When asked "Walk me through how Redis achieves sub-millisecond latency" or "Explain Redis's architecture":**

> "Redis achieves sub-millisecond latency through a deliberate set of architectural choices that trade off multi-core utilization for latency predictability. At its core, Redis uses a single-threaded event loop based on epoll or kqueue, which eliminates all lock contention and context-switch overhead. Every command executes atomically without synchronization primitives.
>
> The data structures are purpose-built for speed: hash tables with incremental rehashing to avoid latency spikes, skip lists for sorted sets that give O(log N) with excellent cache locality, and encoding transitions that automatically switch from compact representations like listpacks to full data structures as elements grow.
>
> Memory management uses jemalloc with size-class allocation, keeping fragmentation under 1.1x. All data lives in memory, so there is zero disk I/O in the command path. Persistence happens asynchronously via background fork for RDB or buffered append for AOF.
>
> Since Redis 6.0, I/O threads parallelize socket reads and writes while keeping command execution strictly single-threaded. This gives roughly 2x throughput improvement for I/O-bound workloads without sacrificing the single-threaded execution guarantee.
>
> In production at scale, the main tuning levers are maxmemory-samples for eviction accuracy, the persistence strategy choice between RDB and AOF with RDB preamble, and monitoring the memory fragmentation ratio to catch degradation before it impacts latency.
>
> The tradeoff is clear: Redis cannot saturate a 64-core machine with a single instance. The scaling answer is sharding via Redis Cluster with 16384 hash slots, which I can elaborate on if relevant."
