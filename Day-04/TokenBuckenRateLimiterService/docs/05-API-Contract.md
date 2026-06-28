# API Contract — Token Bucket Rate Limiter Service

## 1. Transport

All rate limiter APIs communicate over **gRPC** (internal service-to-service) with a **REST gateway** for administrative operations. gRPC is chosen for:
- Strongly typed contracts (protobuf) enforced at compile time.
- Efficient binary serialization for high-throughput decision requests.
- Built-in streaming support for async decision paths.

Base paths:
- **Data Plane:** `rate-limiter.service.internal:8443` (gRPC)
- **Control Plane:** `rate-limiter-admin.service.internal:8444` (gRPC + REST gateway)

## 2. Data Plane APIs

### 2.1 Check Rate Limit (Unary)

The primary decision API. Called synchronously on every request that needs rate limiting.

```protobuf
service RateLimiter {
    // Evaluate whether a request is allowed.
    // Returns the decision and current bucket state.
    rpc Check(CheckRequest) returns (CheckResponse);
}

message CheckRequest {
    // The rate limit key (e.g., "client:abc123:endpoint:/orders")
    string bucket_key = 1;

    // Number of tokens to consume (default: 1)
    int32 cost = 2;

    // Optional context for routing / logging
    RequestContext context = 3;
}

message RequestContext {
    string client_id = 1;
    string endpoint = 2;
    string http_method = 3;
    string user_id = 4;
    string region = 5;
    map<string, string> metadata = 6;
}

message CheckResponse {
    // Whether the request is allowed
    bool allowed = 1;

    // Remaining tokens in the bucket after this decision
    int32 remaining_tokens = 2;

    // Unix timestamp in milliseconds when the bucket will next refill
    int64 next_refill_at_ms = 3;

    // How long the caller should wait before retrying (milliseconds)
    // Only set when allowed == false
    int32 retry_after_ms = 4;

    // Current bucket capacity
    int32 max_tokens = 5;

    // Unique decision ID for audit trail
    string decision_id = 6;
}
```

**Request flow:**
```
1. Client sends CheckRequest with bucket_key and optional cost.
2. Service loads or creates the token bucket for this key.
3. Service computes: current_tokens = min(max_tokens, last_tokens + elapsed * refill_rate)
4. If current_tokens >= cost → consume, return allowed=true.
5. Else → return allowed=false, retry_after = time_until_next_refill.
6. Decision is logged asynchronously.
```

### 2.2 Batch Check (Unary)

For callers that need to check multiple rate limits for a single request (e.g., API gateway checking both per-IP and per-endpoint limits).

```protobuf
rpc BatchCheck(BatchCheckRequest) returns (BatchCheckResponse);

message BatchCheckRequest {
    repeated CheckRequest checks = 1;
    // If true, all checks must pass for the overall decision to be "allowed"
    bool require_all = 2;
}

message BatchCheckResponse {
    repeated CheckResponse results = 1;
    // Overall decision (AND of all results if require_all=true)
    bool overall_allowed = 2;
}
```

### 2.3 Async Decision Stream (Server-Streaming)

For high-throughput callers that can tolerate deferred decisions.

```protobuf
rpc CheckStream(stream CheckRequest) returns (stream CheckResponse);
```

The client sends a stream of requests. The server processes them in order per-key and returns decisions as they become available. This is primarily for internal batch processing pipelines.

### 2.4 Token Top-Up (Unary)

Allows authorized systems to programmatically add tokens to a bucket (e.g., when a customer upgrades their plan mid-cycle).

```protobuf
rpc TopUp(TopUpRequest) returns (TopUpResponse);

message TopUpRequest {
    string bucket_key = 1;
    int32 tokens_to_add = 2;
    string reason = 3;  // Audit reason
}
```

## 3. Control Plane APIs

### 3.1 Rule Management

```protobuf
service RateLimitAdmin {
    // Create or update a rate limit rule
    rpc UpsertRule(UpsertRuleRequest) returns (Rule);

    // Get a specific rule
    rpc GetRule(GetRuleRequest) returns (Rule);

    // List rules with optional filters
    rpc ListRules(ListRulesRequest) returns (ListRulesResponse);

    // Delete a rule
    rpc DeleteRule(DeleteRuleRequest) returns (DeleteRuleResponse);

    // Toggle dry-run mode for a rule
    rpc SetDryRun(SetDryRunRequest) returns (Rule);
}

message RateLimitRule {
    string rule_id = 1;
    string key_pattern = 2;         // e.g., "client:{client_id}:endpoint:/orders"
    int32 max_tokens = 3;
    int32 refill_rate = 4;
    int32 refill_interval_ms = 5;
    int32 request_cost = 6;
    int32 burst_max_tokens = 7;
    int32 priority = 8;             // Higher = evaluated first
    string mode = 9;                // "reject" | "defer"
    bool dry_run = 10;
    repeated string exempt_clients = 11;
    string description = 12;
    string created_by = 13;
    int64 created_at_unix_ms = 14;
    int64 updated_at_unix_ms = 15;
    int32 version = 16;
}
```

### 3.2 Rule Resolution Contract

When the data plane receives a `CheckRequest`, it must resolve which `RateLimitRule` applies:

```
1. Extract key pattern variables from the bucket_key.
   Example: key="client:c123:endpoint:/orders"
            pattern="client:{client_id}:endpoint:{path}"
            → client_id="c123", path="/orders"

2. Match against all active rules:
   a. Find all rules whose key_pattern matches the bucket_key.
   b. Sort by priority (highest first).
   c. Return the first match.
3. If no rule matches, return "allow" (default behavior).
```

Rules are cached locally on each rate limiter node with a TTL of 5 seconds. Cache is invalidated on rule change via Pub/Sub.

### 3.3 Simulation API

```protobuf
rpc Simulate(SimulateRequest) returns (SimulateResponse);

message SimulateRequest {
    repeated RateLimitRule rules = 1;
    // Traffic pattern: simulated requests over time
    repeated SimulatedRequest traffic = 2;
}

message SimulatedRequest {
    string bucket_key = 1;
    int32 cost = 2;
    int64 arrival_time_unix_ms = 3;
}

message SimulateResponse {
    int32 total_requests = 1;
    int32 allowed = 2;
    int32 rejected = 3;
    repeated DecisionSummary decisions = 4;
}
```

## 4. REST Gateway Mapping

The control plane APIs are also exposed as REST for browser-based Admin Console access:

| gRPC Method | REST Equivalent |
|---|---|
| `UpsertRule` | `PUT /api/v1/rules/{rule_id}` |
| `GetRule` | `GET /api/v1/rules/{rule_id}` |
| `ListRules` | `GET /api/v1/rules` |
| `DeleteRule` | `DELETE /api/v1/rules/{rule_id}` |
| `SetDryRun` | `POST /api/v1/rules/{rule_id}/dry-run` |
| `Simulate` | `POST /api/v1/simulate` |

## 5. Error Codes

| gRPC Code | HTTP Code | Condition |
|---|---|---|
| `OK` | 200 | Decision returned successfully |
| `INVALID_ARGUMENT` | 400 | Bucket key is empty; cost is zero or negative |
| `NOT_FOUND` | 404 | Rule ID not found |
| `FAILED_PRECONDITION` | 428 | Rate limiter in fail-close degradation mode |
| `UNAVAILABLE` | 503 | Backing store unreachable; cannot make decision |
| `DEADLINE_EXCEEDED` | 504 | Decision took longer than configured timeout |
