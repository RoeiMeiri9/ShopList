## Overview

- **Realtime store:** Redis — authoritative for real-time state (presence, unread counters, list membership, items cache).
- **Durable store:** Relational/Document DB — long-term persistence (via DB Writer Service).
- **Event bus:** Kafka — reliable event log for async inter-service communication and replay.
- **Transport:** REST for request/response; WebSockets for realtime events and collaboration.

**Consistency model:** Eventual consistency. Redis is the active runtime state; DB is eventually consistent with Redis (DB Writer writes batches). UI must be prepared for eventual reordering/out-of-order events.

**Resilience/principles:**

- Idempotency for all cross-service operations triggered by Kafka.
- Services should prefer _at-least-once_ Kafka consumption with idempotency on the consumer side.
- Services own their keys in Redis; cross-service coordination via Kafka events.

**Operational notes:** rate limits on WS, heartbeats, graceful degradation when Redis not available.
