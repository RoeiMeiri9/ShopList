# Chat Service

### Responsibilities

- Store chat messages in Redis.
- Maintain unread/received/seen counters.
- Broadcast real-time updates via WebSocket.

### Reads from

- Redis

### Writes to

- Redis
- Kafka (for DB Writer + cross-service sync)

### Kafka topics produced

- `chat.created`
- `chat.deleted`
- `chat.message.created`
- `chat.message.edited`
- `chat.message.deleted`

### Message IDs

- Client sends a temporary message ID.
- Chat Service generates the authoritative message ID (UUID or Redis INCR).
- Server returns a mapping `{ temp_id → real_id }`.
- Server applies deduplication based on `(chat_id, temp_id, sender_id)` with a short TTL.

### Seen / Received

- Each event updates Redis counters atomically.
- Kafka events notify DB Writer and analytics consumers.

### Guarantees

- All handlers are idempotent (safe to replay).
- Message creation/edit/delete is atomic at Redis level.

### Failure behavior

- Non-blocking: API never waits for Kafka or slow downstream services.
- Circuit breaker: open after X consecutive downstream errors; half-open after timeout.
- Retry policy: background retry with exponential backoff for Kafka emits.
- Fallback: serve last-known values from Redis.
- Observability: emit metrics on fallback activation, retry attempts, and breaker state.
