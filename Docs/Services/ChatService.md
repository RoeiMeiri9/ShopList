# Chat Service

- **Responsibilities:** chat messages store in Redis, unread counters
- **Reads from:** Redis
- **Writes to:** Redis; emits `chat.message.created`, `chat.message.edited`
- **Message IDs:** accept client UUIDs; apply de-duplication
- **Seen/Received:** WS sends `seen` and `received` events that update counters in Redis; Kafka informs DB Writer.
