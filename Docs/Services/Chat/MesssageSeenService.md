# Message Seen Service

### Responsibilities

- Manage the `seen` value for existing chat messages in Redis.
- Maintain seen counters.
- Broadcast real-time updates via WebSocket.

### Reads from

- Redis

### Writes to

- Redis
- Kafka (for DB Writer + cross-service sync)

### Kafka topics produced

- `chat.message.seen`

### Seen / Received

- Front sends `seen` when the user scrolls to read it.
- Once all participants seen the message, server emits:
  - `seen_by_all`
- Kafka events notify DB Writer and analytics consumers.

### Guarantees

- All handlers are idempotent (safe to replay).
- Modify only the `seen` flag.
- Independent from other chat-related services.

### Failure behavior

- Non-blocking: API never waits for Kafka or slow downstream services.
- Circuit breaker: open after X consecutive downstream errors; half-open after timeout.
- Retry policy: background retry with exponential backoff for Kafka emits.
- Fallback: serve last-known values from Redis.
- Observability: emit metrics on fallback activation, retry attempts, and breaker state.
