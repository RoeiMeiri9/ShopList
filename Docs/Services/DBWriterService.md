# DBWriterService

- **Responsibilities:** durable persistence from Redis/Kafka to DB
- **Policy:** batch every 30s (configurable), idempotent writes, ordering by entity version
- **Failure handling:** on DB down, keep writes in local buffer + Kafka topic for replay
- **Recovery:** on startup, either replay Kafka or read DB snapshot to reconcile Redis (ADR-07)
