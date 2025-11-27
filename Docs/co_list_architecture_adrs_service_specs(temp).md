# CoList — Architecture, ADRs & Service Specs

> This canvas contains a compact set of canonical docs derived from [_Flows.md_](../Flows/Flows.md).
> Use these as the "single source of design decisions" and quick service specs. Keep Flows.md focused on scenarios and reference ADRs when needed.

---

## Repository structure (suggested)

```
/docs
  /architecture
    Architecture.md
  /adrs
    ADR-01-Source-of-Truth.md
    ADR-02-Message-IDs.md
    ADR-03-InterService-Kafka.md
    ADR-04-Eventual-Consistency.md
    ADR-05-Idempotency.md
    ADR-06-WebSocket-Resilience.md
    ADR-07-Redis-Recovery.md
    ADR-08-User-Session-Model.md
  /services
    UsersService.md
    ListsService.md
    ChatService.md
    WebSocketService.md
    DBWriterService.md
Flows.md [kept short, references ADRs]
```

---

# Architecture.md (short — 1 page)

**Purpose:** short, opinionated summary a new engineer can read in ~5 minutes.

**High-level:**

- **Realtime store:** Redis — authoritative for real-time state (presence, unread counters, list membership, items cache).
- **Durable store:** Relational/Document DB — long-term persistence (via DB Writer Service).
- **Event bus:** Kafka — reliable event log for async inter-service communication and replay.
- **Transport:** REST for request/response; WebSockets for realtime events and collaboration.

**Consistency model:** Eventual consistency. Redis is the active runtime state; DB is eventually consistent with Redis (DB Writer writes batches). UI must be prepared for eventual reordering/out-of-order events.

**Resilience/principles:**

- Idempotency for all cross-service operations triggered by Kafka.
- Services should prefer _at-least-once_ Kafka consumption with idempotency on the consumer side.
- Services own their keys in Redis; cross-service coordination via Kafka events.

**Operational notes:** rate limits on WS, heartbeats, graceful degradation when Redis not available.

---

# ADRs (Architecture Decision Records)

> Each ADR is short: Decision, Context, Consequences.

## ADR-01 — Source of Truth

**Decision:** Redis is the authoritative real-time store. DB is durable eventual store populated by DB Writer.
**Context:** Low-latency collaboration, presence and counters require in-memory store.
**Consequences:** DB Writer must be careful not to overwrite fresher Redis state. Recovery process required when Redis restarts.

## ADR-02 — Message IDs

**Decision:** Clients generate `message_id` (UUIDv4) for messages; server validates and persists.
**Context:** Enables client-side optimistic display and idempotency when re-sending after retry.
**Consequences:** Server enforces uniqueness per chat; duplicate message_id = idempotent no-op.

## ADR-03 — Inter-service comms via Kafka

**Decision:** All cross-service mutations are followed by Kafka events. Services consume topics relevant to them.
**Context:** Loose coupling, replayability, and failure retry.
**Consequences:** Consumers must be idempotent; topic naming and schema registry required.

## ADR-04 — Eventual Consistency & Versioning

**Decision:** Use version numbers on mutable entities (items, lists). Higher version wins; on conflict apply CRDT-like simple rule: keep both if version decreased for traceability and surface to UI.
**Context:** Concurrent edits expected.
**Consequences:** UI shows conflict state; background merge policies can be added later.

## ADR-05 — Idempotency

**Decision:** All Kafka-driven side-effects must be idempotent. Use `(entity_type, entity_id, event_id)` dedupe keys in Redis and DB Writer.
**Context:** Kafka at-least-once semantics plus retries.
**Consequences:** Consumers must check dedupe key before applying change.

## ADR-06 — WebSocket Resilience

**Decision:** WS gateway enforces heartbeats, per-connection rate-limits, and session counters. Online status = `active_sessions_count > 0`.
**Context:** Users may have multiple tabs/devices. Chat scale requires backpressure.
**Consequences:** Implement leaky-bucket or token-bucket per room and per-connection limits.

## ADR-07 — Redis Recovery

**Decision:** On Redis restart, DB Writer must replay latest events (from Kafka or DB snapshot) to reconstruct Redis. A short maintenance window may be required.
**Context:** Redis ephemeral; when cleared, some state must be rebuilt.
**Consequences:** Document recovery runbook; mark transient inconsistency to users (banner) if recovery ongoing.

## ADR-08 — User Session Model

**Decision:** Track per-user session entries (`user:{user_id}:sessions`) with TTL. A user considered online when sessions count >= 1.
**Context:** Multiple tabs/devices should report presence independently.
**Consequences:** Logout removes session entry; abrupt disconnect expires TTL after heartbeat window.

---

# Service Spec templates (short)

## UsersService.md (template)

- **Responsibilities:** user profile, settings, session list, presence flags
- **Reads from:** Redis (profile cache), Auth0 for identity authoritative fields
- **Writes to:** Redis; emits `user.updated` events to Kafka
- **Kafka topics produced:** `user.setting.updated`, `user.removed`
- **Guarantees:** Idempotent update handlers; settings not broadcast over WS except via events.
- **Failure behavior:** retry writes via Kafka; do not block main request path on slow external calls to Auth0.

## ListsService.md (template)

- **Responsibilities:** lists CRUD, item CRUD, list membership
- **Reads from:** Redis cache
- **Writes to:** Redis; emits `list.*` Kafka events
- **Special rules:** item versioning; if incoming item.version <= cached.version => create new item instance (conflict branch)
- **Notes:** For Share List, operations publish events and expect ChatService/WebSocketService to consume idempotently.

## ChatService.md (template)

- **Responsibilities:** chat messages store in Redis, unread counters
- **Reads from:** Redis
- **Writes to:** Redis; emits `chat.message.created`, `chat.message.edited`
- **Message IDs:** accept client UUIDs; apply de-duplication
- **Seen/Received:** WS sends `seen` and `received` events that update counters in Redis; Kafka informs DB Writer.

## WebSocketService.md (template)

- **Responsibilities:** gateway for WS traffic, broadcast to rooms, forward pings/seen/received to message services
- **Features:** heartbeats, per-connection rate limit, resume token (optional)
- **Writes to:** Redis (connection/session entries), Kafka events for cross-service consumption

## DBWriterService.md (template)

- **Responsibilities:** durable persistence from Redis/Kafka to DB
- **Policy:** batch every 30s (configurable), idempotent writes, ordering by entity version
- **Failure handling:** on DB down, keep writes in local buffer + Kafka topic for replay
- **Recovery:** on startup, either replay Kafka or read DB snapshot to reconcile Redis (ADR-07)

---

# Flows.md — refactor notes (what I did / what to do)

- Keep Flows.md as the scenario list (unchanged semantics) but _remove repeated infra details_.
- At the end of each flow add `Notes: references ADR-XX` where relevant.
- Split `Remove User` into orchestration flow + subflows: `RemoveUser-Lists`, `RemoveUser-Chats`, `RemoveUser-Auth`.

---

# Next steps (practical)

1. Merge these ADR files to `/docs/adrs/` and commit.
2. Move the short service specs to `/docs/services/`.
3. Edit `Flows.md` to reference ADRs instead of repeating infra text.
4. Add a short `README` at `/docs/` explaining how to add a new ADR.

---

# Quick maintenance commands (suggested)

- Add ADR: `docs/adrs/ADR-09-Name.md` with `Decision / Context / Consequences`.
- Update Flows: find repeated infra items and replace with `Notes: See ADR-03`.
