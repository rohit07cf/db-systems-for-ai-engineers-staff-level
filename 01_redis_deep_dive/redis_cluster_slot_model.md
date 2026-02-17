# Redis Cluster Slot Model — Staff/Principal Deep Dive

## 1. The 16384 Hash Slot Design

Redis Cluster partitions the keyspace into **16384 hash slots** (0-16383). Every key maps to exactly one slot, and every slot is owned by exactly one master node.

### Why 16384 Slots?

This number is a deliberate engineering tradeoff, not arbitrary:

| Factor                   | Analysis                                                   |
|--------------------------|------------------------------------------------------------|
| Gossip message size      | Bitmap of 16384 slots = 2KB per heartbeat (acceptable)     |
| 65536 slots alternative  | Bitmap = 8KB per heartbeat (excessive for 100+ node clusters)|
| Minimum useful cluster   | 3 masters x ~5461 slots each                               |
| Maximum practical cluster| ~1000 nodes (16 slots/node minimum for balanced distribution)|
| Migration granularity    | Fine enough for incremental rebalancing                    |
| Memory overhead          | Slot-to-node mapping table fits in L1 cache                |

**Interview insight**: Antirez (Salvatore Sanfilippo) explicitly chose 16384 because the gossip protocol sends the full slot bitmap in every ping/pong message. At 16384 bits (2KB), this is small enough for clusters of 100-200 nodes exchanging heartbeats every second.

### Slot Assignment Formula

```
  HASH_SLOT = CRC16(key) mod 16384

  CRC16 uses the CCITT polynomial (x^16 + x^12 + x^5 + 1)
  - Not CRC16-ANSI or CRC16-USB
  - Produces uniform distribution across 0-65535
  - mod 16384 maps to slot space
```

### Hash Tags for Co-location

```
  Key: {user:1001}.profile   --> CRC16("user:1001") mod 16384
  Key: {user:1001}.sessions  --> CRC16("user:1001") mod 16384
  Key: {user:1001}.cart      --> CRC16("user:1001") mod 16384

  Rule: If key contains {...}, only the substring inside the FIRST
        occurrence of { and } is hashed.

  All three keys land on the SAME slot, enabling:
  - MULTI/EXEC transactions across these keys
  - Lua scripts accessing all three atomically
  - MGET/MSET across co-located keys
```

**Critical caveat**: Over-using a single hash tag creates a hot slot. At FAANG scale, a viral user whose hash tag receives 100K ops/sec will bottleneck the single node owning that slot. Design hash tags around access-balanced entities, not around high-cardinality user IDs.

---

## 2. Cluster Topology

### Node Architecture

```
  +------------------------------------------------------------------+
  |                     Redis Cluster (6 nodes)                       |
  |                                                                   |
  |  +------------------+  +------------------+  +------------------+ |
  |  | Master A         |  | Master B         |  | Master C         | |
  |  | Slots: 0-5460    |  | Slots: 5461-10922|  | Slots: 10923-16383||
  |  | Port: 6379       |  | Port: 6379       |  | Port: 6379       | |
  |  | Bus:  16379      |  | Bus:  16379      |  | Bus:  16379      | |
  |  +--------+---------+  +--------+---------+  +--------+---------+ |
  |           |                      |                      |         |
  |           | async repl           | async repl           | async   |
  |           v                      v                      v         |
  |  +------------------+  +------------------+  +------------------+ |
  |  | Replica A1       |  | Replica B1       |  | Replica C1       | |
  |  | Slots: 0-5460    |  | Slots: 5461-10922|  | Slots: 10923-16383||
  |  | (read replicas)  |  | (read replicas)  |  | (read replicas)  | |
  |  +------------------+  +------------------+  +------------------+ |
  +------------------------------------------------------------------+
```

### Cluster Bus (Port + 10000)

Every node listens on two ports:
- **Client port** (6379): RESP protocol for client commands
- **Cluster bus port** (16379): Binary protocol for node-to-node communication

```
  Cluster Bus Message Types:
  +------------------+----------------------------------------+
  | Message          | Purpose                                |
  +------------------+----------------------------------------+
  | PING             | Heartbeat + state propagation          |
  | PONG             | Reply to PING (carries same payload)   |
  | MEET             | Join a new node to the cluster         |
  | FAIL             | Broadcast confirmed node failure       |
  | PUBLISH          | Cluster-wide pub/sub forwarding        |
  | FAILOVER_AUTH_REQ| Replica requests promotion vote        |
  | FAILOVER_AUTH_ACK| Master grants promotion vote           |
  | UPDATE           | Slot config update (epoch-based)       |
  +------------------+----------------------------------------+
```

---

## 3. MOVED and ASK Redirections

### MOVED Redirection (Permanent)

```
  Client --> Node A: GET user:5000
  Node A: key "user:5000" hashes to slot 7234
          slot 7234 is owned by Node B

  Node A --> Client: -MOVED 7234 192.168.1.2:6379

  Client updates local slot map:
    slot_cache[7234] = "192.168.1.2:6379"

  Client --> Node B: GET user:5000
  Node B --> Client: "alice"
```

**MOVED means**: "I don't own this slot. The correct owner is X. Update your routing table."

### ASK Redirection (Temporary, During Migration)

```
  Migration in progress: Slot 7234 moving from Node A to Node B

  Client --> Node A: GET user:5000
  Node A: key "user:5000" is in slot 7234
          slot 7234 is MIGRATING to Node B
          key NOT found locally (already migrated)

  Node A --> Client: -ASK 7234 192.168.1.2:6379

  Client --> Node B: ASKING           <-- MUST send ASKING first
  Client --> Node B: GET user:5000
  Node B --> Client: "alice"

  Client does NOT update slot_cache (migration may not be complete)
```

**Key difference**: MOVED triggers a permanent routing table update. ASK is a one-time redirect that does NOT update the client's slot map.

### Redirection Flow Diagram

```
  Client Request for key K (slot S)
       |
       v
  +-- Is slot S in local cache? --+
  |                                |
  | YES                            | NO (first request)
  | Route to cached node           | Route to any known node
  |                                |
  +------+------+---------+--------+
         |                |
    +----v----+     +-----v-----+
    | Correct |     | Wrong node|
    | node    |     +-----------+
    | responds|          |
    | with    |     +----+------+------+
    | data    |     |           |      |
    +---------+  -MOVED      -ASK   Response
                    |           |    (lucky hit)
              Update slot   One-time
              cache map     redirect
              permanently   (send ASKING
                            then retry)
```

---

## 4. Gossip Protocol and Failure Detection

### Gossip Protocol Mechanics

Every node sends a PING to a random node every second, carrying:

```
  PING Message Payload:
  +------------------------------------------+
  | sender_id          (160-bit node ID)     |
  | sender_slots[]     (16384-bit bitmap)    |
  | sender_epoch       (cluster config epoch)|
  | sender_flags       (master/replica/fail) |
  +------------------------------------------+
  | gossip_section[] (info about 3 random nodes):
  |   - node_id                              |
  |   - ip, port                             |
  |   - flags (PFAIL, FAIL, etc.)            |
  |   - last_ping_sent                       |
  |   - last_pong_received                   |
  +------------------------------------------+
```

### Failure Detection: Two-Phase

```
  Phase 1: PFAIL (Probable Failure) — Local suspicion
  +-----------------------------------------------------------+
  | Node A sends PING to Node X                               |
  | No PONG received within cluster-node-timeout (default 15s)|
  | Node A marks Node X as PFAIL locally                      |
  +-----------------------------------------------------------+
          |
          | Gossip propagates PFAIL flags
          v
  Phase 2: FAIL (Confirmed Failure) — Cluster consensus
  +-----------------------------------------------------------+
  | Node A collects PFAIL/FAIL reports from gossip messages   |
  | If majority of masters (N/2 + 1) report PFAIL for Node X |
  | within 2x cluster-node-timeout window:                    |
  |                                                           |
  | Node A promotes PFAIL --> FAIL                            |
  | Node A broadcasts FAIL message to ALL nodes               |
  | Failover process begins for Node X's replica              |
  +-----------------------------------------------------------+
```

### Failover Election (Raft-inspired)

```
  Replica R of failed Master M:

  1. R detects M is in FAIL state
  2. R increments currentEpoch
  3. R broadcasts FAILOVER_AUTH_REQUEST to all masters
  4. Each master votes for the FIRST replica requesting
     failover for a given epoch (prevents split vote)
  5. If R collects majority votes (N/2 + 1 masters):
     - R promotes itself to master
     - R claims M's slots with new configEpoch
     - R broadcasts UPDATE to the cluster

  Timeline:
  [M fails]--[PFAIL]--[FAIL broadcast]--[Election]--[Promotion]
     t=0      t=15s       t=15-30s        t=30-32s    t=32-35s

  Total failover time: 15-35 seconds (with default settings)
```

---

## 5. Epoch-Based Configuration Propagation

### Config Epoch vs Current Epoch

| Concept       | Scope      | Purpose                                  | Who Increments      |
|---------------|------------|------------------------------------------|---------------------|
| currentEpoch  | Cluster    | Monotonic logical clock for the cluster  | Any node (elections) |
| configEpoch   | Per-node   | Version of this node's slot assignment   | Promoted replicas    |

### Conflict Resolution Rule

```
  When two nodes claim the same slot:
    The node with the HIGHER configEpoch wins.

  Example:
    Node A claims slot 100 with configEpoch 5
    Node B claims slot 100 with configEpoch 8

    Node B wins. Node A must release slot 100.

  This is how Redis Cluster achieves "last failover wins" semantics
  and ensures convergence after network partitions heal.
```

### Epoch Propagation via Gossip

```
  +-------+   PING (myEpoch=5, slots=[0-5460])   +-------+
  | Node A| ---------------------------------------->| Node B|
  +-------+                                        +-------+
                                                        |
           If Node B knows a higher configEpoch for     |
           any of slots 0-5460, it replies with         |
           UPDATE message containing the newer config   |
                                                        |
  +-------+   UPDATE (slot 100, epoch=8, owner=C)  +-------+
  | Node A| <----------------------------------------| Node B|
  +-------+                                        +-------+
       |
       v
  Node A updates its slot map:
  slots[100].owner = Node C
  slots[100].configEpoch = 8
```

---

## 6. Live Resharding (Slot Migration)

### Migration Protocol

```
  Moving slot 7234 from Source (Node A) to Target (Node B):

  Step 1: Mark slot state
    Node B> CLUSTER SETSLOT 7234 IMPORTING <nodeA-id>
    Node A> CLUSTER SETSLOT 7234 MIGRATING <nodeB-id>

  Step 2: Migrate keys (iterative)
    Loop:
      Node A> CLUSTER GETKEYSINSLOT 7234 100   (get 100 keys)
      Node A> MIGRATE <nodeB-ip> 6379 "" 0 5000 KEYS k1 k2 k3 ...
        - MIGRATE is atomic: dumps + restores + deletes source
        - Uses DUMP + RESTORE internally with serialized RDB format
        - Keys are locked during individual MIGRATE (brief block)

  Step 3: Finalize
    Node B> CLUSTER SETSLOT 7234 NODE <nodeB-id>
    Node A> CLUSTER SETSLOT 7234 NODE <nodeB-id>
    (Broadcast to all other nodes)
```

### Migration State Machine

```
  Slot States:
  +----------+     SETSLOT IMPORTING     +----------+
  | NORMAL   | ----------------------->  | IMPORTING|  (Target)
  +----------+                           +----------+
       |                                      |
       | SETSLOT MIGRATING                    |
       v                                      v
  +----------+     Keys migrate over     +----------+
  | MIGRATING| ========================>| IMPORTING|
  +----------+     (MIGRATE command)     +----------+
       |                                      |
       | SETSLOT NODE (finalize)              | SETSLOT NODE
       v                                      v
  +----------+                           +----------+
  | NORMAL   |                           | NORMAL   |
  | (no slot)|                           | (owns it)|
  +----------+                           +----------+
```

### Request Routing During Migration

```
  Client request for key K in migrating slot S:

  At Source (MIGRATING state):
    if key K exists locally:
      execute command normally (key hasn't migrated yet)
    else:
      return -ASK <target>

  At Target (IMPORTING state):
    if client sent ASKING command first:
      execute command (accept migrating key access)
    else:
      return -MOVED <source>  (slot not officially mine yet)
```

**Critical for interviews**: During migration, there is a brief window where a key being MIGRATE'd is locked on the source. For large keys (e.g., a 1M-element sorted set), this can cause latency spikes of 100ms+. Production best practice: break large keys into smaller ones before migration, or use lazy migration approaches.

---

## 7. Cluster Constraints and Tradeoffs

### What Redis Cluster Does NOT Support

| Limitation                          | Reason                                | Workaround                      |
|-------------------------------------|---------------------------------------|----------------------------------|
| Multi-key ops across slots          | No distributed transaction protocol   | Hash tags for co-location        |
| SELECT (multi-database)             | Only db0 in cluster mode              | Use key prefixes instead         |
| Lua scripts with cross-slot keys    | Same as multi-key limitation          | Hash tags or split logic         |
| Pub/Sub scalability                 | Messages forwarded to all nodes       | Use Redis Streams instead        |
| Strong consistency                  | Async replication by design           | WAIT command (partial solution)  |
| Atomic cross-slot transactions      | No 2PC/3PC protocol                   | Application-level saga pattern   |

### WAIT Command for Semi-Synchronous Replication

```
  Client:
    SET critical:data "value"
    WAIT 1 5000    # Wait for 1 replica to ACK within 5000ms

  Returns: number of replicas that acknowledged

  Limitations:
  - Does NOT prevent data loss during simultaneous master+replica failure
  - Does NOT make replication synchronous (replica may fail after ACK)
  - Adds latency proportional to replication lag
  - Blocks only the calling client, not the server
```

---

## 8. Production Cluster Sizing

```
  Rule of Thumb for FAANG Scale:
  +--------------------------------------------------+
  | Metric            | Guideline                     |
  |-------------------|-------------------------------|
  | Nodes per cluster | 3-200 masters (practical max) |
  | Replicas per master| 1-2 (balance HA vs resources)|
  | Memory per node   | 25-50GB (leave COW headroom)  |
  | Cluster total     | Up to 10TB with 200 masters   |
  | Slots per node    | ~80-5461 (avoid extreme skew) |
  | Network           | 10Gbps minimum, 25Gbps ideal  |
  | cluster-node-timeout| 5-15s (lower = faster failover, more false positives)|
  +--------------------------------------------------+
```

---

## Interview Narrative

**When asked "How does Redis Cluster sharding work?" or "Explain the hash slot model":**

> "Redis Cluster uses a pre-sharded model with 16384 hash slots. Every key is mapped to a slot via CRC16 mod 16384, and each slot is assigned to exactly one master node. The choice of 16384 is driven by the gossip protocol's overhead: each heartbeat message carries a bitmap of all slots, and 16384 bits is 2KB, which keeps gossip bandwidth manageable even for clusters with 100+ nodes.
>
> Clients discover the slot-to-node mapping via CLUSTER SLOTS or CLUSTER SHARDS commands and cache it locally. When a client sends a command to the wrong node, it receives a MOVED redirection with the correct node address and updates its routing table permanently. During slot migration, the temporary ASK redirection is used instead, which tells the client to try the target node once without updating the cache.
>
> Failure detection is two-phase: individual nodes mark unresponsive peers as PFAIL after cluster-node-timeout, and when a majority of masters agree a node is PFAIL, it is promoted to FAIL, triggering an election. The replica of the failed master requests votes from all masters, and if it gets a majority, it promotes itself with a new configEpoch. This epoch-based config propagation ensures that after network partitions heal, all nodes converge to the same slot assignment: the highest configEpoch wins.
>
> Live resharding moves slots between nodes without downtime using a MIGRATING/IMPORTING state machine. Keys are migrated one batch at a time using the atomic MIGRATE command. During migration, the source serves keys it still has and ASK-redirects for keys that have already moved. The main production concern is that migrating large keys causes brief latency spikes because individual key migration is blocking.
>
> The fundamental tradeoff is that multi-key operations only work within a single slot. We use hash tags like {user:1001} to co-locate related keys. But hash tags create hot spots if a single entity receives disproportionate traffic, so the data model must balance co-location needs against load distribution."
