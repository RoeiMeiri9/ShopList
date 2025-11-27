# ADR-07 — Redis Recovery

## Decision

On Redis restart, DB Writer must replay latest events (from Kafka or DB snapshot) to reconstruct Redis. A short maintenance window may be required.

## Context

Redis ephemeral; when cleared, some state must be rebuilt.

## Consequences

- Document recovery runbook
- Mark transient inconsistency to users (banner) if recovery ongoing
