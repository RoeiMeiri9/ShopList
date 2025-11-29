# CoList — Architecture (one page)

## Purpose

Short reference describing main components, consistency model and operational principles.

## Components

- **Realtime store:** Redis — authoritative for runtime state (presence, unread counts, cached lists/items).
- **Durable store:** Relational/Document DB — long-term persistence via DB Writer Service.
- **Event bus:** Kafka — durable event log for inter-service communication, replay and analytics.
- **Transport:** REST for request/response; WebSockets for realtime collaboration.

## Consistency Model

- CoList operates under **Eventual Consistency**.
- Redis is the active runtime store; writes to Redis are followed by Kafka events that eventually persist to DB via DB Writer.
- UI must handle out-of-order events and present optimistic updates.

## Idempotency & Delivery

- Kafka consumers run under **at-least-once** semantics.
- All Kafka-driven handlers are **idempotent**, using dedupe keys `(entity, id, event_id)` stored in Redis/DB.
- Clients use hybrid message id flow: `client_temp_id` + server canonical `message_id`.

## Resilience & Operational Principles

- **WS Gateway:** heartbeats, per-connection and per-room rate-limits (token-bucket), session counters stored in Redis.
- **Redis Recovery:** DB Writer or Kafka replay rebuilds Redis; document runbook for recovery.
- **Monitoring:** any large version conflicts or repeated low-version drops trigger alerts.
- **UX during recovery:** show non-intrusive banner when system is in recovery mode; consider read-only for affected resources.

## Service Ownership

- Each service owns its Redis keys and Kafka topics that it produces.
- Cross-service coordination is through Kafka events only; avoid direct cross-service sync calls.

## Operational Notes

- Limit per-room message rates to protect real-time delivery.
- Keep ADRs small and authoritative (see /docs/adrs).
