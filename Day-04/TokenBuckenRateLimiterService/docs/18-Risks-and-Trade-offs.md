# Risks & Trade-offs — Token Bucket Rate Limiter Service

## 1. Architectural Trade-offs

### 1.1 Centralized vs. Distributed Rate Limiting

| Approach | Selected? | Rationale |
|---|---|---|
| **Centralized** (single Redis-backed service) | ✅ Selected | Strong per-key consistency. Simple operational model. Well-understood scaling properties. |
| **Distributed** (client-side token buckets per node) | ❌ Rejected | No global coordination. Inconsistent enforcement across nodes. Hard to debug. |

**Risk:** Centralized Redis becomes a single point of failure and a throughput bottleneck.

**Mitigation:** Redis Cluster with sharding + automatic failover. Local fallback when Redis is unreachable. The decision to trade availability for consistency is explicit and documented.

### 1.2 Redis vs. Other Stores for Token State

| Store | Latency (p99) | Atomic Ops | Persistence | Verdict |
|---|---|---|---|---|
| **Redis (in-memory)** | < 1ms | ✅ Lua scripts | ❌ Ephemeral (RPO < 1s) | ✅ Selected |
| **PostgreSQL** | 5–20ms | ✅ Transactions | ✅ Durable | ❌ Too slow |
| **Dragonfly** | < 1ms | ✅ Lua-compatible | ✅ Snapshot | 🔄 Worth evaluating |
| **Local memory** | < 10µs | ❌ No sharing | ❌ Lost on restart | ❌ Used only as fallback |
| **Cassandra/ScyllaDB** | 5–15ms | ❌ LWT expensive | ✅ Durable | ❌ Too slow |

**Trade-off:** Redis gives us speed at the cost of durability. We accept up to 1 second of token state loss. This is a deliberate choice: losing token state means throttled clients regain capacity, which is preferable to the latency or consistency problems of alternatives.

### 1.3 Lua Scripts vs. Optimistic Locking

| Approach | Pros | Cons | Verdict |
|---|---|---|---|
| **Lua script (EVAL)** | Atomic, single round-trip | Lua complexity, Redis CPU cost | ✅ Selected |
| **WATCH/MULTI/EXEC** | No Lua dependency | High contention → retries → latency spikes | ❌ Rejected |
| **Redis functions** | Versioned, reusable | Newer feature, less tooling support | 🔄 Future migration |

**Trade-off:** Lua scripts consume Redis CPU during execution. At very high throughput (> 500K dec/s per Redis node), the Lua script execution time becomes a bottleneck. Mitigation: optimize the script to minimize computation (no string concatenation, no loops over large data), and add more Redis shards to distribute the CPU load.

## 2. Consistency Trade-offs

### 2.1 Strong vs. Eventual Consistency for Global Limits

**Problem:** A user makes 100 requests to us-east-1 and 100 requests to eu-west-1 simultaneously. The global limit is 150 requests/hour. Should both regions allow the first 75 requests each, or should us-east-1 allow 100 and eu-west-1 only allow 50?

**Chosen approach:** Per-region local limits + periodic global reconciliation.

**Trade-off:** Accept up to 5 seconds of inconsistency. A user could exceed their global limit by up to `regions × seconds × rate` before the global ledger catches up.

**Why this is acceptable:**
- Almost all rate limits are per-region (enforced locally, strong consistency).
- Global limits are typically for billing/caps, where occasional overage of 1-2% is tolerable.
- The complexity of strong-consistency global coordination (2PC, Paxos, consensus) is not justified for the use case.

### 2.2 Configuration vs. Decision Consistency

**Design choice:** The data plane caches rules with a 5-second TTL. If a rule is updated, there is a 0–5 second window where the old rule is still enforced.

**Trade-off:** A newly created rule may take up to 5 seconds to take effect across all nodes. This is acceptable because:
- Rule updates are not time-critical (they respond to observed traffic patterns, not instantaneous events).
- A 5-second delay is imperceptible for manual rule changes.
- Emergency overrides use a separate fast-path (Redis Pub/Sub with immediate invalidation, ~100ms propagation).

## 3. Operational Risks

### 3.1 Redis Memory Exhaustion

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Redis runs out of memory | Low (monitored) | Evictions cause token state loss | Alert at 70% memory. Add shards. Configure `maxmemory-policy allkeys-lru` for graceful degradation. |

### 3.2 Lua Script Performance Degradation

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Lua script becomes slow under load | Medium | Increased decision latency | Monitor `redis_cpu` and `rate_limiter_decision_latency`. Profile Lua script for optimization. Add Redis shards. |

### 3.3 Configuration Drift

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Rule cache on some nodes becomes stale beyond 5s | Low | Some nodes enforce stale rules | Polling backup + Pub/Sub invalidation. Alert on cache staleness > 10s. |

### 3.4 Clock Skew

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| RL node and Redis server clocks drift > 100ms | Medium | Token refill timing errors | All time is sourced from Redis TIME command. RL node clock is not used for refill calculations. |

## 4. Security Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Unauthorized rule modification | Low | Attacker could disable rate limiting | mTLS + RBAC + audit logging + anomaly detection on rule changes |
| Rate limiter bypass | Medium | Attacker discovers key pattern and uses a different key | Hash-based key construction; IP-based fallback limits |
| DDoS against the rate limiter | Low | Rate limiter becomes the bottleneck | Self-protection rate limits + API Gateway-level DDoS protection |
| Side-channel via decision timing | Low | Attacker infers token count from response timing | Constant-time response generation (always wait for Redis, even on fast path) |

## 5. Business Risks

| Risk | Impact | Mitigation |
|---|---|---|
| **Overly aggressive limits block revenue** | Legitimate customers unable to use the API | Dry-run mode for new rules. Conservative defaults. Automatic false positive detection. |
| **Overly permissive limits fail to protect** | Downstream systems overwhelmed | Monitoring on downstream system health. Circuit breakers at the gateway layer as a backstop. |
| **Engineering complexity exceeds value** | The rate limiter becomes too complex to maintain | Core functionality delivered in MVP (token bucket + rule CRUD). Advanced features (ML adaptation, global ledger) are optional and independently deployable. |

## 6. Known Design Gaps

### 6.1 Not Yet Addressed

| Gap | Impact | Timeline |
|---|---|---|
| No native WebSocket rate limiting | WebSocket connections are not gated by this service | Q4 2026 |
| No rate limit "credit" system | High-volume bursts from previously underutilized clients are still rejected | Q1 2027 |
| No hierarchical limits (team → service → endpoint) | Complex multi-team rate limit scenarios require manual composition | Q2 2027 |
| No batch decision optimization sending to Redis | Each decision is a separate round-trip to Redis | Future consideration |

### 6.2 Deliberately Out of Scope

| Feature | Reason for Exclusion |
|---|---|
| **API Gateway functionality** | This is a rate limiter, not a full gateway. Routing, auth, and request transformation belong elsewhere. |
| **User-facing quota management** | Quota/resource allocation is a separate domain (provisioning, billing). The rate limiter enforces limits; it does not manage entitlements. |
| **Adaptive rate limiting without human oversight** | ML-based adjustments require a human-in-the-loop for the foreseeable future. Full autonomy is a long-term vision. |
| **Multi-dimensional rate limiting** | Rate limit on (client × endpoint × time_of_day × device_type) is too complex to manage and debug. We limit to 3 dimensions max. |

## 7. Decision Log

| Date | Decision | Rationale | Reconsidered? |
|---|---|---|---|
| 2026-05-01 | Redis over PostgreSQL for token state | Performance requirement (p99 < 5ms) cannot be met by PostgreSQL | No — validated by load tests |
| 2026-05-05 | Lua scripts over optimistic locking | Contention on hot keys caused retry storms in optimistic locking tests | No — Lua consistently faster |
| 2026-05-10 | Sync over async for data plane | Async decision introduces complexity (callback handling, timeout management) without benefit for this use case | No |
| 2026-05-15 | gRPC over REST for data plane | Higher throughput, lower latency, stronger typing | No |
| 2026-05-20 | Fail-open by default for data plane | Preventing legitimate traffic is worse than allowing extra traffic through rate limits | Yes — security-critical limits use fail-close |

## 8. Summary

| Topic | Verdict | Key Trade-off |
|---|---|---|
| **State storage** | Redis (in-memory) | Speed over durability |
| **Atomicity** | Lua scripts | CPU overhead for perfect consistency |
| **Consistency model** | Strong (per-key), Eventual (global) | Accept 5s staleness for global limits |
| **Failure mode** | Fail-open (default), Fail-close (security) | Availability over correctness |
| **Deployment** | Centralized service, not client library | Latency overhead (network round-trip) for operational simplicity |
| **Scaling** | Horizontal + Redis sharding | Cost of additional Redis nodes |
