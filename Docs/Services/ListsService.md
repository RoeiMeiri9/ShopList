# List Service

- **Responsibilities:** lists CRUD, item CRUD, list membership
- **Reads from:** Redis cache
- **Writes to:** Redis; emits `list.*` Kafka events
- **Special rules:** item versioning; if incoming item.version <= cached.version => create new item instance (conflict branch)
- **Notes:** For Share List, operations publish events and expect ChatService/WebSocketService to consume idempotently.
