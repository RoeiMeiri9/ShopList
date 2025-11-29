# DB Writer Service

> For every DB in PostgreSQL, there is a separate type of DB Writer Service.
>
> For brevity, we will consider them all as one service named "DB Writer" and not "Chat DB Writer", as they all serve the same role, and the only difference is the DBs they write to.
>
> All of the DB Writers are following the policies below

### Responsibilities

- Persist authoritative state (from Kafka) into the relational DB.
- Ensure durable, ordered, idempotent writes for all entities.
- On Redis reset, repopulate Redis by replaying Kafka topics.
- Serve as persistent history storage for analytics and long-term queries.

### Reads from

- Kafka (all events emitted from services)

### Writes to

- Database (primary durable store)
- Redis (only during recovery / warm-up by replaying Kafka)

### Policy

- Process Kafka events in small batches (every few seconds or size-based).
- Idempotent writes enforced by entity version numbers or upsert logic.
- Ordering guaranteed by Kafka partitioning and version numbers.

### Guarantees

- Safe to replay: events can be reprocessed without corrupting DB.
- No business logic: DBWriter does not modify fields; it only persists them.
- Kafka is the source of truth for recovery, not the DB.

### Failure behavior

- Non-blocking: DBWriter failure does not affect real-time app flows.
- Back pressure: stop consuming Kafka during DB outage (offset pause).
- Retry: exponential backoff for DB writes.
- Circuit breaker: open after X write failures; half-open after cooldown.
- Fallback: rely on Redis for real-time correctness until DB recovers.
- Observability: metrics for write failures, retry attempts, breaker state, lag.

### Recovery

- On startup:
  1. Load last committed Kafka offsets from durable storage.
  2. Replay messages from Kafka to rebuild Redis (warm cache).
  3. Continue normal consumption and persistence.
