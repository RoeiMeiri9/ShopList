# ADR-01 – Source of Truth

## Decision

Redis is the authoritative real-time state store.

## Context

Redis is fast, supports atomic operations, used by all services.

## Consequences

- Conflicts resolved by Redis version number.
- DB Writer never overwrites Redis.
