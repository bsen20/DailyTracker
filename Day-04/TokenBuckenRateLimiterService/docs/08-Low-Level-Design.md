# Low-Level Design — Token Bucket Rate Limiter Service

## 1. Component Architecture

```mermaid
graph TB
    subgraph "Data Plane Node"
        GRPC["gRPC Server<br/>Decision API"]
        MID["Middleware Layer<br/>Auth, Rate Limit, Timeout"]
        
        subgraph "Core Engine"
            RES["Request Router<br/>Key Extraction + Rule Matching"]
            TBE["Token Bucket Engine<br/>Refill + Consumption Logic"]
            CACHE["Rule Cache<br/>LRU + TTL-based"]
        end
        
        subgraph "Storage Layer"
            RC["Redis Client<br/>Connection Pool<br/>Circuit Breaker"]
            LC["Local Fallback<br/>In-Memory Bucket Store"]
        end
        
        subgraph "Observability"
            MET["Metrics Exporter<br/>Prometheus"]
            LOG["Async Logger<br/>Batch + Buffer"]
            TRACE["Trace Propagator<br/>OpenTelemetry"]
        end
        
        CONN["Connection Manager<br/>gRPC Keep-Alive + Backpressure"]
    end

    subgraph "Control Plane"
        API["REST/gRPC Admin API"]
        VAL["Rule Validator"]
        SIM["Simulation Engine"]
        PUBSUB["Config Pub/Sub Publisher"]
    end

    GRPC --> MID
    MID --> RES
    RES --> TBE
    RES --> CACHE
    TBE --> RC
    TBE --> LC
    RC --> MET
    RC --> LOG
    TBE --> MET
    TBE --> LOG
    MID --> TRACE
    GRPC --> CONN
    CONN --> MET
    API --> VAL
    API --> PUBSUB
    PUBSUB -.-> CACHE
```

## 2. Component Responsibilities

### 2.1 Request Router
- Receives `CheckRequest` from the middleware layer.
- Extracts the bucket key from the request.
- Calls the Rule Cache to find the matching `RateLimitRule`.
- If no rule found, returns **allow** (default behavior).
- Passes the rule parameters + bucket key to the Token Bucket Engine.

### 2.2 Token Bucket Engine
The core decision-making component:

```
Input:  bucket_key, cost, rule(max_tokens, refill_rate, refill_interval, mode)
Output: allowed, remaining_tokens, retry_after

Flow:
1. Construct Redis key: "rl:bucket:{bucket_key}"
2. Execute Lua script with: key, cost, now_ms, refill_rate, refill_interval, max_tokens
3. Parse Lua response: [allowed, remaining, next_refill_ms, retry_after]
4. If Redis call fails → fall back to local in-memory bucket
5. Return result
```

### 2.3 Rule Cache
- Maintains an in-memory cache of all active rate limit rules.
- Cache is populated on startup from PostgreSQL.
- Refreshed every 5 seconds via background goroutine/polling.
- Invalidated immediately on config change via Redis Pub/Sub.
- LRU eviction if memory exceeds threshold.

**Cache entry structure:**
```
Cache Key:   token_bucket_rule:{key_pattern_hash}
Cache Value: { rule_id, max_tokens, refill_rate, refill_interval_ms, 
               request_cost, burst_max, priority, mode, dry_run, exempt_clients }
TTL:         5s (sliding; refreshed on access)
```

### 2.4 Redis Client
- Connection pool with configurable min/max connections.
- Circuit breaker: after 5 consecutive failures within 30s, open circuit for 10s.
- Health check pings every 3 seconds.
- Automatic failover: if primary Redis is unreachable, query replica.
- Timeout: 10ms connect, 5ms operation.

### 2.5 Local Fallback
When Redis is unreachable:
- Maintains a local `sync.Map` (or equivalent) of token bucket states.
- Best-effort accuracy: tokens are tracked locally without persistence.
- Bucket state is lost if the node restarts (acceptable — buckets reset).
- Metrics tagged with `fallback=true` for visibility.

### 2.6 Async Logger
- Buffers decision log entries in a ring buffer.
- Flushes to ClickHouse batch endpoint every 500ms or 10,000 entries.
- If ClickHouse is down, logs are dropped (non-critical path).
- Separate high-priority channel for audit-critical events.

## 3. Request Lifecycle (Detailed)

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as API Gateway
    participant RM as Rate Limiter<br/>Middleware
    participant RR as Request Router
    participant RC as Rule Cache
    participant TBE as Token Bucket<br/>Engine
    participant R as Redis
    participant LF as Local Fallback
    participant O as Observability

    C->>GW: Request
    GW->>RM: Intercept request
    RM->>RM: Extract: client_id=abc, endpoint=/orders
    RM->>RR: Check("client:abc:endpoint:/orders", cost=1)
    
    RR->>RC: GetRule("client:{client_id}:endpoint:{path}")
    RC->>RC: Match key_pattern, check TTL
    
    alt Cache MISS or expired
        RC->>PG: SELECT ... FROM rate_limit_rules WHERE key_pattern matches
        PG-->>RC: Rule found: max_tokens=10, refill_rate=10, interval=1000
        RC->>RC: Cache rule (TTL: 5s)
    end
    
    RC-->>RR: RateLimitRule
    RR->>TBE: Evaluate(bucket_key="client:abc:endpoint:/orders", rule, cost=1)
    
    alt Dry-run mode
        TBE->>TBE: Compute decision but do NOT consume tokens
        TBE-->>RR: Decision { allowed: true, dry_run: true }
        RR-->>RM: allowed=true
        RM-->>GW: Forward request
        GW-->>C: 200 OK
    else Normal mode
        TBE->>R: EVAL token_bucket.lua (key, cost, params)
        
        alt Redis success
            R-->>TBE: { allowed=true, remaining=9 }
        else Redis failure
            TBE->>LF: Evaluate with local in-memory bucket
            LF-->>TBE: { allowed=true, remaining=8 } (best-effort)
            TBE->>O: log(fallback=true)
        end
        
        alt Rejected
            TBE-->>RR: { allowed: false, retry_after: 500 }
            RR-->>RM: allowed=false
            RM-->>GW: HTTP 429 + Retry-After: 500
            GW-->>C: 429 Too Many Requests
        else Allowed
            TBE-->>RR: { allowed: true, remaining: 9 }
            RR-->>RM: allowed=true
            RM->>RM: Forward request to upstream
            RM-->>GW: allowed=true
            GW-->>C: 200 OK
        end
    end
    
    TBE->>O: Emit metrics (latency, decision, bucket state)
    TBE->>O: Enqueue decision log (async)
```

## 4. Concurrency Model

### 4.1 Per-Key Serialization
The Lua script executing on the Redis server ensures that all operations on a single bucket key are **serialized**. Redis is single-threaded for script execution, so two concurrent `Check` requests for the same key are processed atomically. No distributed lock needed.

### 4.2 Node-Level Concurrency
Each data plane node runs multiple worker goroutines/threads:
- Number of workers = `2 × CPU cores` (NIO pattern).
- Workers share a connection pool to Redis (N connections, where N ≈ workers × 2).
- No shared state between workers (stateless design).
- Request affinity: not required (any worker can handle any key).

### 4.3 Backpressure
The gRPC server uses flow control:
- Max concurrent streams per connection: 100,000.
- When connection pool to Redis is exhausted, new requests are queued with a max wait of 5ms.
- If wait exceeds 5ms, return `UNAVAILABLE` with backpressure signal.
- Clients (API Gateway) implement exponential backoff on `UNAVAILABLE`.

## 5. Error Scenarios

| Scenario | Behavior | Response |
|---|---|---|
| Redis connection timeout | Circuit breaker opens (5 failures / 30s) | Fall back to local in-memory bucket |
| Redis replica unreachable | Disable read-replica fallback | Continue with primary only |
| PostgreSQL unreachable (data plane) | Stale rule cache continues serving | Rules may be stale (max 5s) |
| PostgreSQL unreachable (control plane) | Admin API returns 503 | Rules are immutable until recovery |
| Clock skew > 100ms between nodes | Tokens refilled at slightly different times | Acceptable; skew bounded by NTP |
| Lua script execution > 10ms | Log warning, return ALLOW | Safety valve — never reject due to slow script |
| Local fallback memory > 1GB | Flush least-recently-used buckets | LRU eviction |

## 6. Graceful Shutdown

```mermaid
sequenceDiagram
    participant K as K8s / Orchlestrator
    participant RL as Rate Limiter Node
    participant R as Redis
    participant GW as API Gateway

    K->>RL: SIGTERM (shutdown signal)
    RL->>RL: Start draining (10s window)
    RL->>GW: gRPC GOAWAY (stop new requests)
    GW->>RL: In-flight requests complete (max 5s)
    RL->>RL: Wait for pending Redis calls (max 3s)
    RL->>R: Close connection pool
    RL->>RL: Flush async log buffer (1s)
    RL->>RL: Exit
```
