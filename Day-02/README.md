# Day 02 — System Design

## Problem: Design a Scalable Notifications System

Design a system capable of sending 1 billion notifications per day across push, email, SMS, and in-app channels to 100 million daily active users.

### Key Requirements
- **Write throughput:** 50,000 notifications/second peak
- **Read throughput:** 200,000 reads/second peak (notification history)
- **Latency:** p99 < 500ms for real-time delivery (push, in-app)
- **Delivery semantics:** At-least-once with idempotency-based deduplication
- **Priority:** Transactional (password resets, payments) > Promotional (marketing campaigns)
- **Retention:** 90-day notification history with cursor-based pagination
- **Channels:** Push (APNS/FCM), Email (SendGrid/SES), SMS (Twilio/Vonage), In-app (WebSocket)
- **Regions:** Multi-region (US, EU, APAC) with local data isolation

### Key Decisions
- **Database:** ScyllaDB for notification history (append-heavy time-series, linear write scaling, built-in TTL)
- **Queue:** Kafka with separate topics for high/low priority
- **Real-time:** WebSocket gateway with Redis Pub/Sub fan-out
- **Caching:** Redis for hot reads + idempotency; Bloom filter for cursor validation
- **Resilience:** Circuit breakers per provider, dead letter queues, canary deployments with auto-rollback

---

*Solutions and deep-dive materials can be added below this line.*
