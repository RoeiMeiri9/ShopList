# WebSocket Service

- **Responsibilities:** gateway for WS traffic, broadcast to rooms, forward pings/seen/received to message services
- **Features:** heartbeats, per-connection rate limit, resume token (optional)
- **Writes to:** Redis (connection/session entries), Kafka events for cross-service consumption
