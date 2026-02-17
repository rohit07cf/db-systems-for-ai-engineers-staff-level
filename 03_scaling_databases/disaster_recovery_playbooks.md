# Disaster Recovery Playbooks: From RPO/RTO to Chaos Engineering

## Why This Matters at Staff+ Level

Disaster recovery is not a feature -- it is an operational discipline. At the staff/principal level,
you own the DR strategy end-to-end: defining RPO/RTO targets with business stakeholders, designing
backup and failover architectures, building automation that works under pressure, and running chaos
experiments to validate it all before a real incident forces your hand.

---

## 1. RPO/RTO Framework

### Definitions

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  ◄────── RPO ──────►       ◄──────── RTO ────────►         │
  │                      │                              │       │
  │  Last good           │ Disaster                     │       │
  │  backup/checkpoint   │ occurs                       │       │
  │                      │                    Service   │       │
  │                      │                    restored  │       │
  │  ─────────────────── X ──────────────────────────── ✓ ──── │
  │                                                             │
  │  RPO = Recovery Point Objective                             │
  │        Maximum acceptable data loss (measured in time)      │
  │                                                             │
  │  RTO = Recovery Time Objective                              │
  │        Maximum acceptable downtime (measured in time)       │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### RPO/RTO by Business Domain

| Domain               | RPO Target     | RTO Target     | Cost Implication         |
|----------------------|:--------------:|:--------------:|--------------------------|
| Financial ledger      | 0 (zero loss)  | < 1 minute     | Synchronous replication  |
| E-commerce orders     | < 1 second     | < 5 minutes    | Semi-sync + hot standby  |
| User profiles         | < 1 minute     | < 15 minutes   | Async replication        |
| Analytics warehouse   | < 1 hour       | < 4 hours      | Periodic snapshots       |
| ML training data      | < 24 hours     | < 24 hours     | Daily backups            |
| Log/telemetry data    | Best effort    | Best effort    | Re-derive from source    |

### Cost vs RPO/RTO Curve

```
  Annual Cost
      │
  $1M │ ●
      │   ●
  $500K│     ●
      │       ●
  $100K│          ●
      │              ●
  $50K │                   ●
      │                         ●  ●  ●
  $10K │                                    ●
      └────────────────────────────────────────
       0   1s  1m  5m  15m  1h   4h  12h  24h
                    RPO / RTO

  The last order of magnitude (1min -> 0) costs as much
  as the entire rest of the DR budget.
```

---

## 2. Backup Strategies

### Strategy Comparison

| Strategy                | RPO          | Recovery Speed | Storage Cost | Operational Cost |
|------------------------|:------------:|:--------------:|:------------:|:----------------:|
| Continuous replication  | ~0           | Minutes        | 2-3x         | High             |
| WAL/binlog archiving    | Seconds      | 10-60 minutes  | 1.1-1.5x     | Medium           |
| Incremental snapshots   | Hours        | 30-120 minutes | 1.2-2x       | Low              |
| Full daily backup       | Up to 24h   | 1-4 hours      | 1.5-2x       | Low              |
| Logical dump (pg_dump)  | Up to 24h   | Hours          | 1x           | Low              |

### Backup Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │  Primary Database                                        │
  │  ┌────────────┐                                          │
  │  │ WAL / Binlog│──── Continuous ────► Archive Storage    │
  │  │ Stream      │     Shipping        (S3, GCS)          │
  │  └────────────┘                                          │
  │                                                          │
  │  ┌────────────┐                                          │
  │  │ Snapshot   │──── Every 6 hours ──► Snapshot Storage   │
  │  │ Engine     │                       (EBS snapshots)    │
  │  └────────────┘                                          │
  │                                                          │
  │  ┌────────────┐                                          │
  │  │ Logical    │──── Daily ──────────► Dump Storage       │
  │  │ Backup     │                       (S3, cross-region) │
  │  └────────────┘                                          │
  └──────────────────────────────────────────────────────────┘

  Three layers of defense:
  1. WAL archive: RPO in seconds, fast PITR
  2. Snapshots: RPO in hours, fast volume restore
  3. Logical dump: RPO in days, cross-engine portability
```

### Backup Validation (The Part Everyone Skips)

```
  ┌──────────────────────────────────────────────────────────┐
  │  BACKUP IS NOT DR. TESTED RESTORE IS DR.                 │
  └──────────────────────────────────────────────────────────┘

  Weekly automated restore test:
  1. Restore latest backup to isolated environment
  2. Run integrity checks (CHECKSUM TABLE, pg_checksums)
  3. Run application smoke tests against restored data
  4. Compare row counts with production (within backup window)
  5. Alert if any step fails
  6. Record restore time (track RTO trend)

  ┌──────────────────────────────────────────────────┐
  │  Restore Test Dashboard                          │
  │                                                  │
  │  Week   Restore Time   Integrity   Smoke Test    │
  │  ─────  ────────────   ─────────   ──────────    │
  │  W1     42 min         PASS        PASS          │
  │  W2     45 min         PASS        PASS          │
  │  W3     FAILED         ---         ---     !!!   │
  │  W4     41 min         PASS        PASS          │
  │  W5     67 min         PASS        FAIL    !!    │
  │                                                  │
  │  W3: backup file corrupted (S3 lifecycle policy  │
  │      deleted intermediate files)                 │
  │  W5: schema migration broke smoke test queries   │
  └──────────────────────────────────────────────────┘
```

---

## 3. Point-in-Time Recovery (PITR)

### How It Works

```
  Base Snapshot (T0)     WAL Segments          Target Time
       │                 (continuous)               │
       ▼                                            ▼
  ┌─────────┐   ┌────┐ ┌────┐ ┌────┐ ┌────┐   ┌───────┐
  │ Snapshot │ + │ W1 │ │ W2 │ │ W3 │ │ W4 │ = │ State │
  │ at T0    │   │    │ │    │ │    │ │    │   │ at T  │
  └─────────┘   └────┘ └────┘ └────┘ └────┘   └───────┘

  Recovery process:
  1. Restore base snapshot (volume-level or logical)
  2. Replay WAL segments from T0 to target time T
  3. Stop replay at exact target timestamp
  4. Database is now at state as-of time T
```

### PITR Configuration (PostgreSQL Example)

```
  postgresql.conf:
    archive_mode = on
    archive_command = 'aws s3 cp %p s3://wal-archive/%f'
    wal_level = replica

  recovery.conf (during restore):
    restore_command = 'aws s3 cp s3://wal-archive/%f %p'
    recovery_target_time = '2024-01-15 14:30:00 UTC'
    recovery_target_action = 'promote'
```

### PITR Failure Modes

| Failure                             | Impact                          | Prevention                    |
|-------------------------------------|--------------------------------|-------------------------------|
| GAP in WAL archive                  | Cannot recover past the gap    | Monitor archive lag, alert    |
| Base snapshot corrupted             | Must use older snapshot + more WAL | Multiple snapshot copies   |
| WAL replay takes too long           | RTO exceeded                   | More frequent base snapshots  |
| Target time before oldest snapshot  | Cannot recover                 | Retention policy review       |
| Archive storage unavailable         | Cannot recover                 | Cross-region archive copies   |

---

## 4. Cross-Region Failover Automation

### Failover Architecture

```
  ┌───────────────────────────────────────────────────────────┐
  │                    Health Checker                          │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
  │  │ Check: TCP  │  │ Check: SQL  │  │ Check: App  │       │
  │  │ connectivity│  │ SELECT 1    │  │ health EP   │       │
  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │
  │         └────────────────┼────────────────┘               │
  │                    ┌─────▼──────┐                          │
  │                    │ Decision   │                          │
  │                    │ Engine     │                          │
  │                    │            │                          │
  │                    │ 3/3 fail   │─── Automatic failover    │
  │                    │ for 30s+   │                          │
  │                    └─────┬──────┘                          │
  │                          │                                │
  │         ┌────────────────┼────────────────┐               │
  │         ▼                ▼                ▼               │
  │  Promote        Update DNS /       Notify        │
  │  Standby        Route53            On-call        │
  │  to Primary                                       │
  └───────────────────────────────────────────────────────────┘
```

### Failover Timeline

```
  T+0s:    Primary region goes down
  T+10s:   Health checks detect failure (3 consecutive failures)
  T+30s:   Decision engine confirms (avoid false positive)
  T+35s:   Standby promoted to primary
  T+40s:   DNS/routing updated (Route53 health check failover)
  T+60s:   DNS TTL expires, clients connect to new primary
  T+90s:   Application connections established
  T+120s:  Service fully restored

  Total RTO: ~2 minutes (automated)
  vs Manual: 15-45 minutes (human response + decision + execution)
```

### Failover Decision Matrix

| Signal                         | Auto-Failover? | Rationale                           |
|--------------------------------|:--------------:|-------------------------------------|
| Primary unreachable for 30s+   | Yes            | Clear infrastructure failure        |
| Primary reachable but slow     | No             | May be transient; failover costly   |
| Replication lag > 30s          | No             | Indicates issue but not outage      |
| Primary disk full              | No             | Fixable without failover            |
| Primary OOM killed             | Yes (restart)  | Auto-restart first, failover if repeated |
| Region-wide network outage     | Yes            | Multi-signal confirms region down   |
| Single AZ failure              | Maybe          | Depends on multi-AZ deployment      |

### Split-Brain Prevention

```
  CRITICAL: Two nodes both believing they are primary = data corruption

  Prevention mechanisms:
  1. STONITH (Shoot The Other Node In The Head)
     - Before promoting standby, FENCE the old primary
     - AWS: stop-instance API call; GCP: reset-instance
     - Guarantees old primary cannot accept writes

  2. Quorum-based promotion
     - Standby must get agreement from majority of witnesses
     - If network partition, minority side cannot promote

  3. Lease-based leadership
     - Primary holds a lease (e.g., in etcd/ZooKeeper)
     - Lease expires after N seconds without renewal
     - Standby waits for lease expiry + safety margin before promoting

  ┌──────────────────────────────────────────────────────┐
  │  Timeline for lease-based failover:                  │
  │                                                      │
  │  T+0s:   Primary crashes, stops renewing lease       │
  │  T+10s:  Lease TTL expires (10s TTL)                 │
  │  T+15s:  Standby confirms lease expired + 5s margin  │
  │  T+16s:  Standby acquires new lease, becomes primary │
  │                                                      │
  │  Safety margin prevents split-brain from clock skew  │
  └──────────────────────────────────────────────────────┘
```

---

## 5. Data Integrity Validation

### Post-Failover Integrity Checks

```
  Immediately after failover:
  ┌─────────────────────────────────────────────────────┐
  │ 1. Row count comparison (new primary vs last known) │
  │    Acceptable delta: RPO window of writes           │
  │                                                     │
  │ 2. Checksum critical tables                         │
  │    CHECKSUM TABLE accounts, transactions, orders;   │
  │                                                     │
  │ 3. Foreign key integrity scan                       │
  │    Check for orphaned rows (parent deleted but      │
  │    child still exists due to replication lag)        │
  │                                                     │
  │ 4. Application-level invariant checks               │
  │    - Sum of all account balances = expected total   │
  │    - No negative inventory counts                   │
  │    - No duplicate transaction IDs                   │
  │                                                     │
  │ 5. Compare latest N transactions with source-of-     │
  │    truth (if available, e.g., Kafka event log)      │
  └─────────────────────────────────────────────────────┘
```

### Reconciliation Patterns

```
  When data loss is detected (transactions in RPO gap):

  Strategy 1: Replay from event log
    - If all writes flow through Kafka/event bus
    - Replay events from last confirmed position
    - Idempotent handlers prevent duplicates

  Strategy 2: Reconcile from upstream systems
    - Payment processor has authoritative transaction record
    - Pull transactions from payment provider API
    - Insert missing transactions into new primary

  Strategy 3: Manual reconciliation
    - For complex cases, generate discrepancy report
    - Business operations team reviews and resolves
    - Typically for financial systems with regulatory requirements
```

---

## 6. Runbook Templates for Common Failures

### Runbook: Primary Database Unresponsive

```
  ┌──────────────────────────────────────────────────────────┐
  │  RUNBOOK: Primary Database Unresponsive                  │
  │  Severity: P1 (customer-facing impact)                   │
  │  On-call: Database SRE                                   │
  │                                                          │
  │  STEP 1: TRIAGE (0-2 minutes)                            │
  │  □ Check monitoring dashboard for root cause:            │
  │    - CPU > 95%? -> likely runaway query                  │
  │    - Disk full? -> likely WAL accumulation               │
  │    - OOM killed? -> check dmesg / instance metrics       │
  │    - Network? -> check VPC flow logs                     │
  │                                                          │
  │  STEP 2: ATTEMPT RECOVERY (2-5 minutes)                  │
  │  □ If query-related: KILL long-running queries           │
  │  □ If disk: emergency log cleanup / volume expand        │
  │  □ If OOM: restart with increased memory / kill bloater  │
  │  □ If network: check SG/NACL, contact cloud provider     │
  │                                                          │
  │  STEP 3: FAILOVER DECISION (5-10 minutes)                │
  │  □ If recovery failed and primary still unresponsive:    │
  │    - Confirm replication lag on standby                   │
  │    - Estimate data loss (RPO impact)                     │
  │    - Get approval from incident commander                │
  │    - Execute failover: promote standby                   │
  │                                                          │
  │  STEP 4: POST-FAILOVER (10-30 minutes)                   │
  │  □ Verify application connectivity to new primary        │
  │  □ Run integrity checks (see section 5)                  │
  │  □ Monitor error rates for 15 minutes                    │
  │  □ Update DNS if not automatic                           │
  │                                                          │
  │  STEP 5: RESTORE REDUNDANCY (30 min - 4 hours)           │
  │  □ Provision new standby from new primary                │
  │  □ Verify replication is healthy                         │
  │  □ Investigate and remediate root cause on old primary   │
  │  □ Schedule post-incident review                         │
  └──────────────────────────────────────────────────────────┘
```

### Runbook: Replication Lag Exceeds Threshold

```
  ┌──────────────────────────────────────────────────────────┐
  │  RUNBOOK: Replication Lag > 30 seconds                   │
  │  Severity: P2 (degraded consistency, no outage)          │
  │                                                          │
  │  STEP 1: IDENTIFY CAUSE                                  │
  │  □ Check primary write throughput (spike?)               │
  │  □ Check replica CPU/IO (saturated?)                     │
  │  □ Check network between primary and replica             │
  │  □ Check for DDL operations on primary                   │
  │  □ Check for long-running transactions blocking replay   │
  │                                                          │
  │  STEP 2: MITIGATE                                        │
  │  □ If write spike: throttle batch jobs, defer backfills  │
  │  □ If replica saturated: scale replica, kill analytics   │
  │  □ If DDL: wait for completion (expected lag)            │
  │  □ If long transaction: evaluate termination             │
  │                                                          │
  │  STEP 3: MONITOR RECOVERY                                │
  │  □ Watch lag trending toward zero                        │
  │  □ Set alert for lag re-spike                            │
  │  □ Confirm application read-your-writes still working    │
  │                                                          │
  │  ESCALATION:                                             │
  │  □ If lag > 5 minutes and growing: escalate to P1        │
  │  □ If lag > 30 minutes: disable read routing to replica  │
  └──────────────────────────────────────────────────────────┘
```

---

## 7. Chaos Engineering for Databases

### Principles

```
  ┌──────────────────────────────────────────────────────────┐
  │  1. Start with a hypothesis                              │
  │     "If the primary fails, the standby promotes within   │
  │      2 minutes and no data is lost"                      │
  │                                                          │
  │  2. Design the experiment                                │
  │     Inject failure, measure outcome, compare to hypothesis│
  │                                                          │
  │  3. Minimize blast radius                                │
  │     Start in staging. Graduate to production canary.     │
  │     Always have an abort mechanism.                       │
  │                                                          │
  │  4. Run in production (eventually)                       │
  │     Staging cannot replicate production's complexity.    │
  │     Real DR confidence requires real environment testing.│
  └──────────────────────────────────────────────────────────┘
```

### Experiment Catalog

| Experiment                     | Method                          | Validates                        |
|--------------------------------|--------------------------------|----------------------------------|
| Kill primary process           | `kill -9 postgres`             | Auto-restart / failover          |
| Network partition (primary)    | iptables DROP                  | Standby promotion, split-brain   |
| Slow disk I/O                  | `tc qdisc` or `dm-delay`      | Query timeout handling           |
| Fill disk to 100%              | `dd if=/dev/zero`              | WAL archiving, alerts, recovery  |
| Corrupt WAL segment            | Bit-flip in WAL file           | PITR resilience, checksum detection |
| Kill standby during failover   | Kill -9 mid-promotion          | Failover retry / third replica   |
| DNS failure                    | Block Route53 resolution       | Connection retry, caching        |
| Clock skew injection           | `date --set` on one node       | HLC behavior, replication health |

### Game Day Execution Template

```
  ┌──────────────────────────────────────────────────────────┐
  │  GAME DAY: Database Failover Drill                       │
  │  Date: [scheduled]                                       │
  │  Participants: DB SRE, App Eng, Incident Commander       │
  │                                                          │
  │  PRE-DRILL (1 week before):                              │
  │  □ Notify stakeholders                                   │
  │  □ Verify monitoring and alerting                        │
  │  □ Confirm rollback procedure                            │
  │  □ Brief participants on experiment plan                 │
  │                                                          │
  │  DRILL EXECUTION:                                        │
  │  T+0:    Inject failure (kill primary)                   │
  │  T+30s:  Verify: alerts fired?                           │
  │  T+60s:  Verify: failover initiated?                     │
  │  T+120s: Verify: application recovered?                  │
  │  T+300s: Verify: data integrity checks pass?             │
  │  T+600s: Verify: new standby provisioned?                │
  │                                                          │
  │  MEASUREMENTS:                                           │
  │  □ Time to detect (TTD):     _____ seconds               │
  │  □ Time to failover (TTF):   _____ seconds               │
  │  □ Time to recovery (TTR):   _____ seconds               │
  │  □ Data loss (if any):       _____ transactions          │
  │  □ Error rate during event:  _____ %                     │
  │                                                          │
  │  POST-DRILL:                                             │
  │  □ Compare results to RTO/RPO targets                    │
  │  □ Document gaps and action items                        │
  │  □ Schedule remediation work                             │
  │  □ Plan next drill date                                  │
  └──────────────────────────────────────────────────────────┘
```

---

## 8. Post-Incident Review Patterns

### Timeline Reconstruction

```
  ┌──────────────────────────────────────────────────────────┐
  │  INCIDENT TIMELINE: 2024-01-15 Primary DB Outage         │
  │                                                          │
  │  14:23:00  Disk I/O latency begins increasing            │
  │  14:25:12  First user-visible errors (5xx responses)     │
  │  14:25:45  PagerDuty alert fires (P99 > 5s)             │
  │  14:26:30  On-call acknowledges alert                    │
  │  14:28:00  On-call identifies disk I/O as root cause     │
  │  14:30:00  Attempts to kill long-running vacuum process  │
  │  14:32:00  Vacuum killed, but damage done (disk full)    │
  │  14:35:00  Decision to failover                          │
  │  14:36:00  Failover initiated (standby promotion)        │
  │  14:38:30  New primary accepting connections             │
  │  14:39:00  Application reconnected                       │
  │  14:42:00  Error rate returns to baseline                │
  │  14:55:00  Integrity checks complete (no data loss)      │
  │  15:30:00  New standby provisioned, replication healthy  │
  │                                                          │
  │  TTD: 2m 45s | TTF: 3m 30s | Total outage: 17 minutes   │
  └──────────────────────────────────────────────────────────┘
```

### Blameless Post-Mortem Structure

```
  1. INCIDENT SUMMARY
     What happened, impact, duration, affected services.

  2. TIMELINE
     Minute-by-minute reconstruction (see above).

  3. ROOT CAUSE ANALYSIS
     - Proximate cause: disk filled due to autovacuum + large transaction
     - Contributing factor: disk alert threshold set too high (90% vs 80%)
     - Contributing factor: no runbook for disk-full scenario

  4. IMPACT ASSESSMENT
     - Duration: 17 minutes
     - Affected users: ~50,000 (all US-East users)
     - Data loss: None (RPO met)
     - Revenue impact: ~$X,000 estimated

  5. WHAT WENT WELL
     - Failover automation worked as designed
     - On-call responded within 60 seconds
     - Integrity checks confirmed no data loss

  6. WHAT WENT POORLY
     - Disk alert fired too late (90% threshold)
     - No runbook for disk-full scenario
     - Autovacuum not tuned for large table

  7. ACTION ITEMS
     □ Lower disk alert threshold to 75% (P2, due: next sprint)
     □ Create disk-full runbook (P2, due: next sprint)
     □ Tune autovacuum for large tables (P3, due: next quarter)
     □ Add disk growth prediction alert (P3, due: next quarter)
     □ Schedule game day to test disk-full scenario (P3, due: Q2)

  8. RECURRENCE PREVENTION
     How will we prevent this class of failure, not just this instance?
```

---

## 9. DR Maturity Model

| Level | Description            | Characteristics                                    |
|:-----:|------------------------|----------------------------------------------------|
| 0     | No DR                  | Single instance, manual backups, no standby        |
| 1     | Basic Backups          | Automated backups, manual restore tested annually  |
| 2     | Standby Ready          | Hot/warm standby, manual failover, quarterly drills|
| 3     | Automated Failover     | Auto-failover, PITR, monthly drills, runbooks      |
| 4     | Multi-Region Active    | Active-active or fast cross-region failover        |
| 5     | Chaos-Validated        | Regular chaos experiments in production, <1min RTO |

Most companies are at Level 1-2. Level 3 is the target for production systems with SLAs.
Level 4-5 is required for systems with 99.99%+ availability commitments.

---

## Interview Narrative

**When asked about disaster recovery, structure your answer as follows:**

> "I frame DR around two business-driven metrics: RPO (how much data can we afford to lose) and
> RTO (how long can we be down). These are not engineering decisions -- they are business
> decisions with engineering cost implications. I work with business stakeholders to set targets
> per data tier, because not all data justifies the same investment.
>
> For a typical production system, I target RPO < 1 second and RTO < 5 minutes. To achieve this,
> I use three layers: continuous WAL/binlog archiving to cloud storage (RPO in seconds), a hot
> standby with synchronous or semi-synchronous replication (RTO in minutes via automated
> failover), and periodic full snapshots (belt-and-suspenders against archive corruption).
>
> Automated failover is where most teams underinvest. I implement it as a state machine with
> health checks (TCP, SQL, application-level), a decision engine that requires multiple
> consecutive failures before triggering (to avoid false positives), and a fencing mechanism
> (STONITH) to prevent split-brain. The standby must acquire a lease before promoting -- this
> guarantees the old primary cannot accept writes even if it recovers.
>
> But the most critical element is validation. A backup you have never restored is not a backup --
> it is a hope. I run automated weekly restore tests that spin up a fresh instance from the latest
> backup, run integrity checks and application smoke tests, and record the restore time. This
> gives me confidence that when I need the backup, it works, and the restore time is within RTO.
>
> Finally, I validate the entire failover pipeline through quarterly game days -- controlled chaos
> experiments where we kill the primary in production (or a production-like environment) and
> measure TTD (time to detect), TTF (time to failover), and TTR (time to recover). Each game day
> generates action items that improve the next one. The goal is that a real disaster is just
> another game day."

**Follow-up traps to prepare for:**

- "How do you handle a corrupted backup discovered during a real disaster?" (Answer: maintain
  multiple backup generations; the last 7 daily snapshots + 4 weekly snapshots; corruption in
  the latest triggers fallback to the previous generation)
- "What about cross-region failover for globally distributed users?" (Answer: DNS-based routing
  with health checks; each region has a standby; failover promotes the standby in the surviving
  region and updates DNS weights; TTL must be low enough for fast convergence)
- "How do you test for split-brain in production?" (Answer: I do not create actual split-brain;
  I test the prevention mechanism by simulating network partitions and verifying the fencing
  mechanism fires correctly; the test assertion is that exactly one node accepts writes at all
  times)
