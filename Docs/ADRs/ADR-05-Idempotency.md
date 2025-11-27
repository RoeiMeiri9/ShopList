# ADR-05 — Idempotency

## Decision

All Kafka-driven side-effects must be idempotent. Use `(entity_type, entity_id, event_id)` dedupe keys in Redis and DB Writer.

## Context

Kafka at-least-once semantics plus retries.

## Consequences

- Consumers must check dedupe key before applying change.
