# UserService

### Responsibilities

- user profile,
- settings,
- session list,
- presence flags

### Reads from

- Redis (profile cache)
- Auth0 for identity authoritative fields

### Writes to

- Redis
- Emits events to Kafka

### Kafka topics produced

- `user.setting.updated`
- `user.removed`

### Guarantees

- Idempotent update handlers
- Settings not broadcast over WS except via events.

### Failure behavior

- Non-blocking: avoid blocking API responses on slow third-parties (Auth0). Use background retry for external updates.
- Circuit breaker: if Auth0 is failing, flip to degraded mode: mark fields read-only, queue changes in Kafka for later sync.
- Retry policy: exponential backoff for retries to Auth0 (max 5 attempts).
- Fallback: serve cached values from Redis; if missing, return minimal profile and a flag 'profile_partial=true'.
- Observability: emit metric when fallback used.
