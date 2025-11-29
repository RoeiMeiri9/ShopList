# ADR-02 – Message IDs

## Decision

Clients generate `message_id` (UUIDv4) for messages; server validates and persists.
If `message_id` is not unique in DB, returns a different id for the UI to replace.

## Context

Enables client-side optimistic display and idempotency when re-sending after retry.

## Consequences

- Server enforces uniqueness per chat.
- duplicate `message_id` = replace `message_id` with unique one, response to sender with persisted `message_id`.
