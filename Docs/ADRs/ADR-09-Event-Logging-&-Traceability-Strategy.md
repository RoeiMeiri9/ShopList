## **ADR-09 — Event Logging & Traceability Strategy**

### **Status**

Proposed / Draft

### **Context**

The system (CoList) is event-driven and integrates Kafka as a messaging backbone.\
To support debugging, monitoring, replay, analytics, and recovery, the platform must be able to trace:

1. What happened
2. When it happened
3. By whom / for whom it happened
4. How it propagates through services

Kafka already captures message flows, but raw messages alone do **not** provide meaning unless enriched with traceable metadata.

A structured logging and trace correlation strategy is required.

---

### **Decision**

We will adopt a **structured event schema** across the platform that ensures:

✔ Each event is identifiable\
✔ Actions can be traced end-to-end\
✔ The data can be exported into ELK / DataDog / etc.\
✔ The system can replay state or audit behavior when needed

We use a **predictable metadata envelope** wrapping event payloads.

Example:

```json
{
  "eventId": "uuid",
  "eventType": "TaskCreated",
  "correlationId": "uuid",
  "timestamp": "ISO-8601",
  "actor": {
    "userId": "uuid",
    "source": "MobileApp"
  },
  "payload": {
    "taskId": "uuid",
    "title": "Fix Kafka Replay Logic"
  }
}
```

Supporting conventions:

#### 1. **eventId**

- Generated per event (UUID)
- Used for deduplication and auditing

#### 2. **correlationId**

- Generated at entry point (API gateway)
- Attached to every resulting event and log
- Allows tracing a scenario across microservices
- **Based on the [flows.md](../Flows/Flows.md) file!**

#### 3. **eventType**

- Canonical name, e.g., `TaskCreated`, `ChatMessageSent`

#### 4. **actor**

- Represents the initiating entity
- Can be user _(phone app / web app are differentiated!)_, automated task scheduler, bot, etc.

#### 5. **timestamp**

- ISO-8601 UTC time when event occurred

#### 6. **payload**

- Business data

A structured JSON like this becomes your **unit of observability**.

---

### **Consequences**

#### ✔ Positive

- Logs exported to tools like ELK/DataDog become human-meaningful visual traces
- Production issues can be debugged via correlation flows
- Kafka replay becomes deterministic because messages are identifiable
- Enables behavioral analytics (how users actually use the system)
- Supports future audit requirements

#### ✖ Tradeoffs

- Slightly larger messages
- Requires standardization across services
- Logging consistency must become part of code review culture

---

### **Notes**

This ADR intentionally focuses on **traceability and meaning**, not performance tuning.

We will:

- Introduce a small platform SDK / utility for generating event envelopes (UUID, timestamps, correlation inheritance)
- Require all services to emit events only through this SDK

---

### **Next Steps**

1. Define canonical event types (domain glossary)
2. Implement shared library:

   - `createEventEnvelope(payload, actor, eventType)`
   - `propagateCorrelationId()`

3. Update logging and Kafka producer layers to enforce envelope usage
4. Configure log shipping to observability platform

---

### **Future Enhancements (Optional Later ADRs)**

- Event-to-trace visualization layer (timeline view)
- Aggregated domain event stats dashboards
- Replay engine feeding Redis restoration from Kafka logs
