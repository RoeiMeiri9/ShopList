# ADR-03 — Inter-service comms via Kafka

## Decision

All cross-service mutations are followed by Kafka events. Services consume topics relevant to them.

## Context

Loose coupling, replayability, and failure retry.

## Consequences

- Consumers must be idempotent.
- Topic naming and schema registry required.
