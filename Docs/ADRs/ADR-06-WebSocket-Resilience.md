# ADR-06 — WebSocket Resilience

## Decision

WS gateway enforces heartbeats, per-connection rate-limits, and session counters. Online status = `active_sessions_count > 0`.

## Context

Users may have multiple tabs/devices. Chat scale requires backpressure.

## Consequences

- Implement leaky-bucket or token-bucket per room and per-connection limits.
