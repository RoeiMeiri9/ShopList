# ADR-02 – Message IDs

## Decision

Clients generate `message_id` (UUIDv4) for messages; server validates and persists.

## Context

Enables client-side optimistic display and idempotency when re-sending after retry.

## Consequences

- Server enforces uniqueness per chat.
- duplicate `message_id` = idempotent no-op.
