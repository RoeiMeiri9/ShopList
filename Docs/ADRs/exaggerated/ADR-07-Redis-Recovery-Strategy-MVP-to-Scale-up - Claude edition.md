# **ADR-09 — Redis Recovery Strategy & Coordinator (Future Architecture)**

**Status:**

- Accepted: current Redis recovery behavior
- Proposed: future coordinated recovery mechanism

**Decision Type:**

- Architecture – Recovery / Distributed Systems

**Applies to:**

- All microservices that read/write Redis + emit Kafka events

---

## **1. Context**

The CoList platform uses **Redis as the real-time source of truth** and **Kafka as the durable event log**.

During the early stages of development (with around 10-12 running services), Redis failures or resets are:

- Infrequent
- Easy to recover manually
- Low-traffic, low-contention scenarios
- No heavy burst of parallel writes

Therefore, the system is initially built **without** a distributed Redis recovery coordinator.

However, as the system grows into multiple services and replicas (e.g., 12–40 service types × multiple instances → 100–400 Redis writers), an uncontrolled Redis restart could trigger a **thundering herd** of services simultaneously rewriting missing state into Redis.

This poses risks:

- Redis overload immediately after restart
- Write amplification
- Inconsistent partial states
- Replay storms from many independent services
- Delays in fully reconstructing state
- Possible cascading failures

**Critical gap in initial design**: The current approach does not guarantee full recovery when Kafka logs have been cleaned/expired. This ADR defines a comprehensive solution that ensures Redis can **always** be recovered, even when:

- Kafka retention has expired old events
- Redis persistence files are corrupted or lost
- Multiple layers of backup have failed

To prevent thundering herd and guarantee recovery, a dedicated **Recovery Coordinator** and **multi-layered backup strategy** will be required at scale.

This ADR defines **the future recovery strategy**, even though it is **NOT fully implemented at the current stage**.

---

## **2. Decision**

### _For now:_

Redis recovery is **simple**:

- Redis persistence (AOF+RDB) enabled with replicas
- If Redis is wiped → manual/full restart → services repopulate organically via normal operation
- Kafka is still the authoritative history log
- No coordinator
- No throttled replay
- No distributed recovery protocol

This is correct and optimal for the small-scale system.

**Critical gap identified**: Current approach does not guarantee full recovery when Kafka logs have been cleaned/expired. This ADR now defines the comprehensive solution.

---

### _In the future:_

When scaling up, CoList **will adopt a comprehensive, multi-layered Redis recovery strategy** that guarantees recovery **even when Kafka logs have expired**.

---

### **Recovery Architecture: Multiple Layers of Defense**

To ensure Redis can **always** be recovered, the system implements defense in depth:

#### **Layer 1: Redis Native Persistence & Replication**

- AOF (Append-Only File) + RDB snapshots enabled
- Redis Sentinel or Redis Cluster for automatic failover
- Multiple replicas for high availability
- **Recovery time**: Seconds to minutes
- **Coverage**: Protects against Redis process crashes, not data wipes

#### **Layer 2: Kafka Compacted Topics (State Snapshots)**

- Critical entities stored in compacted Kafka topics that retain the latest value for each key
- Compacted topics for: `users.latest`, `lists.latest`, `items.latest`, `chats.latest`, `product_history.latest`
- These topics provide a full snapshot of the latest state for every key, not just recently changed keys
- **Recovery time**: Minutes to tens of minutes
- **Coverage**: Survives standard Kafka retention (e.g., 7-30 days), enables recovery even after event log cleanup

#### **Layer 3: Periodic Materialized Snapshots to Object Storage**

- Every 6-24 hours (configurable), create full Redis dump or materialized view export
- Store snapshots in S3/GCS with longer retention (90+ days)
- Snapshots include: complete Redis state or DB materialized views representing Redis data
- Snapshots can be encrypted and stored in multiple data centers for disaster recovery
- **Recovery time**: Hours (depends on snapshot size and network)
- **Coverage**: Long-term disaster recovery, survives extended Kafka retention expiry

#### **Layer 4: Database as Ultimate Source of Truth**

- PostgreSQL contains the authoritative, durable state written by DBWriter
- **Only accessed during recovery** by authorized Recovery Service (see below)
- **Recovery time**: Hours (full rebuild from DB)
- **Coverage**: Absolute guarantee - can always reconstruct Redis from DB if all else fails

---

### **A. Recovery Service (DBReader/Restorer) — Authorized Component**

**Key principle**: Regular services never read from DB. However, a **dedicated, authorized Recovery Service** is permitted to read from DB **only during Redis recovery operations**.

**Recovery Service characteristics**:

- Separate microservice or privileged mode of DBWriter
- Has read-only access to DB (granted via separate database role/credentials)
- **Only activated** when Redis recovery is needed and faster methods (Layers 1-3) are insufficient
- Operates under coordinator control (see below)
- Fully audited: all DB reads and Redis writes logged
- Can throttle writes to avoid overwhelming Redis during rebuild

**Security & Access Control**:

- Recovery Service uses different credentials than regular services
- DB access limited to read-only operations
- Kubernetes RBAC or similar controls who can trigger recovery mode
- All recovery operations logged to audit trail

---

### **B. Recovery Coordinator ("Leader") for Controlled Recovery**

One service instance elected as coordinator using:

- Kubernetes Lease API, or
- Kafka consumer group leadership, or
- etcd/Consul consensus

**Coordinator responsibilities**:

- Declares recovery mode via global flag (Kafka control topic or ConfigMap)
- Executes recovery algorithm (see Section C)
- Throttles Redis writes to prevent overload
- Tracks recovery progress and checkpoints
- Validates recovery success before resuming normal operations
- Only the coordinator writes to Redis during recovery

---

### **C. Recovery Algorithm: Prioritized, Fail-Safe Process**

When Redis is wiped or becomes unavailable:

#### **Step 1: Detect & Announce Recovery Mode**

- Health monitor (k8s probe or external) detects Redis unavailability
- Coordinator elected/determined
- Coordinator sets `redis_recovery_in_progress = true` in control topic/ConfigMap
- **All services** immediately stop writing to Redis and publish events to Kafka only

#### **Step 2: Try Fastest Restore — Native Redis Persistence**

- Attempt to restore from latest RDB/AOF files
- If successful and recent (< 1 hour old) → **Skip to Step 6**
- If failed or stale → **Continue to Step 3**

#### **Step 3: Restore from Kafka Compacted Topics**

- Coordinator reads all messages from compacted topics to rebuild the latest state for each key
- Rebuild order (by entity priority):
  1. Users (authentication/authorization data)
  2. Lists (core domain objects)
  3. Items (content data)
  4. Chats & Messages (communication data)
  5. Product history & analytics counters
  6. Ephemeral/derived state
- Write to Redis with rate limiting (e.g., 10K writes/sec)
- If compacted topics cover all data adequately → **Skip to Step 6**
- If gaps exist (old entities missing from compacted topics) → **Continue to Step 4**

#### **Step 4: Restore from Object Storage Snapshots**

- Coordinator downloads latest snapshot from S3/GCS
- Load snapshot into Redis (restore from RDB, or replay materialized view)
- If snapshot recent and complete → **Continue to Step 5 (replay recent events)**
- If snapshot too old or incomplete → **Continue to Step 5 (DB fallback)**

#### **Step 5: Fallback to Database Recovery (Last Resort)**

- **Activate Recovery Service** with DB read access
- Recovery Service queries DB for all entities:
  ```sql
  SELECT * FROM users WHERE deleted_at IS NULL;
  SELECT * FROM lists WHERE deleted_at IS NULL;
  -- etc for all entities
  ```
- Build Redis data structures from DB rows
- Compute derived state (counters, unread indicators) from DB or set to defaults
- Write to Redis with throttling
- This step **guarantees** recovery even when all other layers fail

#### **Step 6: Replay Recent Kafka Events**

- After base state restored (from any of Steps 2-5), replay events from Kafka
- Replay from: latest snapshot offset or compaction point → current offset
- Apply events in order to update Redis with recent changes
- Handle version conflicts using event version numbers (if applicable)

#### **Step 7: Validation & Consistency Checks**

- Coordinator performs spot checks: sample entities from Redis vs DB
- Verify critical counts (user count, list count) are reasonable
- Check for obvious inconsistencies
- If major issues detected → **abort, alert operators, retry recovery**

#### **Step 8: Resume Normal Operations**

- Coordinator sets `redis_recovery_in_progress = false`
- All services resume normal Redis writes (and continue Kafka writes)
- Emit recovery metrics: duration, data volume restored, method used
- Alert operations team of successful recovery

---

### **D. Service Behavior During Recovery**

When `redis_recovery_in_progress = true`:

**All regular services MUST**:

- Check the recovery flag before every Redis write
- If flag is true:
  - **Do NOT write to Redis**
  - Publish events to Kafka (event sourcing continues)
  - Serve read requests from Redis (degraded mode: may return stale data or "unavailable" errors)
- Continue normal operation for reads (with appropriate error handling)

**Example code pattern**:

```javascript
async function writeToCache(key, value) {
  const isRecovering = await checkRecoveryFlag();
  if (isRecovering) {
    // Only publish to Kafka during recovery
    await publishEvent({ type: "ENTITY_UPDATED", key, value });
    return; // Do not write to Redis
  }

  // Normal operation: write to both
  await redis.set(key, value);
  await publishEvent({ type: "ENTITY_UPDATED", key, value });
}
```

---

### **E. Compacted Topics Configuration**

For critical entities, configure Kafka topics with compaction:

```bash
# Example: Create compacted topic for users
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic users.latest \
  --partitions 10 \
  --replication-factor 3 \
  --config cleanup.policy=compact \
  --config min.cleanable.dirty.ratio=0.5 \
  --config segment.ms=3600000
```

**Compaction guidelines**:

- Use compacted topics for: users, lists, items, chats, product metadata
- Non-compacted topics for: transient events, logs, analytics streams
- Set retention to "compact" to keep only the most recent value for each key
- Regularly verify compaction is occurring (monitor `kafka.log:type=LogCleanerManager`)

---

### **F. Snapshot Strategy Details**

**Automated snapshot creation** (executed by Snapshotter service or DBWriter):

1. **Trigger**: Every 6-24 hours (configurable, based on data volatility)
2. **Process**:
   - Generate Redis RDB dump (`BGSAVE` command), OR
   - Export materialized view from DB (SELECT all live entities), OR
   - Combine: RDB for hot data + DB export for complete coverage
3. **Storage**:
   - Upload to S3/GCS with naming: `redis-snapshot-{timestamp}.rdb`
   - Retain snapshots for 90+ days
   - Encrypt snapshots for security and transfer to multiple data centers
4. **Metadata**: Store snapshot metadata in control topic (timestamp, size, coverage %)

**Snapshot restoration**:

- Download snapshot from S3/GCS
- Load into Redis (for RDB) or replay via Recovery Service (for DB export)
- Resume from this point + replay Kafka events since snapshot timestamp

---

## **3. Rationale**

### Why postpone full recovery infrastructure until the system grows?

- **Complexity**: Adds significant development overhead (coordinator, leader election, recovery service, compacted topics, snapshot management)
- **Dependencies**: Requires distributed consensus, control channels, throttling mechanisms
- **Operational burden**: More components to deploy, monitor, and maintain
- **Testing complexity**: Must test all recovery paths, coordinator failover, partial failures
- **Development velocity**: Early-stage development benefits from simplicity over resilience

At MVP scale (10-12 services, low traffic):

- Redis failures are rare and recoverable manually
- Simple AOF+RDB persistence is sufficient
- No thundering herd risk with few services
- Operator can manually trigger recovery if needed

### Why this multi-layered approach is the right future design?

**Guarantees recovery under all scenarios**:

- Layer 1 (persistence): Fast recovery from process crashes (seconds)
- Layer 2 (compacted topics): Recovery from data wipes within Kafka retention window (minutes)
- Layer 3 (object storage): Recovery from extended outages beyond Kafka retention (hours)
- Layer 4 (database): Absolute guarantee—can always rebuild from source of truth

**Respects architectural constraints**:

- Regular services never read DB (maintains separation of concerns)
- Only authorized Recovery Service accesses DB during controlled recovery
- Maintains event-sourcing model: all changes still published to Kafka
- Kafka remains the real-time event log; DB remains the durable store

**Prevents thundering herd at scale**:

- Coordinator ensures only one process writes to Redis during recovery
- Global recovery flag prevents all services from writing simultaneously
- Throttled replay protects Redis from overload
- Controlled, ordered rebuild ensures consistency

**Operational practicality**:

- Automatic recovery from most common failures (Layers 1-2)
- Manual intervention only for rare disasters (Layers 3-4)
- Clear recovery metrics and monitoring
- Testable recovery procedures with defined SLAs

### Why Recovery Service is acceptable despite "no DB reads" rule?

The architecture rule "services don't read from DB" is designed to:

1. Maintain separation: writes centralized in DBWriter, reads from Redis cache
2. Prevent tight coupling between services and DB schema
3. Enable independent scaling of services and database

**Recovery Service doesn't violate these goals because**:

- It's not a regular service—it's infrastructure/ops tooling
- Used only in exceptional circumstances (Redis total failure + Kafka expiry)
- Operates under coordinator control, not in normal request path
- Access is privileged and audited
- Maintains the separation: services still don't read DB, only Recovery Service does

**Analogy**: Just as DBWriter is the only component that writes to DB, Recovery Service is the only component that reads from DB during recovery. Both are specialized, controlled exceptions to enable the overall architecture.

---

## **4. Consequences**

### **Positive (now)**

- Faster development
- Simpler architecture
- Lower cognitive load
- No need for leader election at MVP scale
- No need for recovery mode logic in every service

---

### **Positive (later)**

- Safe Redis rebuild at scale
- No stampedes
- No inconsistent rebuilds
- Deterministic replay
- Predictable operational behavior
- Clear invariants for runtime and recovery phases
- **Guaranteed recovery** even when Kafka logs expired
- Multiple fallback options for different failure scenarios
- Automatic recovery without manual intervention (most cases)

---

### **Negative**

- Requires system-wide update when implemented
- Requires ADR changes + refactoring of write paths
- Adds complexity to operations
- Requires testing recovery flows and leader failover
- Additional infrastructure costs (S3 storage, compacted topics)
- More moving parts to monitor and maintain

---

## **5. Implementation Plan (Future)**

### **Phase 0 — Today (MVP Stage)**

**Current state**:

- Redis AOF+RDB enabled with basic persistence
- Kafka receives all events (standard topics with time-based retention)
- No coordinator, no recovery service
- Manual recovery if Redis fails

**Immediate actions** (minimal prep for future):

- Document current Redis persistence configuration
- Verify AOF+RDB snapshots are working
- Ensure Kafka retention is adequate (7-30 days recommended)
- Add basic monitoring: Redis availability, Kafka lag

---

### **Phase 1 — Pre-scale Preparation (8-15 services)**

**Goals**: Lay groundwork for automated recovery without full complexity

**Actions**:

1. **Enable Kafka compacted topics** for core entities:

   - Create `users.latest`, `lists.latest`, `items.latest`, `chats.latest` with `cleanup.policy=compact`
   - Update services to publish to both standard topics (for events) and compacted topics (for state snapshots)
   - Monitor compaction is occurring (`kafka.log:type=LogCleanerManager`)

2. **Implement recovery flag** in shared config:

   - Add `redis_recovery_in_progress` flag to Kafka control topic or ConfigMap
   - Modify service write paths to check flag:
     ```javascript
     if (await isRecoveryMode()) {
       await publishToKafka(event); // Only Kafka
       return;
     }
     await redis.set(key, value); // Normal: Redis + Kafka
     await publishToKafka(event);
     ```
   - Add feature flag to enable/disable this check (for gradual rollout)

3. **Implement basic snapshot automation**:

   - Create Snapshotter service or add to DBWriter
   - Schedule periodic Redis `BGSAVE` every 12-24 hours
   - Upload RDB files to S3/GCS with retention policy (90 days)
   - Store snapshot metadata (timestamp, size) in control topic

4. **Add monitoring & alerts**:
   - `redis.recovery_mode{status="active|inactive"}`
   - `redis.snapshot.last_success_timestamp`
   - `kafka.compacted_topics.coverage{topic}`
   - Alert on Redis unavailability > 5 minutes

---

### **Phase 2 — Coordinator & Recovery Service MVP (15-30 services)**

**Goals**: Implement automated, coordinated recovery for common failure scenarios

**Actions**:

1. **Implement Coordinator with leader election**:

   - Use Kubernetes Lease API for leader election
   - Coordinator monitors Redis health
   - On failure: set recovery flag, execute recovery algorithm
   - Implement Steps 1-3 of recovery algorithm (persistence → compacted topics)

2. **Build Recovery Service (DBReader)**:

   - Create separate service with DB read-only credentials
   - Implement Step 5 of recovery algorithm (DB → Redis rebuild)
   - Only activated by Coordinator when needed
   - Full audit logging of all operations

3. **Implement throttled Kafka replay**:

   - Coordinator replays events from compacted topics or standard topics
   - Rate limiting: configurable writes/second to Redis
   - Progress checkpointing: store offset in control topic
   - Resume capability: recover from coordinator failure during recovery

4. **Testing & validation**:

   - Chaos testing: intentionally wipe Redis in staging
   - Measure recovery time for different scenarios:
     - Restore from persistence (seconds)
     - Restore from compacted topics (minutes)
     - Restore from DB (hours)
   - Validate data consistency after recovery
   - Test coordinator failover during recovery

5. **Create runbook**:
   - Step-by-step recovery procedures
   - Manual trigger instructions (when automatic fails)
   - Rollback procedures
   - Troubleshooting guide

---

### **Phase 3 — Optimized Recovery (30+ services, high scale)**

**Goals**: Optimize for speed, reliability, and operational excellence

**Actions**:

1. **Smart rebuild strategies**:

   - Hot-key prioritization: rebuild frequently-accessed keys first
   - Parallel replay: partition-aware parallel recovery (careful synchronization)
   - Incremental snapshots: delta snapshots to reduce snapshot size

2. **Enhanced snapshot strategy**:

   - Hybrid snapshots: RDB for hot data + DB export for cold data
   - Multiple snapshot frequencies: hourly for hot data, daily for cold data
   - Snapshot validation: checksums, consistency checks

3. **Advanced monitoring & observability**:

   - Recovery time SLA tracking by scenario
   - Recovery dashboard: progress, ETA, current step
   - Automated consistency validation: periodic Redis ↔ DB spot checks
   - Cost tracking: S3 storage, data transfer

4. **Operational improvements**:
   - Automated disaster recovery drills: monthly Redis wipe in staging
   - Recovery simulation tool: estimate recovery time for production
   - Multi-region recovery: snapshots replicated across regions
   - Recovery playbooks for different failure modes

---

### **Phase-by-Phase Recovery Capabilities**

| Phase       | Failure Scenario         | Recovery Time | Method                                       | Automation        |
| ----------- | ------------------------ | ------------- | -------------------------------------------- | ----------------- |
| **Phase 0** | Redis process crash      | < 1 min       | AOF/RDB restore                              | Automatic (Redis) |
| **Phase 1** | Redis data wipe          | 10-60 min     | Manual restore from snapshot                 | Manual            |
| **Phase 1** | + Kafka compacted topics | 5-30 min      | Manual replay from compacted                 | Manual            |
| **Phase 2** | Redis data wipe          | 5-30 min      | Auto restore: persistence → compacted topics | Automatic         |
| **Phase 2** | + Kafka expiry           | 1-4 hours     | Auto restore: snapshot or DB                 | Automatic         |
| **Phase 3** | Any failure              | 2-20 min      | Optimized auto restore (parallel, smart)     | Automatic         |

---

## **6. Required Metrics (Future)**

### **Recovery Process Metrics**

- `redis.recovery.in_progress{status="active|inactive"}` — Boolean indicating recovery mode
- `redis.recovery.coordinator.leader{instance}` — Current coordinator instance
- `redis.recovery.current_step{step="persistence|compacted|snapshot|db|replay"}` — Current recovery step
- `redis.recovery.progress_percent` — Overall recovery progress (0-100%)
- `redis.recovery.duration_seconds{method}` — Time to complete recovery by method
- `redis.recovery.method_used{method="persistence|compacted|snapshot|db"}` — Which recovery method succeeded

### **Kafka Replay Metrics**

- `kafka.replay.offset_lag{topic, partition}` — How far behind current offset during replay
- `kafka.replay.records_processed{topic}` — Number of records replayed
- `kafka.replay.write_rate_throttle{status="active|inactive"}` — Whether throttling is active
- `kafka.replay.errors{topic, type}` — Replay errors by topic and error type

### **Compacted Topics Metrics**

- `kafka.compacted.coverage_percent{topic}` — Percentage of keys present in compacted topic
- `kafka.compacted.last_compaction_timestamp{topic}` — When compaction last ran
- `kafka.compacted.size_bytes{topic}` — Current size of compacted topic

### **Snapshot Metrics**

- `redis.snapshot.last_success_timestamp` — When last snapshot succeeded
- `redis.snapshot.size_bytes` — Size of latest snapshot
- `redis.snapshot.duration_seconds` — Time to create snapshot
- `redis.snapshot.upload_duration_seconds` — Time to upload to S3/GCS
- `redis.snapshot.age_seconds` — Age of latest snapshot (for staleness detection)
- `redis.snapshot.s3_retention_count` — Number of snapshots in S3/GCS

### **Recovery Service (DBReader) Metrics**

- `recovery.db_reader.activated{status="true|false"}` — Whether DB reader is active
- `recovery.db_reader.queries_executed` — Number of DB queries executed during recovery
- `recovery.db_reader.rows_processed{table}` — Rows read from each DB table
- `recovery.db_reader.write_rate{unit="keys/sec"}` — Rate of writes to Redis
- `recovery.db_reader.errors{table, type}` — Errors during DB read or Redis write

### **Consistency Validation Metrics**

- `redis.validation.consistency_check_result{status="pass|fail"}` — Result of spot checks
- `redis.validation.divergence_count{entity_type}` — Number of inconsistencies detected
- `redis.validation.last_check_timestamp` — When last consistency check ran

### **Operational Health Metrics**

- `redis.health.availability{status="up|down"}` — Redis availability status
- `redis.health.persistence.aof_size_bytes` — Current AOF file size
- `redis.health.persistence.rdb_last_save_timestamp` — Last RDB save time
- `redis.health.replication.lag_seconds` — Replication lag for Redis replicas

### **Alerting Thresholds** (examples)

- Recovery in progress > 30 minutes → P2 alert
- Recovery failed → P1 alert (page on-call)
- Snapshot age > 36 hours → P3 alert
- Compacted topic coverage < 80% → P3 alert
- Consistency check failures > 5% → P2 alert
- Redis unavailable > 5 minutes → P1 alert

---

## **7. Security & Operational Considerations**

### **Security**

**Recovery Service Access Control**:

- Separate database credentials for Recovery Service (read-only role)
- Kubernetes RBAC to control who can trigger recovery mode manually
- All recovery operations logged to audit trail with timestamps and actors
- Recovery flag changes require privileged access

**Snapshot Security**:

- Encrypt snapshots at rest in S3/GCS (AES-256)
- Encrypt snapshots in transit (TLS for uploads)
- Access to snapshot storage restricted via IAM/GCS policies
- Snapshots may contain sensitive data—treat as production data

**Coordinator Security**:

- Leader election secured via Kubernetes Lease API (namespace-scoped)
- Only coordinator pod can set recovery flag
- Coordinator logs all actions for audit trail

### **Operational Practices**

**Testing & Drills**:

- Monthly: Test recovery from persistence (Phase 1+)
- Quarterly: Test recovery from compacted topics (Phase 2+)
- Bi-annually: Test recovery from DB (Phase 2+)
- Annual: Multi-region disaster recovery drill
- Always test in staging before production

**Runbook Requirements**:

- Step-by-step procedures for each recovery scenario
- Decision tree: which recovery method to use when
- Manual override instructions if automation fails
- Rollback procedures if recovery introduces bad data
- Contact information for escalation

**Monitoring & Alerting**:

- Dashboard showing recovery status, progress, ETA
- Alerts for recovery initiated, recovery failed, recovery duration exceeded
- Proactive alerts: snapshot staleness, compacted topic coverage
- On-call runbook integration: link from alert to runbook

**Capacity Planning**:

- Redis sizing: account for full dataset + overhead during recovery
- S3/GCS costs: snapshot size × retention days × replication factor
- Network bandwidth: consider snapshot download time during recovery
- Coordinator resources: memory for tracking state, CPU for replay

**Consistency Management**:

- During recovery, services may see stale or missing data in Redis
- Services should handle Redis misses gracefully (fallback to "unavailable" or default values)
- After recovery, services may need to refresh local caches
- Consider short "warm-up" period after recovery before marking Redis as fully available

### **Failure Modes & Mitigation**

| Failure Mode                      | Impact                   | Mitigation                                                              |
| --------------------------------- | ------------------------ | ----------------------------------------------------------------------- |
| Coordinator fails during recovery | Recovery stalls          | New leader elected, resumes from checkpoint                             |
| Recovery Service crashes          | DB recovery fails        | Coordinator retries, alerts on-call                                     |
| S3/GCS unavailable                | Can't retrieve snapshots | Fall back to DB recovery, multiple region snapshots                     |
| Kafka unavailable during recovery | Can't replay events      | Wait for Kafka, fall back to DB if timeout                              |
| DB unavailable during recovery    | DB recovery fails        | Wait for DB, alert on-call, no automated fallback                       |
| Compacted topics incomplete       | Partial state restored   | Fall back to older snapshot or DB recovery                              |
| Recovery takes too long           | Extended downtime        | Partial warm-up: prioritize hot keys, mark Redis degraded but available |

### **Data Consistency Guarantees**

**During recovery**:

- Services continue writing to Kafka (events not lost)
- Redis may have stale or missing data (services must handle gracefully)
- No writes to DB (DBWriter paused or respects recovery flag)

**After recovery**:

- Redis reflects state as of recovery completion time
- Recent events (during recovery) applied via Kafka replay
- Version numbers/timestamps resolve conflicts if present
- Spot checks validate Redis ↔ DB consistency
- If major divergence detected: alert, investigate, potentially re-run recovery

**Eventual consistency model**:

- System is eventually consistent after recovery completes
- Brief window (seconds to minutes) where Redis may lag behind Kafka
- Services designed to tolerate temporary inconsistency

### **Cost Considerations**

**Storage costs** (Phase 2+):

- Compacted Kafka topics: ~10-30% overhead vs. standard topics (more keys retained)
- S3/GCS snapshots: ~1-5 GB per snapshot × retention days (e.g., 90 days = 90-450 GB)
- Redis AOF/RDB: local disk, minimal cost

**Compute costs** (Phase 2+):

- Recovery Service: small footprint, only runs during recovery (< 0.1% of total compute)
- Coordinator: negligible overhead (monitoring + leader election)
- Snapshotter: runs periodically, low CPU (BGSAVE is background)

**Network costs**:

- Snapshot uploads to S3/GCS: ~1-5 GB per snapshot (depending on data size)
- Snapshot downloads during recovery: same as upload (infrequent)
- Kafka replication: standard Kafka costs (compaction doesn't significantly increase network)

**Total estimated overhead**: < 5% of infrastructure costs for comprehensive recovery capability

---

## **8. Status & Timeline**

**Current Status (Phase 0)**:

- Redis AOF+RDB enabled
- Kafka standard topics with time-based retention
- No recovery coordinator, no Recovery Service
- Manual recovery only

**Planned Implementation**:

- **Phase 1** (Pre-scale): Start when 8-15 services deployed (~Q2 2025 estimated)
  - Implement compacted topics, recovery flag, basic snapshots
  - Estimated effort: 2-3 weeks development + 1 week testing
- **Phase 2** (Coordinator MVP): Start when 15-30 services deployed (~Q3-Q4 2025 estimated)
  - Implement Coordinator, Recovery Service, automated recovery
  - Estimated effort: 4-6 weeks development + 2 weeks testing
- **Phase 3** (Optimized): Start when 30+ services deployed (~Q1 2026+ estimated)
  - Optimize recovery speed, advanced monitoring
  - Estimated effort: 3-4 weeks development + ongoing iteration

**Review & Update Trigger**:

- Review this ADR when service count reaches 8
- Update based on actual failure incidents or near-misses
- Revisit if Kafka retention policy changes significantly

---

## **9. References**

### **Internal Documentation**

- ADR-06 Write Path Guarantees (TBD)
- ADR-05 Event Envelope and Version Numbers (TBD)
- DBWriter service specification
- CoList Architecture Overview

### **External Resources**

**Redis Persistence**:

- Redis offers both RDB snapshots (point-in-time dumps) and AOF (append-only file) for data durability
- Redis forks a child process to write snapshots, allowing the parent to continue serving requests
- Redis Persistence documentation: https://redis.io/docs/management/persistence/

**Kafka Log Compaction**:

- Log compaction in Kafka provides finer-grained per-record retention, keeping at least the last known value for each record key
- Compaction is key-based retention where the goal is to keep the most recent value for a given key
- Compacted logs are useful for restoring state after crashes, providing a full snapshot of final record values
- Kafka Compaction guide: https://docs.confluent.io/kafka/design/log_compaction.html

**Distributed Coordination**:

- Kubernetes Lease API for leader election: https://kubernetes.io/docs/concepts/architecture/leases/
- Kafka consumer group coordination: https://kafka.apache.org/documentation/#consumerapi

**Disaster Recovery Patterns**:

- Disaster recovery involves transferring backups to external data centers to secure data against catastrophic events
- AWS S3 for backup storage: https://aws.amazon.com/s3/
- Google Cloud Storage: https://cloud.google.com/storage

---

## **Appendix A: Recovery Decision Tree**

```
Redis unavailable detected
    ↓
Coordinator elected & recovery flag set
    ↓
Try: Restore from RDB/AOF
    ├─ Success & recent? → Skip to Kafka replay (Step 6)
    └─ Failed or stale? → Continue
          ↓
Try: Restore from Kafka compacted topics
    ├─ Complete coverage? → Continue to Kafka replay (Step 6)
    └─ Incomplete? → Continue
          ↓
Try: Restore from S3/GCS snapshot
    ├─ Recent & complete? → Continue to Kafka replay (Step 6)
    └─ Too old or missing? → Continue
          ↓
Fallback: Activate Recovery Service (DB read)
    ├─ Success? → Continue to Kafka replay (Step 6)
    └─ Failed? → ALERT ON-CALL, manual intervention
          ↓
Kafka replay: Apply recent events
    ↓
Validation: Spot check consistency
    ├─ Pass? → Resume normal operations
    └─ Fail? → ALERT, investigate, retry if needed
```

---

## **Appendix B: Example Service Code (Recovery Flag Check)**

**Node.js / TypeScript example**:

```typescript
// Shared utility function
async function isRecoveryMode(): Promise<boolean> {
  // Option 1: Check Kafka control topic (cache for 10s)
  const cachedFlag = cache.get("redis_recovery_in_progress");
  if (cachedFlag !== undefined) return cachedFlag;

  const latestMessage = await kafkaAdmin.fetchLatestMessage("control.recovery");
  const flag = latestMessage?.value?.in_progress ?? false;
  cache.set("redis_recovery_in_progress", flag, 10); // 10s TTL
  return flag;

  // Option 2: Check Kubernetes ConfigMap (if using K8s)
  // const configMap = await k8sApi.readNamespacedConfigMap('redis-recovery', 'default');
  // return configMap.data.in_progress === 'true';
}

// In your service's write path
async function updateEntity(entityId: string, data: any) {
  // Always publish to Kafka (event sourcing)
  await publishEvent({
    type: "ENTITY_UPDATED",
    entityId,
    data,
    timestamp: Date.now(),
  });

  // Only write to Redis if NOT in recovery mode
  if (await isRecoveryMode()) {
    logger.info("Skipping Redis write: recovery in progress", { entityId });
    return;
  }

  await redis.hset(`entity:${entityId}`, data);
  logger.debug("Entity updated in Redis and Kafka", { entityId });
}
```

**Go example**:

```go
// Global recovery flag cache
var recoveryModeCache struct {
    sync.RWMutex
    value     bool
    expiresAt time.Time
}

func isRecoveryMode(ctx context.Context) (bool, error) {
    // Check cache first
    recoveryModeCache.RLock()
    if time.Now().Before(recoveryModeCache.expiresAt) {
        result := recoveryModeCache.value
        recoveryModeCache.RUnlock()
        return result, nil
    }
    recoveryModeCache.RUnlock()

    // Fetch from Kafka control topic
    msg, err := kafkaReader.FetchLatestMessage(ctx, "control.recovery")
    if err != nil {
        return false, fmt.Errorf("failed to check recovery mode: %w", err)
    }

    inProgress := msg != nil && msg.Value["in_progress"] == true

    // Update cache
    recoveryModeCache.Lock()
    recoveryModeCache.value = inProgress
    recoveryModeCache.expiresAt = time.Now().Add(10 * time.Second)
    recoveryModeCache.Unlock()

    return inProgress, nil
}

func updateEntity(ctx context.Context, entityID string, data map[string]interface{}) error {
    // Always publish to Kafka
    if err := publishEvent(ctx, Event{
        Type:      "ENTITY_UPDATED",
        EntityID:  entityID,
        Data:      data,
        Timestamp: time.Now(),
    }); err != nil {
        return fmt.Errorf("failed to publish event: %w", err)
    }

    // Check recovery mode before writing to Redis
    recovering, err := isRecoveryMode(ctx)
    if err != nil {
        log.Warn("Failed to check recovery mode, skipping Redis write", "error", err)
        return nil // Fail open: skip Redis write if can't determine mode
    }

    if recovering {
        log.Info("Skipping Redis write: recovery in progress", "entity_id", entityID)
        return nil
    }

    // Write to Redis
    if err := redisClient.HSet(ctx, fmt.Sprintf("entity:%s", entityID), data).Err(); err != nil {
        return fmt.Errorf("failed to write to Redis: %w", err)
    }

    return nil
}
```

---

## **Appendix C: Compacted Topic Configuration Examples**

**Create compacted topic for users**:

```bash
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic users.latest \
  --partitions 10 \
  --replication-factor 3 \
  --config cleanup.policy=compact \
  --config min.cleanable.dirty.ratio=0.5 \
  --config segment.ms=3600000 \
  --config delete.retention.ms=86400000
```

**Producer code to publish to compacted topic**:

```typescript
// Publish user update to compacted topic
async function publishUserSnapshot(userId: string, userData: User) {
  await kafkaProducer.send({
    topic: "users.latest",
    messages: [
      {
        key: userId, // Critical: compaction based on key
        value: JSON.stringify(userData),
        headers: {
          "event-type": "USER_SNAPSHOT",
          timestamp: Date.now().toString(),
        },
      },
    ],
  });
}

// Call this on every user update
async function updateUser(userId: string, updates: Partial<User>) {
  const user = await getUser(userId);
  const updatedUser = { ...user, ...updates };

  // Write to Redis
  await redis.hset(`user:${userId}`, updatedUser);

  // Publish event to standard topic (for event sourcing)
  await publishEvent("users.events", { type: "USER_UPDATED", userId, updates });

  // Publish snapshot to compacted topic (for state recovery)
  await publishUserSnapshot(userId, updatedUser);
}
```

---

## **Appendix D: Snapshot Automation Example**

**Snapshotter service (pseudo-code)**:

```typescript
class RedisSnapshotter {
  async createAndUploadSnapshot() {
    const timestamp = new Date().toISOString();
    const filename = `redis-snapshot-${timestamp}.rdb`;

    try {
      // Trigger Redis BGSAVE
      logger.info("Starting Redis snapshot");
      await redis.bgsave();

      // Wait for BGSAVE to complete
      await this.waitForSnapshotComplete();

      // Copy RDB file
      const rdbPath = "/var/lib/redis/dump.rdb";
      const localPath = `/tmp/${filename}`;
      await fs.copyFile(rdbPath, localPath);

      // Upload to S3
      logger.info("Uploading snapshot to S3", { filename });
      const s3Key = `redis-snapshots/${filename}`;
      await s3Client
        .upload({
          Bucket: "colist-backups",
          Key: s3Key,
          Body: fs.createReadStream(localPath),
          ServerSideEncryption: "AES256",
        })
        .promise();

      // Store metadata in control topic
      await publishSnapshotMetadata({
        filename,
        s3Key,
        timestamp,
        size: (await fs.stat(localPath)).size,
        redisVersion: await redis.info("server"),
      });

      logger.info("Snapshot completed successfully", { filename, s3Key });

      // Cleanup local file
      await fs.unlink(localPath);

      // Update metrics
      metrics.recordSnapshotSuccess(timestamp);
    } catch (error) {
      logger.error("Snapshot failed", { error });
      metrics.recordSnapshotFailure();
      throw error;
    }
  }

  async waitForSnapshotComplete(maxWaitSec = 300) {
    const startTime = Date.now();
    while (Date.now() - startTime < maxWaitSec * 1000) {
      const info = await redis.info("persistence");
      if (info.includes("rdb_bgsave_in_progress:0")) {
        return; // BGSAVE completed
      }
      await new Promise((resolve) => setTimeout(resolve, 1000)); // Wait 1s
    }
    throw new Error("BGSAVE did not complete in time");
  }
}

// Schedule: every 12 hours
setInterval(() => {
  snapshotter.createAndUploadSnapshot().catch((err) => {
    logger.error("Scheduled snapshot failed", { error: err });
  });
}, 12 * 60 * 60 * 1000);
```
