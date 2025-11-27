# ADR-01 – Source of Truth

## Decision

Redis is the authoritative real-time store. DB is durable eventual store populated by DB Writer.

## Context

Low-latency collaboration, presence and counters require in-memory store.

## Consequences

- DB Writer must be careful not to overwrite fresher Redis state.
- Recovery process required when Redis restarts.
