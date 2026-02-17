# Design the Storage Layer for a Global Chat System (WhatsApp/Messenger Scale)

## Staff/Principal-Level System Design Exercise

---

## 1. Problem Statement

Design the complete storage layer for a global chat system handling 2 billion active
users and 100 billion messages per day. The system must support 1:1 chats, group chats
(up to 1024 members), media sharing, read receipts, presence/status, end-to-end
encryption key management, full-text message search, and a tiered archive strategy.

This is not a "build a chat app" question. At staff level, you must reason about
write amplification, cross-region consistency tradeoffs, encryption key rotation at
scale, and cost-per-message economics that determine whether the business is viable.

---

## 2. Requirements

### 2.1 Functional Requirements

- **Message Storage**: Persist text, media references, reactions, edits, and deletes.
- **Conversation Metadata**: Track participants, mute state, pinned messages, and unread counts.
- **Presence/Status**: Online/offline/last-seen with configurable privacy.
- **Read Receipts**: Per-message delivery and read state for 1:1 and group chats.
- **Media Storage**: Photos, videos, voice messages, documents with thumbnails.
- **End-to-End Encryption (E2EE)**: Per-device key storage, ratchet state, group key distribution.
- **Message Search**: Full-text search across a user's entire chat history.
- **Archive/Retention**: Tiered storage — hot (recent), warm (30d), cold (>30d), glacier (>1yr).

### 2.2 Non-Functional Requirements

- **Write Throughput**: ~1.2M messages/sec sustained (100B/day).
- **Read Latency**: p50 < 5ms, p99 < 20ms for recent messages.
- **Durability**: Zero message loss — RPO = 0 for delivered messages.
- **Availability**: 99.99% (< 53 min downtime/year).
- **Global**: Serve users in 180+ countries with data residency compliance.
- **Cost**: Sub-$0.0001 per message stored (at scale, storage cost determines margins).

---

## 3. Scale Numbers

| Metric | Value |
|---|---|
| Monthly Active Users (MAU) | 2B |
| Daily Active Users (DAU) | 1.2B |
| Messages/day | 100B |
| Messages/sec (avg) | ~1.16M |
| Messages/sec (peak, 3x) | ~3.5M |
| Avg message size (text) | 200 bytes |
| Media messages (% of total) | ~15% |
| Avg media size | 500 KB |
| Daily text data | ~20 TB/day |
| Daily media data | ~7.5 PB/day |
| Active conversations | ~50B |
| Avg group size | 12 members |
| Read receipt events/sec | ~5M (group fan-out) |
| Presence updates/sec | ~2M |
| Search queries/sec | ~200K |

---

## 4. High-Level Architecture

```
                         ┌──────────────────────────────┐
                         │        Global DNS / LB        │
                         └──────────┬───────────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
     ┌────────▼───────┐   ┌────────▼───────┐   ┌────────▼───────┐
     │  Region: US-E   │   │  Region: EU-W   │   │  Region: APAC  │
     │                 │   │                 │   │                 │
     │ ┌─────────────┐ │   │ ┌─────────────┐ │   │ ┌─────────────┐ │
     │ │  Gateway     │ │   │ │  Gateway     │ │   │ │  Gateway     │ │
     │ │  (WebSocket) │ │   │ │  (WebSocket) │ │   │ │  (WebSocket) │ │
     │ └──────┬──────┘ │   │ └──────┬──────┘ │   │ └──────┬──────┘ │
     │        │        │   │        │        │   │        │        │
     │ ┌──────▼──────┐ │   │ ┌──────▼──────┐ │   │ ┌──────▼──────┐ │
     │ │ Msg Router   │ │   │ │ Msg Router   │ │   │ │ Msg Router   │ │
     │ └──────┬──────┘ │   │ └──────┬──────┘ │   │ └──────┬──────┘ │
     │        │        │   │        │        │   │        │        │
     │ ┌──────▼──────┐ │   │ ┌──────▼──────┐ │   │ ┌──────▼──────┐ │
     │ │ Write Path   │ │   │ │ Write Path   │ │   │ │ Write Path   │ │
     │ │ (Kafka)      │ │   │ │ (Kafka)      │ │   │ │ (Kafka)      │ │
     │ └──────┬──────┘ │   │ └──────┬──────┘ │   │ └──────┬──────┘ │
     │        │        │   │        │        │   │        │        │
     │ ┌──────▼──────┐ │   │ ┌──────▼──────┐ │   │ ┌──────▼──────┐ │
     │ │ Storage      │ │   │ │ Storage      │ │   │ │ Storage      │ │
     │ │ (Cassandra/  │ │   │ │ (Cassandra/  │ │   │ │ (Cassandra/  │ │
     │ │  ScyllaDB)   │ │   │ │  ScyllaDB)   │ │   │ │  ScyllaDB)   │ │
     │ └─────────────┘ │   │ └─────────────┘ │   │ └─────────────┘ │
     └─────────────────┘   └─────────────────┘   └─────────────────┘
              │                     │                     │
              └─────────────────────┼─────────────────────┘
                                    │
                     ┌──────────────▼──────────────┐
                     │   Cross-Region Replication   │
                     │   (Async, Conflict-Free)     │
                     └─────────────────────────────┘
```

---

## 5. Storage Layer Design

### 5.1 Message Storage (Primary — ScyllaDB/Cassandra)

**Why LSM-tree based wide-column store?**
- Write-optimized: 1.2M writes/sec is the dominant workload.
- Time-series access pattern: "fetch last N messages in conversation" maps perfectly
  to clustering key on timestamp.
- Linear horizontal scaling via consistent hashing.

**Schema Design:**

```
-- Messages table: Partitioned by conversation, clustered by time
CREATE TABLE messages (
    conversation_id  UUID,
    bucket           INT,          -- Time bucket (weekly) to bound partition size
    message_id       TIMEUUID,     -- Sortable, globally unique
    sender_id        BIGINT,
    message_type     TINYINT,      -- text, image, video, voice, system
    body             BLOB,         -- Encrypted payload
    media_ref        TEXT,         -- S3/blob storage reference
    reply_to         TIMEUUID,
    edited_at        TIMESTAMP,
    deleted          BOOLEAN,
    server_ts        TIMESTAMP,
    PRIMARY KEY ((conversation_id, bucket), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC)
  AND compaction = {'class': 'TimeWindowCompactionStrategy',
                    'compaction_window_size': 7,
                    'compaction_window_unit': 'DAYS'};
```

**Key design decisions:**
- **Bucketing**: Weekly time buckets prevent unbounded partition growth. A group chat
  with 1000 messages/day stays under 7K rows per partition per week.
- **TWCS compaction**: Optimal for time-series writes; avoids read amplification from
  overlapping SSTables in size-tiered compaction.
- **Encrypted body**: The server stores ciphertext only. No plaintext indexing possible
  (search is client-side or uses a separate encrypted index).

### 5.2 Conversation Metadata (ScyllaDB)

```
CREATE TABLE conversations (
    user_id          BIGINT,
    conversation_id  UUID,
    last_message_ts  TIMESTAMP,
    last_message_preview BLOB,     -- Encrypted snippet
    unread_count     INT,
    muted_until      TIMESTAMP,
    pinned           BOOLEAN,
    archived         BOOLEAN,
    PRIMARY KEY (user_id, last_message_ts)
) WITH CLUSTERING ORDER BY (last_message_ts DESC);
```

**Unread count challenge at group scale:**
- Naive: Increment counter for every group member on every message -> 1024 writes per
  message in a max-size group. At 100B messages/day with avg group of 12, this is
  ~1.2T counter updates/day.
- Optimization: Store `last_read_message_id` per user per conversation. Compute
  unread count lazily on fetch: `COUNT messages WHERE message_id > last_read_id`.
  Cache the result. Invalidate on new message or read event.

### 5.3 Presence and Status (Redis Cluster)

```
-- Presence: ephemeral, in-memory only
HSET presence:{user_id} status "online" last_seen 1706000000 region "us-east"

-- TTL-based expiry: if no heartbeat in 30s, key expires -> offline
EXPIRE presence:{user_id} 30
```

- **Why Redis?** Presence is ephemeral (no durability needed), high-frequency
  (heartbeats every 10s from 1.2B DAU = ~120M writes/sec at peak), and read-heavy
  (every chat screen checks presence).
- **Sharding**: Hash user_id across 500+ Redis nodes. Each node handles ~240K ops/sec.
- **Cross-region**: Each region maintains its own presence cluster. A lightweight
  gossip protocol syncs "is user online in ANY region" with ~2s convergence.

### 5.4 Read Receipts (Kafka + ScyllaDB)

```
-- Delivery/read state per message per recipient
CREATE TABLE read_receipts (
    conversation_id  UUID,
    message_id       TIMEUUID,
    user_id          BIGINT,
    delivered_at     TIMESTAMP,
    read_at          TIMESTAMP,
    PRIMARY KEY ((conversation_id, message_id), user_id)
);
```

**Fan-out problem**: A message in a 1024-member group generates up to 1023 receipt
records. Solution:
1. Batch receipts: Client sends "I have read up to message_id X" rather than
   per-message receipts.
2. Aggregate on read: For "who has read this message," query
   `last_read_id >= this_message_id` rather than storing per-message state.
3. Fire-and-forget via Kafka: Receipt events are produced to a compacted topic.
   Consumers update ScyllaDB asynchronously. UI shows "delivered" optimistically.

### 5.5 Media Storage (Object Store + CDN)

```
Upload Path:
  Client -> Gateway -> Presigned URL -> S3-compatible store (regional)
                                         │
                                         ├── Original (encrypted at rest)
                                         ├── Thumbnail (generated async)
                                         └── Transcoded variants

Metadata:
  media_id -> { bucket, key, size, mime_type, encryption_key_hash,
                thumbnail_key, duration, dimensions }
```

- **Storage tiers**: Hot (S3 Standard, 0-7d), Warm (S3-IA, 7-90d), Cold (S3 Glacier,
  >90d). Lifecycle policies automate transitions.
- **Cost math**: 7.5 PB/day * 365 = ~2.7 EB/year. At $0.023/GB (S3 Standard),
  hot tier alone = $62M/month. Aggressive tiering to Glacier ($0.004/GB) is existential.
- **Deduplication**: Hash-based dedup for viral content. A forwarded meme shared 10M
  times stores one copy with 10M references. Saves ~30% media storage.

### 5.6 E2EE Key Storage (Dedicated Encrypted KV Store)

```
-- Per-device identity keys
CREATE TABLE identity_keys (
    user_id        BIGINT,
    device_id      UUID,
    public_key     BLOB,         -- Curve25519
    signed_prekey  BLOB,
    prekey_sig     BLOB,
    uploaded_at    TIMESTAMP,
    PRIMARY KEY (user_id, device_id)
);

-- One-time prekeys (consumed on first contact)
CREATE TABLE one_time_prekeys (
    user_id        BIGINT,
    prekey_id      INT,
    public_key     BLOB,
    PRIMARY KEY (user_id, prekey_id)
);
```

**Critical considerations:**
- One-time prekeys are **consumed** (deleted after use). This is a DELETE-heavy
  workload on an otherwise write-heavy system. Use a separate keyspace with
  LeveledCompactionStrategy to handle tombstone cleanup efficiently.
- Prekey exhaustion: If a user's device is offline and all prekeys are consumed,
  fall back to signed prekey (less forward secrecy). Monitor prekey inventory;
  push replenishment requests when count < 20.
- Key rotation: Group ratchet requires re-keying when a member leaves. For a 1024-member
  group, this means distributing 1023 new key shares. Use sender keys (Signal Protocol
  variant) to reduce this to O(1) per sender.

### 5.7 Message Search (Encrypted Search Index)

Full-text search over E2EE messages is architecturally contentious. Two approaches:

**Approach A: Client-Side Index (WhatsApp model)**
- Index lives on the device. SQLite FTS5 locally.
- Pros: Zero server-side plaintext exposure.
- Cons: No cross-device search, no search on new device until history re-downloaded.

**Approach B: Server-Side Encrypted Index (Blind Index)**
- Client computes `HMAC(term, search_key)` for each word and uploads blind tokens.
- Server stores `{blind_token -> [message_id list]}` in a dedicated search cluster.
- Query: Client sends `HMAC(query_term, search_key)`, server returns matching message_ids.
- Cons: Leaks access patterns (which tokens are queried). Mitigate with ORAM or
  batched dummy queries.

**Recommended**: Approach A for default, Approach B opt-in for business/enterprise tier.

### 5.8 Archive and Tiering Strategy

```
┌────────────────────────────────────────────────────┐
│                   Message Lifecycle                  │
│                                                      │
│  Hot (ScyllaDB SSD)    0-7 days      p99 < 10ms    │
│  ──────────────────────────────────────────────     │
│  Warm (ScyllaDB HDD)   7-30 days     p99 < 50ms    │
│  ──────────────────────────────────────────────     │
│  Cold (Parquet on S3)   30d-1yr      p99 < 500ms   │
│  ──────────────────────────────────────────────     │
│  Glacier (S3 Glacier)   >1yr         p99 < hours   │
│                                                      │
│  Compaction job runs daily, moves data down tiers   │
└────────────────────────────────────────────────────┘
```

- Hot-to-warm: Change ScyllaDB read priority; data stays in same cluster but on
  cheaper HDD-backed nodes.
- Warm-to-cold: Export to Parquet files keyed by `(conversation_id, week)`.
  Register in a Hive metastore for ad-hoc analytics. Serve via a cold-read
  microservice that fetches from S3 on demand.
- Cold-to-glacier: Automated S3 lifecycle rule. Retrieval requires user-initiated
  "download history" flow with a progress indicator (minutes to hours).

---

## 6. Detailed Component Design

### 6.1 Write Path (Critical Path)

```
1. Client sends message over WebSocket (encrypted payload)
2. Gateway validates auth token, assigns server_ts and message_id (TIMEUUID)
3. Gateway produces to Kafka topic: messages.{region}.{partition_key=conv_id}
4. Kafka acks to gateway (durability guarantee)
5. Gateway sends delivery receipt to sender (server has it)
6. Consumer reads from Kafka:
   a. Write to ScyllaDB (messages table)
   b. Update conversation metadata (last_message_ts, increment unread hints)
   c. Fan-out: Produce to per-recipient delivery topics
7. Recipient's gateway reads delivery topic, pushes via WebSocket
8. If recipient offline: Write to pending_messages table, trigger push notification
```

**Latency budget**: Steps 1-5 must complete in < 100ms (user-perceived send latency).
Steps 6-7 are asynchronous (target < 500ms end-to-end delivery).

### 6.2 Cross-Region Replication

- **Model**: Async replication with last-writer-wins (LWW) on conversation metadata.
  Messages are immutable (no conflict). Edits/deletes use TIMEUUID-based ordering.
- **Kafka MirrorMaker 2**: Replicate message topics across regions. Consumer offset
  translation ensures exactly-once processing per region.
- **Conflict resolution**: For mutable fields (mute, pin, archive), use CRDTs:
  - `muted_until`: Max-wins register.
  - `pinned_messages`: Add-wins OR-set.
  - `unread_count`: Derived from `last_read_id` (convergent by construction).

### 6.3 Data Residency

- Some jurisdictions (EU/GDPR, Russia, India) require data to stay within borders.
- Partition conversations by the "home region" of the conversation creator.
- Cross-border chats: Replicate to both regions, but the "source of truth" lives
  in the creator's home region. Metadata references point to the canonical region.
- Deletion requests (GDPR Article 17): Propagate tombstones across all replicas
  with a 30-day hard-delete compaction window.

---

## 7. Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| ScyllaDB node failure | Partial read/write degradation | RF=3, CL=LOCAL_QUORUM; tolerate 1 node down per DC |
| Kafka broker failure | Write path stalls | 3-way replication, ISR >= 2, auto-leader election |
| Full region outage | Users in that region cannot send/receive | DNS failover to nearest region; pending messages queue |
| Presence Redis failure | All users appear offline | Graceful degradation: hide presence UI, do not block messaging |
| Media S3 outage | Cannot upload/download media | Multi-region S3 with cross-region replication; serve from replica |
| Encryption key loss | User loses access to messages | Key backup to cloud (encrypted with user passphrase); recovery flow |
| Kafka consumer lag | Messages delayed | Auto-scale consumers; alert on lag > 10s; prioritize 1:1 over group |
| Cross-region repl lag | Stale conversation metadata | Show "syncing" indicator; do not block local writes |
| Tombstone accumulation | Read latency spike in ScyllaDB | gc_grace_seconds tuning; repair cycles; monitoring tombstone/read ratio |

---

## 8. Cost Analysis

| Component | Monthly Cost Estimate |
|---|---|
| ScyllaDB cluster (hot, 3 regions, 500 nodes) | $8M |
| ScyllaDB cluster (warm, HDD-backed) | $2M |
| S3 (media, with tiering) | $15M |
| Kafka clusters (3 regions) | $3M |
| Redis (presence, 500 nodes) | $2M |
| CDN (media delivery) | $10M |
| Network (cross-region transfer) | $5M |
| Search infrastructure | $1M |
| **Total** | **~$46M/month** |

**Cost per message**: $46M / (100B * 30) = ~$0.000015/message.
**Cost per MAU**: $46M / 2B = $0.023/user/month.

At these unit economics, the system is viable only with ad-supported or subscription
models generating > $0.03/user/month.

---

## 9. Evolution Path (Handling 10x Growth)

### 9.1 From 2B to 20B Messages/Day (10x Writes)

- **ScyllaDB**: Add nodes linearly. ScyllaDB scales horizontally; 10x nodes for 10x throughput.
- **Kafka**: Increase partition count. Rebalance consumers.
- **Already designed for this**: The bucketing and TWCS strategy handles increased data volume.

### 9.2 From 2B to 20B Users

- **Partition key cardinality**: `conversation_id` is already UUID; no hotspot risk.
- **Metadata fan-out**: More conversations per user. Paginate the conversation list
  endpoint; do not load all conversations on app open.
- **Presence**: 10x Redis nodes. Consider switching to a probabilistic structure
  (Bloom filter for "is online") to reduce memory.

### 9.3 From 100B to 1T Messages/Day

- Move to a custom LSM-tree storage engine optimized for this exact access pattern.
  (This is what Meta/WhatsApp actually did — built a custom storage layer atop RocksDB.)
- Introduce a write-ahead buffer in DRAM with battery-backed persistence.
- Shard Kafka by conversation_id hash AND time to limit topic partition counts.

### 9.4 AI Features (Summarization, Smart Reply)

- Add a streaming consumer that processes messages through an LLM pipeline.
- Store summaries alongside conversation metadata.
- Key challenge: E2EE prevents server-side processing. Require client-side inference
  or an explicit opt-in that decrypts on a TEE (Trusted Execution Environment).

---

## 10. Interview Narrative

### How to Present This in 45 Minutes

**Minutes 0-5: Clarify scope.** "I will focus on the storage layer, not the transport
(WebSocket, push notifications). I will design for 2B MAU and 100B messages/day. I will
address E2EE constraints on search and analytics."

**Minutes 5-15: Data model and primary store.** Walk through the ScyllaDB schema.
Explain bucketing, TWCS, and why Cassandra-family is the right choice over relational
or document stores. Explicitly call out the partition key design and how it avoids
hotspots.

**Minutes 15-25: Write path and replication.** Draw the Kafka-backed write path.
Explain the async cross-region model with CRDTs for metadata. Address the read receipt
fan-out problem. This is where you demonstrate systems maturity — acknowledging that
the "obvious" per-message receipt model breaks at group scale.

**Minutes 25-35: Hard problems.** E2EE key management (prekey exhaustion, group
ratchet). Encrypted search tradeoffs. Media cost at exabyte scale. These are the
topics that separate L5 from L6/L7 answers.

**Minutes 35-45: Cost, failure modes, and evolution.** Walk through the cost table.
Show you understand that storage cost is a business constraint, not just a technical
one. Discuss how the architecture handles 10x growth without a rewrite.

### Key Phrases That Signal Staff-Level Thinking

- "The write amplification of per-message read receipts in groups makes this O(N^2).
  We solve it by storing last_read_id and computing counts lazily."
- "At exabyte scale, the S3 bill dominates total cost. Tiering to Glacier is not
  optional — it is the difference between a viable and non-viable product."
- "E2EE fundamentally limits server-side search. We must choose between user
  experience (server-side) and privacy (client-side). I recommend client-side by
  default with an enterprise opt-in."
- "CRDTs for metadata convergence avoid the need for distributed transactions
  across regions, which would violate our latency SLO."
- "This design is operationally simple — the write path has exactly one durable
  commit (Kafka produce). Everything downstream is idempotent and replayable."
