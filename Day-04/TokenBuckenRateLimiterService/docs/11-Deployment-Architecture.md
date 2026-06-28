# Deployment Architecture — Token Bucket Rate Limiter Service

## 1. Deployment Model

The service is deployed as a set of **stateless microservices** running on Kubernetes, with stateful dependencies managed outside of Kubernetes (Redis Cluster, PostgreSQL).

```mermaid
graph TB
    subgraph "Region: us-east-1"
        subgraph "Kubernetes Cluster"
            subgraph "Rate Limiter Data Plane"
                RL_POD1["RL Pod 1<br/>100K dec/s"]
                RL_POD2["RL Pod 2<br/>100K dec/s"]
                RL_POD3["RL Pod N<br/>100K dec/s"]
            end
            subgraph "Rate Limiter Control Plane"
                CP_POD1["Admin API Pod 1"]
                CP_POD2["Admin API Pod 2"]
            end
            subgraph "Supporting Services"
                SIM["Simulation Engine<br/>Pod"]
            end
        end
        
        subgraph "Data Tier"
            REDIS_M["Redis Cluster<br/>3 primaries + 3 replicas<br/>c6g.xlarge × 6"]
            PG["PostgreSQL<br/>Primary + Standby<br/>db.r6g.large × 2"]
        end
        
        subgraph "Observability"
            PROM["Prometheus<br/>Server"]
        end
        
        K8S_ING["Ingress Controller<br/>(For Control Plane)"]
        SVC["Service Mesh<br/>Sidecar Proxy<br/>(Envoy)"]
    end

    subgraph "Region: eu-west-1"
        subgraph "Kubernetes Cluster EU"
            RL_POD_EU1["RL Pod 1<br/>100K dec/s"]
            RL_POD_EU2["RL Pod 2<br/>100K dec/s"]
            CP_POD_EU["Admin API Pod"]
        end
        subgraph "Data Tier EU"
            REDIS_EU["Redis Cluster<br/>3 primaries + 3 replicas"]
            PG_EU["PostgreSQL<br/>Replica (cross-region)"]
        end
    end

    subgraph "Global"
        DNS["Global Traffic Manager<br/>Route53 Latency-based"]
        GRAF["Grafana<br/>Cross-Region Dashboard"]
    end

    RL_POD1 --> SVC
    RL_POD2 --> SVC
    RL_POD3 --> SVC
    SVC --> REDIS_M
    SVC --> PG
    CP_POD1 --> PG
    CP_POD2 --> PG
    CP_POD1 --> REDIS_M
    SIM --> CP_POD1
    
    RL_POD_EU1 --> REDIS_EU
    RL_POD_EU2 --> REDIS_EU
    CP_POD_EU --> PG_EU
    CP_POD_EU --> REDIS_EU
    REDIS_EU -.-> |"Async Replication"| REDIS_M
    PG_EU -.-> |"WAL Streaming"| PG
    
    DNS --> K8S_ING
    DNS --> RL_POD1
    DNS --> RL_POD2
    DNS --> RL_POD_EU1
    
    PROM --> GRAF
```

## 2. Infrastructure Requirements

### 2.1 Compute (Kubernetes)

| Component | Instance Type | Replicas | Resources per Pod |
|---|---|---|---|
| Data plane node | c6i.xlarge (4 vCPU, 8 GB) | 3-20 (HPA) | 2 CPU, 4 GB RAM |
| Control plane node | c6i.large (2 vCPU, 4 GB) | 2-4 (HPA) | 1 CPU, 2 GB RAM |
| Simulation engine | c6i.2xlarge (8 vCPU, 16 GB) | 1 (on-demand) | 4 CPU, 8 GB RAM |

### 2.2 Data Stores

| Component | Instance Type | Count | Storage |
|---|---|---|---|
| Redis (Cluster) | c6g.xlarge (4 vCPU, 8 GB RAM) | 3 primaries + 3 replicas | In-memory only |
| PostgreSQL | db.r6g.large (2 vCPU, 16 GB RAM) | 1 primary + 1 standby | gp3: 500 GB |
| ClickHouse | c6i.2xlarge (8 vCPU, 16 GB) | 3 nodes | gp3: 1 TB each |

### 2.3 Networking

- **Latency budget:** Data plane → Redis < 1ms RTT (same AZ).
- **Bandwidth:** Each data plane node: ~50 Mbps (100K dec/s × ~500 bytes/request).
- **Firewall:** Port 6379 (Redis) accessible only from rate limiter service accounts.
- **TLS:** All inter-service communication uses mTLS.

## 3. Scaling Strategy

### 3.1 Horizontal Pod Autoscaler (Data Plane)

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
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
  minReplicas: 3
  maxReplicas: 20
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 120
    scaleUp:
      stabilizationWindowSeconds: 30
```

### 3.2 Redis Cluster Scaling

Redis Cluster scales by adding shards (primary + replica pairs):
- Current: 3 primaries (3 hash slots groups).
- Scale to: 6 primaries by migrating hash slots.
- No downtime: Redis Cluster supports online re-sharding.
- Trigger: CPU > 70% on any Redis node sustained for 5 minutes.

## 4. CI/CD Pipeline

```mermaid
graph LR
    subgraph "CI"
        CODE["Code Commit"]
        LINT["Lint + Format"]
        TEST["Unit + Integration Tests"]
        BUILD["Container Build<br/>Docker"]
        SCAN["Vulnerability Scan<br/>Trivy"]
    end
    
    subgraph "Deploy"
        DEV["Deploy to Dev<br/>Auto"]
        STAGE["Deploy to Staging<br/>Auto"]
        PROD_CANARY["Deploy to Prod<br/>Canary 10%"]
        PROD_FULL["Deploy to Prod<br/>100%"]
        ROLLBACK["Auto-Rollback<br/>If Error > 1%"]
    end
    
    subgraph "Verify"
        SMOKE["Smoke Tests<br/>Health + Decision API"]
        PERF["Performance Gate<br/>p99 < 5ms"]
        COMP["Compliance Scan<br/>RBAC + Audit"]
    end
    
    CODE --> LINT
    LINT --> TEST
    TEST --> BUILD
    BUILD --> SCAN
    SCAN --> DEV
    DEV --> SMOKE
    SMOKE --> STAGE
    STAGE --> PERF
    PERF --> PROD_CANARY
    PROD_CANARY --> |Error rate +- 1%| PROD_FULL
    PROD_CANARY --> |Error rate > 1%| ROLLBACK
    PROD_FULL --> COMP
```

## 5. Multi-Region Deployment

### 5.1 Active-Active (Data Plane)

Rate limit decision-making is **active-active** across regions:
- Each region has its own data plane nodes and Redis cluster.
- Per-region rate limits are enforced locally with no cross-region dependency.
- Global rate limits (e.g., per-user across all regions) use asynchronous state replication.

### 5.2 Active-Passive (Control Plane)

The Admin API is **active-passive**:
- US-East is the primary region for rule management.
- EU-West and APAC operate as warm standbys.
- On primary region failure, DNS failover routes admin traffic to the secondary region.
- Cross-region PostgreSQL replication (streaming WAL) keeps the standby current.

### 5.3 Cross-Region Consistency for Global Limits

For global rate limits (e.g., "User X can make 1000 requests/hour globally"):

```mermaid
sequenceDiagram
    participant R1 as Region A<br/>Rate Limiter
    participant R2 as Region B<br/>Rate Limiter
    participant R3 as Region C<br/>Rate Limiter
    participant G as Global Redis<br/>(Aggregated)

    Note over R1,R3: Each region makes local decisions independently
    
    R1->>G: ASYNC: Report token consumption every 1s
    R2->>G: ASYNC: Report token consumption every 1s
    R3->>G: ASYNC: Report token consumption every 1s
    
    Note over G: Global aggregator computes: total = sum(region reports)
    
    G->>R1: ASYNC: Broadcast global remaining tokens (every 1s)
    G->>R2: ASYNC: Broadcast global remaining tokens (every 1s)
    G->>R3: ASYNC: Broadcast global remaining tokens (every 1s)
    
    Note over R1: Uses min(local_remaining, global_remaining / N_regions)
    Note over R2: Local over-consumption bounded by 1 second × N_regions × rate
```

**Trade-off:** Global rate limits are eventually consistent. A user could over-consume by up to (regions × 1 second × rate) before the global aggregate catches up. For most use cases (hourly/daily limits), this is acceptable.
