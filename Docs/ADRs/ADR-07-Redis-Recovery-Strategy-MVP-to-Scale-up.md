### ADR-07 — Redis Recovery Strategy: From MVP to Scale-up

| Parameter         | Value                                                                 |
| :---------------- | :-------------------------------------------------------------------- |
| **Status**        | **Accepted** (Phases and principles)                                  |
| **Decision Type** | Architecture – Resilience / Distributed Systems                       |
| **Applies to**    | All microservices (read/write Redis), DB Writer Service, Kafka, Redis |

---

### 1. Context

The CoList platform utilizes **Redis as the real-time cache/Materialized View** and **Kafka as the authoritative Event Log**. The **DB Writer Service** is the sole component permitted to write to **PostgreSQL**. **Services only write to Redis** and emit Kafka events.

The system faces a critical architectural decision regarding **Redis recovery** after a catastrophic event (e.g., full Redis wipe).

**Current Constraints and Goals:**

- **Data Integrity Rule:** Only the DB Writer Service interacts with the DB. No other service may read from the DB.
- **Current State:** Small scale (Proof of Concept, < 8 services).
- **Future Goal:** Implement a robust, scalable recovery mechanism that avoids the **Thundering Herd** problem when the system scales (8-12+ services with multiple replicas).
- **Mandate:** Design for future scale to demonstrate high architectural awareness.

---

### 2. Decision

We will implement a two-phased recovery strategy that minimizes complexity at the MVP stage while clearly defining the future path to a controlled, centralized recovery mechanism.

#### Phase 1: MVP / Initial Stage (Current Implementation)

Recovery at this stage will be **passive** and relies on existing mechanisms, prioritizing development speed.

1.  **Redis Persistence:** Redis **must** have persistence enabled (AOF and/or RDB) as the first line of defense.
2.  **Recovery:** If a full wipe occurs (assuming AOF/RDB fail or are corrupt), recovery is **manual/organic**.
    - **DB Writer Role:** The DB Writer Service will be manually triggered to perform a **one-time full read from PostgreSQL** and populate the base state into Redis. This is the **only exception** where a service may read the DB, and it must be executed by the authorized component.
    - **Services:** Other services repopulate Redis organically via normal read/write operations (Cache Miss logic) or by consuming recent Kafka events.
3.  **No Coordinator:** A distributed coordinator is **explicitly excluded** at this stage.

#### Phase 2: Scale-Up / Future Architecture (8+ Services)

When the system reaches a size where multiple replicas exist and simultaneous writes could overload Redis, a **Centralized, Coordinated Recovery** mechanism will be implemented.

---

### **Phase 2 Components:**

#### A. **Redis Recovery Coordinator** (The Leader)

A single service instance will be elected (via Kubernetes Lease, Kafka Group Balancing, or similar mechanism). Only the Coordinator is allowed to manage the recovery process:

- Initiate the process and set the recovery flag.
- Direct the DB Writer for base state population (if needed).
- Perform controlled, **throttled replay** of Kafka events to bring Redis up to date.

#### B. **Global Recovery Flag**

A globally accessible flag (`redis_recovery_in_progress`) stored in a Kafka control topic or ConfigMap.

- **When `true`:** All non-Coordinator services **MUST STOP** writing directly to Redis. All intended writes must be published **ONLY** to Kafka.
- **When `false`:** Services resume normal writes to Redis (and continue publishing to Kafka).

#### C. **Recovery Service Flow (DB Writer)**

The existing DB Writer Service will be granted a specific, audited recovery mode role.

1.  **Base State Restore:** The Coordinator directs the DB Writer to **read the authoritative state from PostgreSQL** and populate the Redis base key-set.
2.  **Consistency Guarantee:** This mechanism guarantees that even if Kafka's retention has expired for old events, the recovery can start from the **authoritative state** in the DB, maintaining the integrity rule.

#### D. **Controlled Kafka Replay**

Following the base state restore, the Coordinator will perform a **throttled replay of recent Kafka events** that occurred after the DB snapshot (or since the last stable offset) to catch up on real-time changes (e.g., chat messages, counter updates) that may not be fully represented in the DB Writer's snapshot.

---

### 3. Rationale

| Phase                  | Benefit                                                                                                                                                 | Rationale for Postponement                                                                                    |
| :--------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------ | :------------------------------------------------------------------------------------------------------------ |
| **Phase 1 (MVP)**      | **Fast Development:** Avoids implementing complex distributed logic (Leader Election, Global Flag).                                                     | Complexity is **not needed** at small scale. The risk of **Thundering Herd** is negligible with few replicas. |
| **Phase 2 (Scale-Up)** | **Thundering Herd Prevention:** Centralized control prevents many services from writing simultaneously, ensuring stability.                             | The system needs resilience only when the number of writers (replicas) is high enough to cause overload.      |
| **DB Writer Role**     | **Integrity Preservation:** Ensures the DB remains untouched by regular services, as the recovery read is delegated to the single authorized component. | This decision upholds the core architectural invariant.                                                       |

---

### 4. Consequences

| Positive                                                                                                   | Negative                                                                                                                                      |
| :--------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------- |
| **Future Proof:** The architecture is designed for scale, demonstrating advanced system design principles. | **System-wide Refactoring:** Implementing Phase 2 requires modifying the write path of **every service** to respect the global recovery flag. |
| **Deterministic Recovery:** Recovery is sequential, controlled, and verifiable.                            | **Increased Latency during Recovery:** The system operates in degraded mode (read-only from Redis) until the Coordinator finishes the replay. |

---

### 5. Implementation Plan

| Phase                   | Action Items (MVP to Scale)                                                                                                                                                                                       | Status   |
| :---------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------- |
| **Phase 0 (Today)**     | **1.** Enable Redis Persistence (AOF/RDB). **2.** Ensure all services publish all writes to Kafka. **3.** Implement runbook for manual DB Writer full-restore.                                                    | Complete |
| **Phase 1 (Pre-Scale)** | **1.** Design and implement the global `redis_recovery_in_progress` flag mechanism (e.g., ConfigMap). **2.** Implement the flag check in **ALL** services' write path: If `true`, write to Kafka only.            | Planned  |
| **Phase 2 (Scale-Up)**  | **1.** Implement Leader Election (Coordinator). **2.** Implement Coordinator logic for recovery orchestration (direct DB Writer, manage replay). **3.** Implement Kafka Replay logic (throttling, checkpointing). | Planned  |
