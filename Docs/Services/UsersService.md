# UserService

- **Responsibilities:** user profile, settings, session list, presence flags
- **Reads from:** Redis (profile cache), Auth0 for identity authoritative fields
- **Writes to:** Redis; emits `user.updated` events to Kafka
- **Kafka topics produced:** `user.setting.updated`, `user.removed`
- **Guarantees:** Idempotent update handlers; settings not broadcast over WS except via events.
- **Failure behavior:** retry writes via Kafka; do not block main request path on slow external calls to Auth0.
