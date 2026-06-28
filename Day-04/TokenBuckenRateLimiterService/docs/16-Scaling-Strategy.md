# Scaling Strategy — Token Bucket Rate Limiter Service

## 1. Scaling Dimensions

The rate limiter must scale across four dimensions:

| Dimension | Current Target | Future Target | Primary Constraint |
|---|---|---|---|
| **Throughput** (decisions/sec) | 5M/sec global | 50M/sec global | Redis CPU |
| **Bucket count** (unique keys) | 10M | 100M | Redis memory |
| **Rule count** (configs) | 100K | 1M | PostgreSQL + cache size |
| **Latency** (p99 decision) | 5ms | 2ms | Network + Redis proximity |

## 2. Data Plane Scaling

### 2.1 Horizontal Scaling (Throughput)

The data plane scales horizontally by adding more stateless pods. Since all nodes are stateless and share nothing, scaling is linear:

```mermaid
graph TB
    subgraph "Current: 5M dec/s"
        LB1["Internal LB"]
        P1["Pod × 50<br/>100K dec/s each"]
        R1["Redis Cluster<br/>3 primaries × 3 replicas"]
        LB1 --> P1
        P1 --> R1
    end
    
    subgraph "Scale to: 15M dec/s"
        LB2["Internal LB"]
        P2["Pod × 150<br/>100K dec/s each"]
        R2["Redis Cluster<br/>9 primaries × 9 replicas"]
        LB2 --> P2
        P2 --> R2
    end
```

**Scaling formula:** `nodes = (target_decisions_per_second / 100,000) × redundancy_factor(1.2)`

At 5M dec/s: `(5,000,000 / 100,000) × 1.2 = 60 nodes` (plus HPA buffer)

### 2.2 Redis Scaling

Redis is the primary bottleneck at scale. Scaling strategy:

**Phase 1 — Add shards (0 → 3 → 9 → 27 primaries):**
- Start with 3 primaries (Redis Cluster default).
- Monitor CPU per node. When any node exceeds 60% CPU, add 3 more primaries.
- Redis Cluster supports online re-sharding — no downtime.

**Phase 2 — Key splitting for hot keys:**
- If a single key receives > 10,000 dec/s, split it into N sub-keys.
- Each sub-key has `max_tokens / N` capacity.
- Distributes load across N hash slots.

**Phase 3 — Read replicas for cache warming:**
- Redis replicas serve as read-only fallback.
- Not used for primary decisions (write must go to primary).
- Used for cache warming: when a new node starts, it can pre-fetch hot keys from replicas.

### 2.3 Redis Memory Scaling

```mermaid
graph LR
    subgraph "Memory Budget per 1M Buckets"
        ITEM["Key + Hash fields<br/>~200 bytes each<br/>= 200 MB"]
        OVERHEAD["Redis overhead<br/>(dict, pointers)<br/>~100 MB"]
        AOF["AOF buffer<br/>~50 MB"]
        TOTAL["Total<br/>~350 MB per 1M<br/>buckets"]
    end
    
    ITEM --> TOTAL
    OVERHEAD --> TOTAL
    AOF --> TOTAL
```

At 10M buckets: ~3.5 GB per Redis node. At 100M buckets: ~35 GB. Scaling to 100M buckets requires larger instance types (r6g.2xlarge → r6g.8xlarge) or more shards.

## 3. Control Plane Scaling

### 3.1 Admin API

The control plane has low traffic (tens of requests per second — rule changes are infrequent). It scales by adding pods behind a load balancer.

**HPA configuration:**
- Metric: CPU > 60%.
- Min: 2 pods (HA).
- Max: 10 pods.

### 3.2 Rule Cache Propagation

As the number of nodes grows, rule cache propagation becomes a concern:

| Node Count | Invalidation Strategy | Max Propagation Time |
|---|---|---|
| < 50 | Redis Pub/Sub (all nodes subscribe) | < 1s |
| 50–500 | Redis Pub/Sub + polling (5s backup) | < 1s (Pub/Sub) / < 5s (poll) |
| > 500 | Gossip protocol (SWIM-based) + periodic polling | < 2s (gossip) / < 5s (poll) |

At the current scale (< 200 nodes), Redis Pub/Sub is sufficient. At massive scale (> 500 nodes), Pub/Sub scalability becomes a concern (each message is fanned out to N subscribers). A gossip-based protocol distributes the invalidation message without a single fan-out bottleneck.

## 4. Global Scaling (Multi-Region)

```mermaid
graph TB
    subgraph "Global Traffic Manager"
        T["Route53 Latency-based<br/>+ Health Checks"]
    end

    subgraph "Region: us-east-1"
        RL_US["Data Plane<br/>100 pods"]
        R_US["Redis Cluster<br/>9 primaries"]
        PG_US["PostgreSQL<br/>Primary"]
    end

    subgraph "Region: eu-west-1"
        RL_EU["Data Plane<br/>60 pods"]
        R_EU["Redis Cluster<br/>6 primaries"]
        PG_EU["PostgreSQL<br/>Read Replica"]
    end

    subgraph "Region: ap-southeast-1"
        RL_AP["Data Plane<br/>40 pods"]
        R_AP["Redis Cluster<br/>4 primaries"]
        PG_AP["PostgreSQL<br/>Read Replica"]
    end

    T --> RL_US
    T --> RL_EU
    T --> RL_AP
    
    PG_US -.-> |"WAL Streaming"| PG_EU
    PG_US -.-> |"WAL Streaming"| PG_AP
```

### 4.1 Regional Sizing by Traffic

| Region | Traffic Share | Data Plane Pods | Redis Primaries |
|---|---|---|---|
| us-east-1 | 40% | 100 | 9 |
| eu-west-1 | 30% | 60 | 6 |
| ap-southeast-1 | 20% | 40 | 4 |
| sa-east-1 | 10% | 20 | 2 |

### 4.2 Global Rate Limits

For rate limits that span regions (e.g., "1000 requests/hour per user globally"):

**Approach: Global token ledger**

```mermaid
sequenceDiagram
    participant R1 as RL Node (us-east-1)
    participant R2 as RL Node (eu-west-1)
    participant L as Global Ledger<br/>(Central Redis)
    participant P as Period Reset

    Note over R1,R2: Each region consumes from a local cache of the global limit
    R1->>R1: Local bucket: tokens=100 (cached global, updated every 5s)
    R1->>L: Report consumption (every 5s)
    R2->>R2: Local bucket: tokens=100
    R2->>L: Report consumption (every 5s)
    
    Note over L: Ledger tracks: region_A_consumed + region_B_consumed
    
    L->>L: remaining_global = max_tokens - sum(region_reports)
    L->>R1: Broadcast: remaining_global (every 5s)
    L->>R2: Broadcast: remaining_global (every 5s)
    
    R1->>R1: effective_limit = min(local_tokens, remaining_global / 3)
    
    Note over P: On period reset → all ledgers reset
```

**Trade-off:** Global limits are eventually consistent (up to 5 seconds stale). Over-consumption of up to `regions × 5s × rate` is possible. For hourly/daily limits, this is acceptable.

## 5. Auto-Scaling Rules

### 5.1 Data Plane HPA

```yaml
metrics:
  - type: Pods
    pods:
      metric: rate_limiter_decisions_per_second
      target:
        type: AverageValue
        averageValue: 80000
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
behavior:
  scaleDown:
    stabilizationWindowSeconds: 120
    policies:
      - type: Pods
        value: 2
        periodSeconds: 60
  scaleUp:
    stabilizationWindowSeconds: 30
    policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

### 5.2 Redis Cluster Scaling (Manual / Scheduled)

Redis Cluster does not support auto-scaling natively. Scaling is a planned operation:

**Trigger:** Primary CPU > 60% for 1 hour or Memory > 70%.

**Process:**
1. Add 3 new Redis nodes to the cluster.
2. Rebalance hash slots: `redis-cli --cluster rebalance <new-node>`.
3. Monitor: CPU utilization across all nodes should decrease proportionally.
4. Verify: Decision latency remains stable during the migration.

## 6. Capacity Planning

### 6.1 Current Requirements

| Component | Current | Growth Rate | 12-Month Projection |
|---|---|---|---|
| Decisions/sec | 5M | 3× / year | 15M |
| Unique bucket keys | 10M | 2× / year | 20M |
| Rate limit rules | 10K | 1.5× / year | 15K |
| Redis memory | 35 GB | 2× / year | 70 GB |

### 6.2 Cost Projection

| Component | Current Monthly | 12-Month Projected | Scaling Driver |
|---|---|---|---|
| Data plane pods (compute) | $8K | $24K | Decisions/sec |
| Redis Cluster | $12K | $36K | Bucket count + throughput |
| PostgreSQL | $2K | $3K | Rule count (minor) |
| ClickHouse | $4K | $8K | Decision log volume |
| **Total** | **$26K** | **$71K** | |

**Optimization levers:**
- Reserved instances for Redis and PostgreSQL (30% discount).
- Spot instances for data plane pods (60% discount).
- Decision log sampling (reduce ClickHouse volume by 90% with 1% sampling for high-traffic keys).
