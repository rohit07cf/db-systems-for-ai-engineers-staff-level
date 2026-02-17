# Redis Failure Modes — Staff/Principal Deep Dive

## 1. Failure Mode Taxonomy

```
  +------------------------------------------------------------------+
  |                    Redis Failure Modes                             |
  +------------------------------------------------------------------+
  |                                                                    |
  |  +------------------+  +------------------+  +------------------+  |
  |  | Data Loss        |  | Availability     |  | Performance      |  |
  |  | Failures         |  | Failures         |  | Degradation      |  |
  |  +------------------+  +------------------+  +------------------+  |
  |  | - Async repl gap |  | - Split-brain    |  | - Slow commands  |  |
  |  | - AOF corruption |  | - Failover storm |  | - Memory OOM     |  |
  |  | - RDB corruption |  | - Network part.  |  | - Fork latency   |  |
  |  | - OOM eviction   |  | - Thundering herd|  | - Replication lag|  |
  |  | - Partial failure |  | - Quorum loss    |  | - Fragmentation  |  |
  |  +------------------+  +------------------+  +------------------+  |
  +------------------------------------------------------------------+
```

---

## 2. Split-Brain in Redis Cluster

### The Scenario

```
  Normal State:
  +--------+     +--------+     +--------+
  | Master |     | Master |     | Master |
  |   A    |<--->|   B    |<--->|   C    |
  +---+----+     +---+----+     +---+----+
      |              |              |
      v              v              v
  +--------+     +--------+     +--------+
  |Replica |     |Replica |     |Replica |
  |  A1    |     |  B1    |     |  C1    |
  +--------+     +--------+     +--------+

  Network Partition:
  +-----------+    |PARTITION|    +-----------+
  | Master A  |    |         |    | Master B  |
  | Replica B1|    |   |||   |    | Master C  |
  |           |    |   |||   |    | Replica A1|
  +-----------+    |         |    +-----------+
   Minority        |         |     Majority
   side            |         |     side
```

### What Happens

```
  Timeline:
  t=0     : Partition occurs
  t=0-15s : Master A still serves writes (doesn't know it's partitioned)
  t=15s   : Majority side detects A as PFAIL
  t=16-30s: Majority promotes Replica A1 to Master A1
  t=30s   : A1 now serves slots 0-5460 with NEW data

  SPLIT-BRAIN WINDOW (t=0 to t=30s):
  - Master A accepts writes on minority side (stale master)
  - Master A1 accepts writes on majority side (new master)
  - Writes to BOTH are lost when partition heals

  t=60s   : Partition heals
  - Master A discovers A1 has higher configEpoch
  - Master A DEMOTES itself to replica of A1
  - ALL writes to Master A during the split are PERMANENTLY LOST
```

### Mitigation: cluster-node-timeout and min-replicas-to-write

```
  # Reduce split-brain window:
  cluster-node-timeout 5000          # 5s instead of 15s (faster detection)
                                     # Risk: more false-positive failovers

  # Prevent stale master from accepting writes:
  min-replicas-to-write 1            # Require >= 1 replica ACK
  min-replicas-max-lag 10            # Replica must be within 10s of lag

  Effect: Master A on minority side will REFUSE writes because
  it cannot reach any replica, returning:
  -NOREPLICAS Not enough good replicas to write.

  Tradeoff: This blocks writes during ANY replica failure,
  not just partitions. In a 3-master cluster with 1 replica each,
  a single replica failure makes its master read-only.
```

---

## 3. Data Loss During Failover (Async Replication)

### The Fundamental Problem

```
  Client         Master           Replica
    |               |                |
    |--SET key val->|                |
    |               |---(queued in   |
    |<--+OK---------|    repl buffer)|
    |               |                |
    |               |  [MASTER CRASHES before propagation]
    |               X                |
    |                                |
    |            [Replica promoted]  |
    |               |<-- new master  |
    |--GET key ---->|                |
    |<--nil---------|  DATA LOST     |
```

### Quantifying Data Loss Risk

```
  Data loss window = replication_lag + unacknowledged_buffer

  Typical values:
  +-----------------------------------------------+
  | Scenario                  | Data Loss Window   |
  |---------------------------|--------------------|
  | Same rack, low traffic    | < 1ms (~1 command) |
  | Same DC, moderate traffic | 1-10ms (~100 cmds) |
  | Same DC, heavy writes     | 10-100ms (~10K)    |
  | Cross-DC replication      | 1-50ms network     |
  | Replica under BGSAVE      | 100ms-1s           |
  | Replica disk I/O pressure | 1-10s              |
  +-----------------------------------------------+

  At 100K writes/sec with 10ms lag:
    ~1000 writes lost per failover event
```

### Partial Mitigation with WAIT

```
  SET critical:payment "..."
  WAIT 1 5000    # Wait for 1 replica, 5s timeout

  Returns: 1 (one replica acknowledged)

  LIMITATION: This does NOT guarantee durability.
  Scenario where WAIT still loses data:
  1. WAIT returns 1 (replica A1 acknowledged)
  2. Master A crashes
  3. Replica A1 crashes SIMULTANEOUSLY (e.g., rack power failure)
  4. Replica A2 (if exists) may not have the write
  5. Data is lost despite WAIT

  For true durability: Use WAIT + AOF with appendfsync always on replica
  Cost: ~3-5x latency increase (network RTT + fsync)
```

---

## 4. Memory OOM Failure

### OOM Kill Chain

```
  Memory Usage Timeline:
  |
  | maxmemory -------- |========================| eviction active
  |                     |########################| <-- eviction fails
  |                     |XXXXXXXXXXXXXXXXXXXXXXXX| (all keys have no TTL
  | RSS grows           |                        |  with volatile-lru)
  | from COW -------->  |                        |
  |                     |                        |
  | OOM Killer -------->X  [PROCESS KILLED]      |
  |
  +---------------------------------------------------> time
       BGSAVE fork    writes during      OOM
       starts         BGSAVE increase    kill
                      COW pages
```

### Recovery Playbook: OOM

```
  IMMEDIATE (during incident):
  1. Check if process is alive: redis-cli PING
  2. If alive but rejecting writes:
     a. CONFIG SET maxmemory-policy allkeys-lru  (if was noeviction)
     b. Identify and delete large keys:
        redis-cli --bigkeys (scan for top offenders)
     c. FLUSHDB ASYNC (nuclear option, non-blocking)
  3. If OOM-killed:
     a. Restart with increased maxmemory or reduced dataset
     b. Check vm.overcommit_memory = 1 (allows fork without guarantees)
     c. Review /var/log/kern.log for OOM killer output

  PREVENTION:
  +----------------------------------------------------------+
  | Setting                    | Recommended Value            |
  |----------------------------|------------------------------|
  | maxmemory                  | 75% of physical RAM          |
  | maxmemory-policy           | allkeys-lfu (not noeviction) |
  | vm.overcommit_memory       | 1 (sysctl)                   |
  | Monitoring alert threshold | 80% of maxmemory             |
  | Disable THP                | echo never > /sys/.../thp    |
  +----------------------------------------------------------+

  Memory Budget Formula:
    physical_RAM = maxmemory + BGSAVE_COW_headroom + OS_overhead
    Example: 64GB RAM = 48GB maxmemory + 12GB COW + 4GB OS
```

---

## 5. Replication Lag Divergence

### Cascading Lag Problem

```
  Master (100K writes/sec)
     |
     | repl_backlog: 256MB
     |
     v
  Replica A (processing 100K/sec, lag: 2ms)
     |
     | Replica A runs BGSAVE (fork takes 500ms)
     | During fork: replication paused
     | Lag grows: 2ms -> 500ms -> catches up over 2s
     |
     | If lag exceeds backlog capacity:
     | FULL RESYNC triggered (transfers entire dataset)
     |
     v
  Replica B (chained, lag: 10ms + A's lag)
     |
     | Chained replica amplifies lag from A
     | If A does full resync, B also needs full resync
```

### Full Resync Thunderstorm

```
  Failure cascade:
  1. Master under heavy write load
  2. Replica disconnects briefly (network blip, 5s)
  3. Reconnects, requests partial sync
  4. repl-backlog-size too small (default 1MB!)
  5. Partial sync fails -> FULL SYNC
  6. Master forks for RDB generation -> COW pressure
  7. Fork increases memory -> approaches maxmemory
  8. Other replicas lag increases due to master load
  9. Other replicas also trigger FULL SYNC
  10. Master overwhelmed by multiple RDB transfers
  11. Master becomes unresponsive -> failover triggered

  PREVENTION:
    repl-backlog-size = write_rate_bytes_sec * max_disconnect_seconds * 2
    Example: 10MB/s * 60s * 2 = 1.2GB (set to 2GB for safety)
```

---

## 6. Failover Storms

### What Triggers a Failover Storm

```
  Scenario: 50-node Redis Cluster, network becomes unstable

  t=0:   Network jitter causes 3 masters to miss heartbeats
  t=5s:  Nodes start marking each other as PFAIL
  t=10s: Multiple FAIL votes accumulate simultaneously
  t=15s: Three failovers trigger at once

  During each failover:
  - Epoch increments
  - Replicas request votes
  - Slot ownership transfers
  - Clients receive MOVED redirections
  - Client routing caches invalidated

  Cascading effect:
  - Failover elections compete for votes (epoch conflicts)
  - Network congestion from gossip storm (FAIL broadcasts)
  - Clients retry to wrong nodes (stale caches)
  - Application timeout errors spike
  - Load balancer marks backends unhealthy
  - Thundering herd on recovery
```

### Mitigation Strategies

```
  1. Tune cluster-node-timeout conservatively:
     cluster-node-timeout 15000   # Don't be too aggressive
     cluster-replica-validity-factor 10  # Prevent stale replicas
                                          from winning elections

  2. Implement failover backoff:
     cluster-allow-reads-when-down yes   # Serve stale reads vs total outage
     cluster-allow-pubsubshard-when-down yes

  3. Monitor and alert on:
     - cluster_state != ok
     - cluster_slots_fail > 0
     - Number of MOVED/ASK redirections per second
     - Epoch velocity (rapid epoch increments = instability)

  4. Circuit breaker in application layer:
     if (redis_error_rate > threshold):
         fallback_to_database()
         backoff_exponentially()
```

---

## 7. Client Reconnection Thundering Herd

### The Problem

```
  Redis restart or failover:
  +----------+
  | Redis    | <--- goes down
  +----------+

  1000 application threads detect connection failure simultaneously
  All 1000 threads try to reconnect at the same instant

  +-----+  +-----+  +-----+       +-----+
  |App 1|  |App 2|  |App 3| ..... |App N|
  +--+--+  +--+--+  +--+--+       +--+--+
     |        |        |              |
     +--------+--------+--------------+
                    |
              THUNDERING HERD
              (1000 simultaneous TCP SYNs)
                    |
                    v
  +----------+
  | Redis    | <--- overwhelmed on startup
  +----------+      - TCP backlog overflow
                    - AUTH command flood
                    - Slow to serve actual requests
```

### Recovery Playbook: Thundering Herd

```
  CLIENT-SIDE MITIGATIONS:

  1. Exponential backoff with jitter:
     reconnect_delay = min(base * 2^attempt + random(0, jitter), max_delay)
     Example: min(100ms * 2^attempt + random(0, 100ms), 30s)

  2. Connection pool warmup:
     - Don't create all connections at once
     - Stagger: open 1 connection, wait 100ms, open next
     - Lazy initialization: create on first use

  3. Circuit breaker pattern:
     CLOSED --[failures > threshold]--> OPEN
       ^                                  |
       |                           [timeout expires]
       |                                  |
       +-------[success]-------- HALF-OPEN

  SERVER-SIDE MITIGATIONS:

  1. TCP backlog tuning:
     tcp-backlog 511          # Redis config
     net.core.somaxconn 65535 # Kernel sysctl
     net.ipv4.tcp_max_syn_backlog 65535

  2. Protect with maxclients:
     maxclients 10000         # Hard limit on connections

  3. Gradual availability:
     - Redis Cluster: nodes join incrementally
     - Sentinel: stagger failover timing
```

---

## 8. Slow Command Impact

### Why Slow Commands Are Critical in Single-Threaded Redis

```
  Normal command flow:
  [cmd1: 0.1ms][cmd2: 0.1ms][cmd3: 0.1ms][cmd4: 0.1ms]...
  Total: 4 commands in 0.4ms, all clients happy

  One slow command (KEYS *, SORT, large ZRANGEBYSCORE):
  [cmd1: 0.1ms][KEYS *: 500ms!!!][cmd2: 0.1ms][cmd3: 0.1ms]...
                 ^
                 |
  EVERY other client is BLOCKED for 500ms
  All clients experience 500ms+ latency spike
  Connection pool exhaustion in applications
  Health checks fail -> cascading failures
```

### Dangerous Commands at Scale

| Command             | Time Complexity | Risk at 10M Keys      | Mitigation                    |
|---------------------|-----------------|------------------------|-------------------------------|
| KEYS *              | O(N)            | 500ms-5s block         | Use SCAN (cursor-based)       |
| FLUSHALL            | O(N)            | Seconds of blocking    | Use FLUSHALL ASYNC            |
| SORT on large sets  | O(N+M*log(M))  | 100ms-1s               | Pre-sort, limit results       |
| DEL large key       | O(M) elements   | 100ms+ for 1M entries  | Use UNLINK (async delete)     |
| HGETALL large hash  | O(N) fields     | 10-100ms               | Use HSCAN or specific HGET    |
| SMEMBERS large set  | O(N) members    | 10-100ms               | Use SSCAN                     |
| DEBUG SLEEP         | O(1)            | Exact sleep duration   | Disable in production ACL     |
| SAVE (foreground)   | O(N)            | Blocks until complete  | Always use BGSAVE             |

### Production Safeguards

```
  # Rename dangerous commands
  rename-command KEYS ""
  rename-command FLUSHALL ""
  rename-command FLUSHDB ""
  rename-command DEBUG ""

  # ACL-based restrictions (Redis 6+)
  ACL SETUSER app-user on >password ~app:* +@all -@dangerous -KEYS -DEBUG

  # Latency monitoring
  CONFIG SET latency-monitor-threshold 5   # Track > 5ms events
  LATENCY LATEST                           # Check recent spikes
  LATENCY HISTORY <event-name>             # Detailed history

  # Slow log
  CONFIG SET slowlog-log-slower-than 5000  # Log > 5ms (in usec)
  CONFIG SET slowlog-max-len 1000
  SLOWLOG GET 10                           # Review slow queries
```

---

## 9. Persistence Corruption

### AOF Corruption Scenarios

```
  Corruption causes:
  1. Disk full during AOF write (partial command written)
  2. Kernel crash during fsync (torn write)
  3. Hardware failure (bit flip on disk)
  4. Bug in AOF rewrite (rare but possible)

  Detection:
  $ redis-check-aof --fix appendonly.aof

  Output:
  0x    3a2f: Expected \r\n, got: 0x6b
  AOF analyzed: size=123456, ok_up_to=14895, diff=108561

  Fix options:
  1. --fix flag truncates AOF at last valid command
     Data loss: commands after corruption point
  2. Manual repair: hex-edit the file (expert only)
```

### RDB Corruption Scenarios

```
  $ redis-check-rdb dump.rdb

  Corruption causes:
  1. BGSAVE child killed (OOM) mid-write -> incomplete RDB
  2. Disk failure during atomic rename -> old RDB used (stale data)
  3. Network corruption during replica sync (RDB transfer)

  RDB checksum (CRC64):
  - Redis appends CRC64 checksum to RDB file
  - Verified on load; corrupted RDB rejected
  - rdbchecksum yes (default, don't disable)

  Recovery priority:
  1. Load from replica (if available and healthy)
  2. Use AOF if aof-use-rdb-preamble enabled
  3. Load last known good RDB backup
  4. Accept data loss and start fresh
```

---

## 10. Network Partition Behavior Matrix

```
  +-------------------------------------------------------------+
  | Partition Scenario        | Behavior           | Data Risk   |
  |---------------------------|--------------------|-------------|
  | Master isolated from      | Replica promoted   | Writes to   |
  | majority of cluster       | on majority side.  | old master  |
  |                           | Old master demoted | during split|
  |                           | on rejoin.         | are LOST.   |
  |---------------------------|--------------------|-------------|
  | Replica isolated from     | Replica reconnects | None        |
  | master                    | and partial/full   | (replica is |
  |                           | resync.            | read-only)  |
  |---------------------------|--------------------|-------------|
  | Client isolated from      | Client gets        | None (client|
  | its Redis node            | timeouts, failover | retries to  |
  |                           | to another node.   | other nodes)|
  |---------------------------|--------------------|-------------|
  | Cluster split 50/50       | NEITHER side has   | BOTH sides  |
  | (even partition)          | majority. No       | refuse      |
  |                           | failovers. Cluster | writes.     |
  |                           | marked as FAIL.    | Full outage.|
  |---------------------------|--------------------|-------------|
  | Inter-DC partition        | DC with majority   | Writes in   |
  | (2 DCs)                   | continues. Other   | minority DC |
  |                           | DC read-only or    | lost on     |
  |                           | down.              | rejoin.     |
  +-------------------------------------------------------------+
```

---

## 11. Comprehensive Recovery Playbook

### Incident Severity Classification

```
  SEV-1 (Critical): Complete data loss or cluster-wide outage
  SEV-2 (High):     Partial data loss or degraded writes
  SEV-3 (Medium):   Failover occurred, brief disruption
  SEV-4 (Low):      Performance degradation, no data impact
```

### SEV-1 Recovery: Cluster-Wide Outage

```
  1. ASSESS (first 5 minutes):
     - redis-cli -c CLUSTER INFO on each reachable node
     - Check cluster_state, cluster_slots_ok, cluster_known_nodes
     - Identify which nodes are up/down
     - Check system logs: dmesg, /var/log/syslog for OOM/hardware

  2. STABILIZE (5-15 minutes):
     - If OOM: increase maxmemory or add swap temporarily
     - If network: verify cluster bus ports (16379) reachable
     - Restart failed nodes one at a time:
       redis-server /etc/redis/redis.conf
     - Wait for each node to load data (RDB/AOF) before next

  3. RECOVER (15-60 minutes):
     - CLUSTER FAILOVER FORCE on healthy replicas if masters won't recover
     - CLUSTER SETSLOT <slot> NODE <id> to fix orphaned slots
     - redis-cli --cluster fix <any-node>:6379 (auto-repair tool)
     - Verify: CLUSTER SLOTS shows all 16384 slots covered

  4. VALIDATE:
     - Run application health checks
     - Compare key counts: DBSIZE on each node
     - Check replication: INFO replication on all nodes
     - Monitor for 30 minutes before declaring resolved
```

### SEV-2 Recovery: Data Loss After Failover

```
  1. QUANTIFY LOSS:
     - Compare master_repl_offset between old master and promoted replica
     - Lost commands = old_master_offset - replica_offset
     - Check AOF on old master (if recoverable from disk)

  2. PARTIAL RECOVERY:
     - If old master's AOF is intact:
       a. Start old master as standalone instance (different port)
       b. SCAN for keys modified in the lost window
       c. Migrate recovered keys back using DUMP + RESTORE
     - If AOF unavailable:
       a. Check application-level logs for recent writes
       b. Replay from upstream data source (database of record)

  3. PREVENT RECURRENCE:
     - Increase repl-backlog-size
     - Enable min-replicas-to-write
     - Add second replica per master
     - Deploy across 3 failure domains (racks/AZs)
```

---

## 12. Monitoring Checklist for Failure Prevention

```
  +-------------------------------------------------------------+
  | Metric                     | Alert Threshold   | Severity   |
  |----------------------------|-------------------|------------|
  | used_memory/maxmemory      | > 85%             | WARNING    |
  | used_memory/maxmemory      | > 95%             | CRITICAL   |
  | mem_fragmentation_ratio    | < 0.8 or > 2.0   | WARNING    |
  | rdb_last_bgsave_status     | != "ok"           | CRITICAL   |
  | aof_last_bgrewrite_status  | != "ok"           | WARNING    |
  | master_link_status         | != "up"           | CRITICAL   |
  | repl_backlog_active        | 0 (should be 1)   | CRITICAL   |
  | connected_clients          | > maxclients * 0.8| WARNING    |
  | cluster_state              | != "ok"           | CRITICAL   |
  | cluster_slots_fail         | > 0               | CRITICAL   |
  | instantaneous_ops_per_sec  | drop > 50%        | WARNING    |
  | latency_percentile p99     | > 5ms             | WARNING    |
  | rejected_connections       | > 0               | WARNING    |
  | keyspace_misses/hits ratio | > 0.5             | INFO       |
  +-------------------------------------------------------------+
```

---

## Interview Narrative

**When asked "What are the failure modes of Redis and how do you handle them?":**

> "Redis failure modes fall into three categories: data loss, availability loss, and performance degradation. Let me walk through the most impactful ones.
>
> The most fundamental data loss risk is async replication. When a master fails, any writes not yet propagated to the replica are permanently lost. The window is typically 1-10ms in the same datacenter, but can grow to seconds under heavy write load or when the replica is performing a BGSAVE. We mitigate this with the WAIT command for critical writes and by sizing the replication backlog large enough to prevent full resyncs that amplify the problem.
>
> For availability, split-brain is the most dangerous cluster failure mode. During a network partition, the minority-side master continues accepting writes that will be discarded when the partition heals. We use min-replicas-to-write to make the minority-side master refuse writes, accepting reduced availability for consistency. The tradeoff is that any single replica failure now makes its master read-only.
>
> The most common production incident is OOM. Redis running BGSAVE forks the process, and copy-on-write during heavy writes can temporarily double memory usage. The fix is setting maxmemory to 75% of physical RAM and using allkeys-lfu eviction. We also disable transparent huge pages, which causes catastrophic COW amplification: one modified byte copies a 2MB page instead of 4KB.
>
> For performance degradation, the single-threaded model means one slow command blocks everything. A single KEYS * on a 10-million-key instance blocks all clients for 500ms. We rename dangerous commands in production, enforce ACLs, and monitor the slow log with alerts on any command exceeding 5ms.
>
> Operationally, we maintain tiered recovery playbooks. For SEV-1 cluster outages, the priority is getting slots covered by forcing failovers and running cluster fix. For SEV-2 data loss, we quantify the gap using replication offsets and recover from AOF if possible, falling back to the upstream database of record. Prevention is about monitoring the right metrics: memory fragmentation ratio, replication lag, and cluster state, with alerts set well before critical thresholds."
