# ADR-08 — User Session Model

## Decision

Track per-user session entries (`user:{user_id}:sessions`) with TTL. A user considered online when sessions count >= 1.

## Context

Multiple tabs/devices should report presence independently.

## Consequences

- Logout removes session entry
- Abrupt disconnect expires TTL after heartbeat window.
