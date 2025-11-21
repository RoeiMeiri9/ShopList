**Flow:**

```
User adds item → Kafka ✅ + Redis ✅ → Instant response
                   ↓
              DB Writer (30 seconds later)
                   ↓
              PostgreSQL ✅
```
