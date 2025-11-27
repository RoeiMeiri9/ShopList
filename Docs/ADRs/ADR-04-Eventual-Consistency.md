# ADR-04 — Eventual Consistency & Versioning

## Decision

Use version numbers on mutable entities (items, lists). Higher version wins; on conflict apply CRDT-like simple rule: keep both if version decreased for traceability and surface to UI.

## Context

Concurrent edits expected.

## Consequences

- UI shows conflict state.
- Background merge policies can be added later.
