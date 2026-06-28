# Non-Functional Requirements — Token Bucket Rate Limiter Service

## 1. Performance

### NFR-1: Decision Latency
| Percentile | Target | Degraded |
|---|---|---|
| p50 | < 1ms | < 5ms |
| p95 | < 3ms | < 15ms |
| p99 | < 5ms | < 50ms |
| p99.9 | < 20ms | < 100ms |

Latency is measured from the moment the rate limiter receives a decision request to the moment it returns a response. Network round-trip between the caller and the rate limiter is excluded.

### NFR-2: Throughput
- A single rate limiter node MUST handle > 100,000 decisions/second.
- The service MUST scale linearly with additional nodes (by partitioning keyspace).
- Peak throughput target: 5 million decisions/second globally.

### NFR-3: Tail Latency
The service MUST employ techniques to control tail latency:
- **Hedged requests** to replicas when p99 exceeds threshold.
- **Request coalescing** for duplicate concurrent requests to the same bucket.
- **Client-side timeout** with fast failure path (< 10ms hard cap per decision).

## 2. Availability

### NFR-4: Service Availability
| Tier | Target | Calculation |
|---|---|---|
| Control plane (rule config) | 99.99% | ~52 min downtime/year |
| Data plane (rate decisions) | 99.999% | ~5 min downtime/year |

### NFR-5: Degradation Mode
If the data store (Redis) is unreachable:
- **Fail-open** (allow all requests) for non-critical paths.
- **Fail-close** (reject all requests) only for security-critical limits (e.g., login rate limiting).
- **Local fallback** using an in-memory token bucket with best-effort accuracy.

### NFR-6: Recovery Time Objective (RTO)
- Data plane: < 10 seconds (automatic failover to standby).
- Control plane: < 5 minutes (full rule store recovery).

### NFR-7: Recovery Point Objective (RPO)
- Token state: < 1 second (near-real-time replication).
- Rule configuration: < 5 seconds (WAL-based PostgreSQL replication).

## 3. Consistency

### NFR-8: Per-Key Strong Consistency
For any single bucket key, the service MUST guarantee that two concurrent requests see a consistent view of token availability. This prevents over-consumption when two requests arrive simultaneously.

### NFR-9: Cross-Region Eventual Consistency
Global rate limit state (e.g., user total across all regions) MAY be eventually consistent with a maximum staleness of 2 seconds.

### NFR-10: Configuration Strong Consistency
Rate limit rule changes MUST be strongly consistent. Once a rule update is acknowledged, all subsequent rate decisions MUST use the new rule.

## 4. Durability

### NFR-11: Token State Durability
Token bucket state is ephemeral. Loss of in-memory/Redis state results in reset buckets, which is acceptable (throttled clients regain full capacity). However:
- Rate limit **audit logs** MUST be durably persisted.
- Rate limit **usage reports** for billing MUST be backed by a durable store.

### NFR-12: Configuration Durability
Rate limit rules MUST be stored in a durable, ACID-compliant database (PostgreSQL). Configuration loss is unacceptable.

## 5. Scalability

### NFR-13: Horizontal Scaling
All components of the rate limiter MUST support horizontal scaling by partitioning the keyspace. Adding nodes must increase throughput linearly without requiring a service restart.

### NFR-14: Regional Isolation
Rate limit decisions in one region MUST NOT depend on the availability of another region for normal operation.

### NFR-15: Configuration Scale
The system MUST support at least 100,000 distinct rate limit rules and 10 million active bucket keys.

## 6. Security

### NFR-16: Authentication
All administrative API calls MUST be authenticated via mTLS or OAuth 2.0 (machine-to-machine tokens).

### NFR-17: Authorization
Rate limit rule management MUST support role-based access control (RBAC) with at minimum: Admin, Editor, Viewer roles.

### NFR-18: Audit Trail
All configuration changes MUST be logged with: who, what, when, previous value, new value, and source IP.

## 7. Operability

### NFR-19: Observability
The service MUST expose:
- **Metrics** (Prometheus format): decision count, latency histograms, error rates, cache hit ratio.
- **Structured logs** (JSON): every decision and configuration change.
- **Distributed traces** (OpenTelemetry): end-to-end decision propagation.

### NFR-20: Deployment
- Zero-downtime deployments (rolling update).
- Canary deployment with automatic rollback on error rate increase > 1%.
- Configuration changes deployable without restarting the service.
