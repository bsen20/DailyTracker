# Business Requirements — Token Bucket Rate Limiter Service

## 1. Business Drivers

The organization operates a multi-tenant API platform serving thousands of internal services and external partners. Without a centralized rate limiting capability, the following business risks materialize:

**Revenue leakage.** API-tiered pricing (requests per second/month) requires enforcement at the infrastructure level. Without it, customers can consume beyond their purchased tier without penalty.

**Brand damage.** When one tenant's traffic spike causes degraded performance for all tenants, affected customers lose trust. This directly impacts retention and Net Promoter Score (NPS).

**Operational cost overrun.** Downstream dependencies (e.g., third-party LLM APIs, database connectors, payment gateways) charge per request. Uncontrolled traffic inflates monthly bills unpredictably.

## 2. Stakeholder Requirements

### 2.1 Platform Engineering
- Needs a **unified interface** for defining rate limits across all services.
- Must support **per-tenant, per-endpoint, and global** limit scopes.
- Requires **self-service** configuration without deploying new code.
- Needs **role-based access control** for rule management.

### 2.2 Product Teams
- Can define rate limits for their APIs without infrastructure expertise.
- Receive **real-time feedback** when a limit is approached or breached.
- Can **simulate and test** rate limit configurations in staging.
- Need **granularity** — different limits for authenticated vs. unauthenticated users.

### 2.3 SRE
- Must **monitor rate limit effectiveness** — how often limits are hit, which clients are throttled.
- Needs **alerts** when a single client's throttled rate exceeds a threshold (potential DDoS).
- Requires **audit trail** — who changed which rule and when.
- Must be able to **exempt critical internal services** from certain limits.

### 2.4 Security
- Rate limiting is a **first layer of defense** against credential stuffing and API abuse.
- Must support **adaptive limits** — tighter throttling for suspicious IP ranges or patterns.
- Needs **integration with the threat intelligence pipeline** for dynamic limit adjustments.

### 2.5 Finance / BizOps
- Rate limit usage must be **auditable for billing** purposes.
- Need **reports** on per-tenant request counts and throttling events.
- Must be able to **define plan-level rate limit tiers** (Free: 10 req/s, Pro: 100 req/s, Enterprise: custom).

## 3. Business Constraints

| Constraint | Rationale |
|---|---|
| **No single point of failure** | Rate limiter failure cannot block production traffic |
| **Sub-millisecond overhead** | Added latency must be imperceptible to end users |
| **Existing infrastructure integration** | Must work with current Envoy-based API Gateway and Redis infrastructure |
| **Global deployment** | Service must operate across US, EU, and APAC regions |
| **Compliance** | Rate limit decisions for EU users must stay within EU boundaries (GDPR) |

## 4. Key Business Metrics

| Metric | Why It Matters |
|---|---|
| **Throttled request ratio** | % of requests rejected. Too high = customer friction. Too low = limits ineffective. |
| **Time-to-throttle** | How quickly new rate limit rules take effect after configuration. |
| **Decision accuracy** | False positives (incorrectly rejecting valid traffic) directly impact revenue. |
| **Configuration change frequency** | Indicates how well static limits fit dynamic traffic patterns. |
| **Cross-team rule conflicts** | Overlapping limits between teams create unpredictable behavior. |

## 5. Success Criteria (Business)

1. **Tenant isolation.** A traffic spike from any single tenant never degrades the p99 latency of other tenants by more than 5%.

2. **Cost containment.** Downstream API costs are predictable within ±5% of forecast, with rate limiting as the primary control mechanism.

3. **Self-service adoption.** 80% of product teams configure their own rate limits within the first quarter without requiring Platform Engineering tickets.

4. **Incident reduction.** Rate-limit-related incidents (noisy neighbors, accidental DDoS from buggy clients) decrease by 90% after adoption.
