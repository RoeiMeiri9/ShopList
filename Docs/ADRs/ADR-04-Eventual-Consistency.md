# ADR-04 — Eventual Consistency & Versioning

## Decision

Use version numbers on mutable entities (items, lists). Higher version wins; on conflict apply simple rule: if it can't be duplicated, drop and raise a monitoring alert.

## Context

Concurrent edits expected.

## Consequences

- UI shows conflict state.
- Background merge policies can be added later.
