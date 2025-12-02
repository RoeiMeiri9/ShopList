## **ADR-10 — Linear Tracking of Unfinished Tasks/Processes**

### **Status**

Accepted

### **Context**

CoList is an event-driven system using Kafka for messaging. Events describe state transitions of tasks or processes:

- `TaskCreated` / `ProcessStarted`
- `TaskProcessingStarted` / `ProcessInProgress`
- `TaskProcessingSucceeded` / `ProcessCompleted`
- `TaskProcessingFailed`

Requirement: **identify all tasks/processes that started but have not completed**, potentially for monitoring, retries, or alerts.

Challenges:

1. Kafka topics contain a mix of unrelated events.
2. Naive approach (collect all IDs, then check for completion) is **O(n²)** — not scalable.
3. Need **linear-time solution** to handle high-throughput streams.

---

### **Decision**

Adopt a **materialized projection/state map** pattern:

1. Maintain an **in-memory map / hash table** keyed by `taskId` / `processId`.

2. Iterate events **once**, updating state inline:

   - On “Started” event → mark `state = STARTED`
   - On “Succeeded/Completed/Failed” event → mark `state = DONE`

3. After processing, tasks with `state = STARTED` are **unfinished**.

4. **Reusability:** The solution should be implemented as a reusable library in an abstract way, allowing it to support a wide range of services.

---

### **Rationale**

- Linear-time (O(n)) iteration avoids nested loops.
- Projection map allows **immediate retrieval of unfinished tasks**.
- Supports **replay** of Kafka topics to reconstruct state at any moment.
- Works even if events are interleaved with unrelated events.
- Memory usage scales with **active tasks only**, not total events.

---

### **Example Implementation (Python-like pseudocode)**

```python
state_map = {}

for event in kafka_events:  # events may come from Kafka partition stream
    task_id = event["taskId"]
    if event["eventType"] in ("TaskCreated", "TaskProcessingStarted"):
        state_map[task_id] = "started"
    elif event["eventType"] in ("TaskProcessingSucceeded", "TaskProcessingFailed"):
        state_map[task_id] = "done"

unfinished_tasks = [tid for tid, status in state_map.items() if status == "started"]
```

- `unfinished_tasks` now contains all tasks/processes that started but never reached a terminal state.

---

### **Consequences**

**Positive:**

- Linear scan → high performance on large streams
- Enables accurate, up-to-date view of unfinished tasks
- Compatible with replay and recovery scenarios
- Idempotent — multiple reads of the same event do not break correctness

**Negative:**

- Memory usage grows with number of concurrent active tasks
- Requires consumer/application to maintain projection
- Needs careful TTL or pruning for long-running streams

---

### **Alternatives Considered**

1. **Naive O(n²) approach:** Rejected — not scalable
2. **Updating status inside raw Kafka events:** Rejected — violates append-only principle, breaks replay
3. **Using external DB for each update:** Possible but adds latency and complexity; still requires projection for performance

---

### **Implementation Notes**

- This pattern forms the **backbone of monitoring and alerting** for unfinished tasks.
- Supports Kafka replay: projection can rebuild current state at any time.
- Can be extended to **distributed consumers** by partitioning state map by `taskId` hash.
