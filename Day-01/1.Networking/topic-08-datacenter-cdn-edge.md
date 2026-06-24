# Topic 08: Data Center Architecture, CDN & Edge Computing

## 1. Theoretical Foundation & System Mechanics

### The Core Concept

Modern internet-scale applications are not served from a single server room. They operate from a global network of **data centers** interconnected through **Content Delivery Networks (CDNs)** and **edge computing** infrastructure. Understanding this stack is essential for any engineer building systems that serve millions of users across continents.

**Data Center Tiers (Tier 1-4):**

```
TIER CLASSIFICATION MATRIX

                  Availability          Annual Downtime      Redundancy
Tier 1            99.671%               28.8 hours           None (N)
Tier 2            99.741%               22.0 hours           Partial (N+1)
Tier 3            99.982%               1.6 hours            N+1 (maintainable)
Tier 4            99.995%               26.3 minutes         2N+1 (fault-tolerant)
```

- **Tier 1:** Basic capacity. No redundancy. Single path for power/cooling. A UPS battery and that is it. Used for dev/test environments.
- **Tier 2:** Redundant power/cooling components (N+1) but still a single distribution path. Maintenance requires downtime.
- **Tier 3:** Concurrently maintainable. N+1 redundancy with dual power feeds. Any component can be taken offline without disrupting operations. This is the **minimum enterprise standard**.
- **Tier 4:** Fault-tolerant. 2N+1 redundancy — every component is dual-powered with an independent backup. Automatic failover. Zero single point of failure.

**Power Redundancy Models:**

```mermaid
flowchart TD
    subgraph N1["N+1 Redundancy: 2 units required, 1 spare"]
        PSU1["PSU1"] --> S1["Server"]
        PSU2["PSU2"] --> S2["Server"]
        PSU3["PSU3 (Spare)"] --> S3["Server"]
    end
    subgraph N2["2N Redundancy: Dual independent feeds"]
        AFEED["A-Feed"] --> PSA["PSU-A"] --> S4["Server"] --> CA["Cooling-A"]
        BFEED["B-Feed"] --> PSB["PSU-B"] --> S5["Server"] --> CB["Cooling-B"]
    end
    subgraph N3["2N+1 Redundancy: Fault-tolerant"]
        AF2["A-Feed"] --> ACOOL["A-Redundant Cooling"]
        BF2["B-Feed"] --> BCOOL["B-Redundant Cooling"]
        STBY["Standby"] -.-> BCOOL
    end

    style N1 fill:#e1f5fe,stroke:#01579b,color:#000
    style N2 fill:#fff3e0,stroke:#e65100,color:#000
    style N3 fill:#e8f5e9,stroke:#1b5e20,color:#000
```

A Tier 4 data center costs roughly **2-3x more** per square foot than Tier 3. The decision of which tier to use depends on the application's **criticality** — a banking transaction system requires Tier 4; a content website can operate on Tier 3.

**Server Rack Architecture & ToR Switches:**

```mermaid
flowchart TD
    subgraph SPINE["Spine Layer — Aggregation"]
        SP1["Spine-1"]
        SP2["Spine-2"]
        SP3["Spine-3"]
    end
    subgraph TOR["Top of Rack (ToR) Switches"]
        T1["ToR-1"]
        T2["ToR-2"]
        T3["ToR-3"]
    end
    subgraph RACKS["Server Racks"]
        R1["Rack 1<br/>Servers"]
        R2["Rack 2<br/>Servers"]
        R3["Rack 3<br/>Servers"]
    end

    SP1 --- T1
    SP1 --- T2
    SP2 --- T1
    SP2 --- T2
    SP2 --- T3
    SP3 --- T2
    SP3 --- T3

    T1 --- R1
    T2 --- R2
    T3 --- R3

    style SPINE fill:#e1f5fe,stroke:#01579b,color:#000
    style TOR fill:#fff3e0,stroke:#e65100,color:#000
    style RACKS fill:#e8f5e9,stroke:#1b5e20,color:#000
```

**Top of Rack (ToR)** switches sit at the top of each server rack. Every server in the rack connects to the ToR via 10/25/40/100 GbE links. The ToR then connects upward to **spine switches** (the aggregation layer). This is the foundation of the **Spine-Leaf (Clos) topology**.

**Spine-Leaf Network Topology (Clos Network):**

A Clos network is a multistage circuit-switching topology originally invented by Charles Clos in 1952. It has been revived in modern data centers because of its **predictable latency** and **linear scalability**.

```mermaid
flowchart TD
    CR1["Core Router"]
    CR2["Core Router"]
    S1["Spine-1"]
    S2["Spine-2"]
    S3["Spine-3"]
    L1A["Leaf-1"]
    L1B["Leaf-2"]
    L2A["Leaf-1"]
    L2B["Leaf-2"]
    L3A["Leaf-1"]
    L3B["Leaf-2"]
    RA["Rack A"]
    RB["Rack B"]
    RC["Rack C"]
    RD["Rack D"]
    RE["Rack E"]
    RF["Rack F"]

    CR1 --- S1
    CR1 --- S2
    CR2 --- S2
    CR2 --- S3
    S1 --- L1A
    S1 --- L1B
    S2 --- L1A
    S2 --- L1B
    S2 --- L2A
    S2 --- L2B
    S3 --- L2A
    S3 --- L2B
    S3 --- L3A
    S3 --- L3B
    L1A --- RA
    L1B --- RB
    L2A --- RC
    L2B --- RD
    L3A --- RE
    L3B --- RF

    style CR1 fill:#e1f5fe,stroke:#01579b,color:#000
    style CR2 fill:#e1f5fe,stroke:#01579b,color:#000
    style S1 fill:#fff3e0,stroke:#e65100,color:#000
    style S2 fill:#fff3e0,stroke:#e65100,color:#000
    style S3 fill:#fff3e0,stroke:#e65100,color:#000
    style L1A fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style L1B fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style L2A fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style L2B fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style L3A fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style L3B fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style RA fill:#e8f5e9,stroke:#1b5e20,color:#000
    style RB fill:#e8f5e9,stroke:#1b5e20,color:#000
    style RC fill:#e8f5e9,stroke:#1b5e20,color:#000
    style RD fill:#e8f5e9,stroke:#1b5e20,color:#000
    style RE fill:#e8f5e9,stroke:#1b5e20,color:#000
    style RF fill:#e8f5e9,stroke:#1b5e20,color:#000
```

Key properties of a Clos network:
- **Every leaf switch connects to every spine switch.** This means any server can reach any other server in exactly **2 hops** (leaf -> spine -> leaf).
- **Bandwidth scales linearly** with the number of spine switches. Add more spines -> add more bandwidth.
- **No single point of failure.** If a spine dies, traffic re-routes through remaining spines.
- **ECMP (Equal-Cost Multi-Path)** routing distributes traffic across all available spines.

The bottleneck in a spine-leaf topology is the **oversubscription ratio**: the ratio of downlink bandwidth (to servers) to uplink bandwidth (to spines). A 3:1 oversubscription ratio means 3 Gbps of server bandwidth shares 1 Gbps of spine bandwidth.

### The "Why"

**Why do we need data center tiers and spine-leaf topologies?**

The bottleneck being solved is **predictable performance under failure**. In traditional three-tier architectures (access-aggregate-core), traffic must traverse multiple layers, creating variable latency and bandwidth contention. A spine-leaf Clos network guarantees:

- **Predictable latency:** Exactly 2 hops between any two points.
- **Linear bandwidth scaling:** Add more spines, get more bisection bandwidth.
- **Fault isolation:** A failed spine switch affects capacity, not connectivity.

**Why do we need CDNs?**

The fundamental bottleneck is **the speed of light** combined with **internet congestion**. A user in Mumbai requesting content from a server in Virginia will experience:
- Round-trip latency of ~200-300ms (geographic distance)
- Packet loss across 15+ router hops
- Peering congestion at transit points

A CDN solves this by **caching content at the edge** — bringing data as close to the user as physically possible.

```mermaid
flowchart TD
    subgraph WITHOUT["Without CDN"]
        USER1["User (Mumbai)"] --> ISP1["ISP A"] --> TRANSIT["Transit"] --> ISP2["ISP B"] --> ORIGIN1["Origin (Virginia)<br/>250ms latency, 15 hops, 3% loss"]
    end
    subgraph WITH["With CDN"]
        USER2["User (Mumbai)"] --> EDGE["CDN Edge Mumbai"]
        EDGE -->|Cache Hit: 5ms| USER2
        EDGE -->|Cache Miss| REGIONAL["Regional Edge Singapore"] --> ORIGIN2["Origin<br/>~80ms on cache miss"]
    end

    style WITHOUT fill:#fce4ec,stroke:#c62828,color:#000
    style WITH fill:#e8f5e9,stroke:#1b5e20,color:#000
```

### Trade-offs

| Component | Hidden Cost | Impact |
|-----------|-------------|--------|
| **Tier 4 DC** | 2-3x CapEx, 40% more power per sq ft | Only justified for mission-critical workloads |
| **Spine-Leaf** | More fiber/cabling than traditional 3-tier | Higher initial cabling cost, simpler ongoing ops |
| **CDN Caching** | Stale content, cache miss penalty | 1% cache miss degradation can increase origin load 100x |
| **Edge Computing** | Limited compute, state management complexity | Stateless apps only; state must live at origin |

**SSL Termination at the Edge** — While offloading SSL to the CDN edge reduces origin CPU load, it means the CDN provider can see decrypted traffic. For PCI/HIPAA compliance, end-to-end encryption may be mandatory, preventing edge SSL termination.

---

## 2. Production Implementation (Full Stack & Cloud)

### Backend & Code Architecture

**Cache-Control Header Strategy (Spring Boot):**

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(5))
            .disableCachingNullValues()
            .computePrefixWith(cacheName -> "app:" + cacheName + ":");
        return RedisCacheManager.builder(redisConnectionFactory())
            .cacheDefaults(config)
            .build();
    }
}

@RestController
@RequestMapping("/api/v1/projects")
public class ProjectController {

    @GetMapping("/{id}")
    public ResponseEntity<Project> getProject(@PathVariable String id) {
        Project project = projectService.findById(id);
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(300, TimeUnit.SECONDS)
                .sMaxAge(600, TimeUnit.SECONDS)
                .mustRevalidate())
            .eTag(project.getVersion())
            .body(project);
    }
}
```

The `s-maxage` directive tells CDN edges to cache for 600 seconds, while the browser caches for only 300 seconds. The `ETag` enables **conditional requests** — the CDN sends a `If-None-Match` header to origin and gets a `304 Not Modified` response without payload, saving bandwidth.

### DevOps & Infrastructure

**Dockerfile for Edge-Optimized Static Assets:**

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM nginx:1.25-alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Security headers for edge serving
RUN echo 'add_header X-Content-Type-Options "nosniff";' \
     'add_header X-Frame-Options "DENY";' \
     'add_header Cache-Control "public, max-age=31536000, immutable";' \
     >> /etc/nginx/conf.d/security.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Kubernetes Deployment with Pod Topology Spread:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: edge-cache-worker
spec:
  replicas: 12
  selector:
    matchLabels:
      app: edge-cache-worker
  template:
    metadata:
      labels:
        app: edge-cache-worker
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: edge-cache-worker
      containers:
        - name: worker
          image: myapp/edge-worker:1.0
          env:
            - name: CACHE_SIZE_MB
              value: "4096"
            - name: ORIGIN_URL
              value: "https://origin.internal.example.com"
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "2"
```

The `topologySpreadConstraints` ensures pods are evenly distributed across availability zones — critical for edge caching where every zone must have a cache node.

**Terraform for CloudFront Distribution:**

```hcl
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Production CDN distribution"
  price_class         = "PriceClass_All"
  http_version        = "http2and3"

  origin {
    domain_name = aws_lb.main.dns_name
    origin_id   = "ALBOrigin"
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
    origin_shield {
      enabled              = true
      origin_shield_region = "ap-south-1"
    }
  }

  default_cache_behavior {
    target_origin_id       = "ALBOrigin"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    compress               = true

    forwarded_values {
      query_string = true
      cookies {
        forward = "whitelist"
        whitelisted_names = ["session", "locale"]
      }
    }

    min_ttl     = 0
    default_ttl = 3600
    max_ttl     = 86400

    function_association {
      event_type = "viewer-request"
      function_arn = aws_cloudfront_function.redirect.arn
    }
  }

  ordered_cache_behavior {
    path_pattern     = "/api/*"
    target_origin_id = "ALBOrigin"
    viewer_protocol_policy = "https-only"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    compress         = true
    forwarded_values {
      query_string = true
      cookies {
        forward = "all"
      }
    }
    min_ttl     = 0
    default_ttl = 0
    max_ttl     = 0
  }

  restrictions {
    geo_restriction {
      restriction_type = "whitelist"
      locations        = ["IN", "US", "GB", "DE", "SG"]
    }
  }

  viewer_certificate {
    acm_certificate_arn      = "arn:aws:acm:us-east-1:xxxxx:certificate/xxxxx"
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }
}
```

**Origin Shield** creates a regional caching layer that aggregates requests from multiple edge locations before forwarding to the origin. This reduces origin load by up to 5x.

### Cloud Architecture

```mermaid
flowchart TD
    USERS["Users Worldwide"]
    R53["Route53 Latency-Based Routing"]
    subgraph EDGE2["CloudFront Edge Locations"]
        CF_IN["CloudFront Edge IN"]
        CF_SG["CloudFront Edge SG"]
        CF_DE["CloudFront Edge DE"]
    end
    OS["Origin Shield (ap-south-1)"]
    WAF2["WAF (Web ACL)"]
    ALB2["ALB (Application Load Balancer)"]
    subgraph COMPUTE["ECS Fargate"]
        WEB["Web Servers"]
        API["API Servers"]
    end
    CACHE["ElastiCache Redis<br/>Session/Query Cache"]
    AURORA["RDS Aurora<br/>Primary + Read Replica"]
    GLOBAL_CACHE["ElastiCache Global Datastore<br/>Replicated to us-west-2 for DR"]

    USERS --> R53
    R53 --> CF_IN
    R53 --> CF_SG
    R53 --> CF_DE
    CF_IN --> OS
    CF_SG --> OS
    CF_DE --> OS
    OS --> WAF2
    WAF2 --> ALB2
    ALB2 --> WEB
    ALB2 --> API
    WEB --> CACHE
    API --> CACHE
    WEB --> AURORA
    API --> AURORA
    CACHE --> GLOBAL_CACHE

    style USERS fill:#e1f5fe,stroke:#01579b,color:#000
    style R53 fill:#fff3e0,stroke:#e65100,color:#000
    style CF_IN fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style CF_SG fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style CF_DE fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style OS fill:#fce4ec,stroke:#c62828,color:#000
    style WAF2 fill:#fff8e1,stroke:#f9a825,color:#000
    style ALB2 fill:#e8f5e9,stroke:#1b5e20,color:#000
    style WEB fill:#e0f2f1,stroke:#004d40,color:#000
    style API fill:#e0f2f1,stroke:#004d40,color:#000
    style CACHE fill:#fce4ec,stroke:#c62828,color:#000
    style AURORA fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style GLOBAL_CACHE fill:#fff3e0,stroke:#e65100,color:#000
```

**Regional Edge Caches** sit between edge locations and the origin. They have larger caches (hundreds of GB) and longer TTLs, acting as a shock absorber for the origin.

---

## 3. Real-World Scaling Scenarios

### The Bottleneck

**Scenario:** A major e-commerce platform runs a Flash Sale event. 5 million concurrent users hit the site within 30 seconds. The origin servers are in a single AWS region (us-east-1).

**Failure Sequence:**

```mermaid
flowchart TD
    T0["T=0s: Sale starts. 5M users request product pages."]
    T1["T=1s: CDN edge locations (100+) receive requests.<br/>Cache miss rate spikes to 70%"]
    T3["T=3s: All 100+ edges simultaneously request origin<br/>Regional Edge Cache gets 100x normal traffic"]
    T5["T=5s: Regional Edge Cache saturates its 10Gbps uplink"]
    T7["T=7s: ALB receives 500K req/s. Target groups max out at 200K"]
    T10["T=10s: RDS Primary hits 100% CPU. Connection pool exhausted"]
    T15["T=15s: ALB 503 errors propagate. Users see Service Unavailable"]
    T30["T=30s: Cache stampede — all edges retry simultaneously<br/>Origin overwhelmed. Full outage for 45 minutes"]

    T0 --> T1 --> T3 --> T5 --> T7 --> T10 --> T15 --> T30
    style T0 fill:#e1f5fe,stroke:#01579b,color:#000
    style T1 fill:#fff3e0,stroke:#e65100,color:#000
    style T3 fill:#fff3e0,stroke:#e65100,color:#000
    style T5 fill:#fce4ec,stroke:#c62828,color:#000
    style T7 fill:#fce4ec,stroke:#c62828,color:#000
    style T10 fill:#fce4ec,stroke:#c62828,color:#000
    style T15 fill:#c62828,stroke:#b71c1c,color:#fff
    style T30 fill:#b71c1c,stroke:#880e4f,color:#fff
```

**Root Cause Analysis:**
1. **Cache Stampede:** All edges missed cache simultaneously and all hit origin at once.
2. **Single Region:** No regional diversity for compute/database.
3. **No read replicas:** All queries hit the primary database.
4. **No request collapsing:** Origin Shield was not configured, so redundant requests reached origin.

### The Solution

**Step-by-Step Architectural Adjustments:**

```mermaid
flowchart TD
    subgraph P1["PHASE 1: Immediate Mitigation (Minutes)"]
        direction TB
        OS2["1. Enable Origin Shield with request collapsing<br/>Only one request per unique resource reaches origin<br/>Reduces origin load by 70-90%"]
        TTL["2. Increase cache TTL for product pages<br/>Extend from 300s to 3600s<br/>Cache-Control: s-maxage=3600"]
        PREWARM["3. Enable pre-warming<br/>Push content to CDN before sale starts"]
    end
    subgraph P2["PHASE 2: Short-Term Fixes (Days)"]
        direction TB
        MULTI["4. Multi-region deployment<br/>us-east-1, eu-west-1, ap-south-1"]
        REPLICA["5. Read replicas with Global Database<br/>Local read replica per region<br/>Async replication to primary"]
        SURGE["6. Implement surge queue<br/>Excess → SQS → Lambda → async<br/>'You are in queue' response"]
    end
    subgraph P3["PHASE 3: Architectural Redesign (Weeks)"]
        direction TB
        STALE["7. Stale-while-revalidate<br/>Serve stale while fetching fresh<br/>Zero cache miss penalty"]
        SHARD["8. Database sharding<br/>16 RDS instances<br/>Product ID hash → shard"]
        FAILOVER["9. Regional failover automation<br/>Route53 health checks<br/>Failover to eu-west-1"]
    end
    P1 --> P2 --> P3
    style P1 fill:#e1f5fe,stroke:#01579b,color:#000
    style P2 fill:#fff3e0,stroke:#e65100,color:#000
    style P3 fill:#e8f5e9,stroke:#1b5e20,color:#000
```

```mermaid
flowchart TD
    GU["Global Users"]
    R53LB["Route53 Latency-Based"]
    subgraph EDGES["Edge Locations"]
        IN["IN"]
        SG["SG"]
        DE["DE"]
        BR["BR"]
        JP["JP"]
    end
    subgraph US_EAST["us-east-1"]
        OS1["Origin Shield"]
        ALB1["ALB"]
        RDSR1["RDS Read"]
        ECS1["ECS Tasks"]
    end
    subgraph EU_WEST["eu-west-1"]
        OS2["Origin Shield"]
        ALB2["ALB"]
        RDSR2["RDS Read"]
        ECS2["ECS Tasks"]
    end
    RDSP["RDS Primary (us-east-1)"]
    AURORA_GD["Aurora Global Database<br/>(Cross-region replication)"]

    GU --> R53LB
    R53LB --> IN
    R53LB --> SG
    R53LB --> DE
    R53LB --> BR
    R53LB --> JP
    IN --> OS1
    SG --> OS1
    DE --> OS2
    BR --> OS2
    JP --> OS2
    OS1 --> ALB1
    ALB1 --> ECS1
    ALB1 --> RDSR1
    OS2 --> ALB2
    ALB2 --> ECS2
    ALB2 --> RDSR2
    ECS1 --> RDSP
    ECS2 --> RDSP
    RDSR1 --> RDSP
    RDSR2 --> RDSP
    RDSP --> AURORA_GD

    style GU fill:#e1f5fe,stroke:#01579b,color:#000
    style R53LB fill:#fff3e0,stroke:#e65100,color:#000
    style IN fill:#e8f5e9,stroke:#1b5e20,color:#000
    style SG fill:#e8f5e9,stroke:#1b5e20,color:#000
    style DE fill:#e8f5e9,stroke:#1b5e20,color:#000
    style BR fill:#e8f5e9,stroke:#1b5e20,color:#000
    style JP fill:#e8f5e9,stroke:#1b5e20,color:#000
    style OS1 fill:#fce4ec,stroke:#c62828,color:#000
    style OS2 fill:#fce4ec,stroke:#c62828,color:#000
    style ALB1 fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style ALB2 fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style RDSR1 fill:#e0f2f1,stroke:#004d40,color:#000
    style RDSR2 fill:#e0f2f1,stroke:#004d40,color:#000
    style ECS1 fill:#fff8e1,stroke:#f9a825,color:#000
    style ECS2 fill:#fff8e1,stroke:#f9a825,color:#000
    style RDSP fill:#fce4ec,stroke:#c62828,color:#000
    style AURORA_GD fill:#fff3e0,stroke:#e65100,color:#000
```

---

## 4. Interview Preparation: Multi-Level QA

### System Design Challenge

**Question:** Design a global CDN and edge computing platform that supports dynamic content personalization at the edge. Users in different geographies should see localized content, and authenticated users should see personalized dashboards — all without hitting the origin server.

**Optimal Blueprint:**

```mermaid
flowchart TD
    USER["User Request"]
    EDGE_LOC["CloudFront Edge Location"]
    subgraph EF["Edge Functions"]
        CF_FUNC["CloudFront Functions<br/>• Parse JWT from Cookie<br/>• Extract user_id, locale<br/>• Modify cache key: /content/{locale}/{user_tier}/{path}<br/>• Set X-Cache-Key header"]
    end
    REG_CACHE["Regional Edge Cache<br/>Read-through from S3 + Lambda@Edge<br/>Cache key includes user_tier (not user_id)"]
    LAMBDA["Lambda@Edge (Origin Request)<br/>• Fetch user prefs from DynamoDB Global Tables<br/>• Generate personalized HTML fragment<br/>• Return to regional cache (60s TTL)<br/>• Cache-Control: private"]
    S3["S3 Origin: Static Content"]
    ALB_ORIGIN["ALB Origin: API Content"]
    DDB["DynamoDB Global Tables: User Prefs"]

    USER --> EDGE_LOC
    EDGE_LOC --> CF_FUNC
    EDGE_LOC --> REG_CACHE
    REG_CACHE --> LAMBDA
    LAMBDA --> S3
    LAMBDA --> ALB_ORIGIN
    LAMBDA --> DDB

    style USER fill:#e1f5fe,stroke:#01579b,color:#000
    style EDGE_LOC fill:#fff3e0,stroke:#e65100,color:#000
    style CF_FUNC fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style REG_CACHE fill:#fce4ec,stroke:#c62828,color:#000
    style LAMBDA fill:#e8f5e9,stroke:#1b5e20,color:#000
    style S3 fill:#fff8e1,stroke:#f9a825,color:#000
    style ALB_ORIGIN fill:#e0f2f1,stroke:#004d40,color:#000
    style DDB fill:#fce4ec,stroke:#c62828,color:#000
```

Key design decisions:
- **Cache key includes user tier but not user ID** — all users in the same tier share cached content.
- **Stale-while-revalidate = 86400** — serve stale content for up to 24 hours if origin is down.
- **Private Cache-Control** for personalized fragments ensures they are not cached at shared edges.

#### 🟢 Basic Level (Jr. Engineer / 0-2 Yrs)

**Q1: What is a CDN and how does it improve website performance?**
**A1:** A Content Delivery Network (CDN) is a globally distributed network of proxy servers that cache content close to users. It improves performance by: (1) reducing latency — content is served from a nearby edge location instead of a distant origin server; (2) reducing origin load — cached requests never reach the origin; (3) handling traffic spikes — the CDN absorbs sudden traffic increases. Without a CDN, a user in India accessing a US-based server experiences 200-300ms latency vs. 5-20ms with a local CDN edge.

**Q2: What is the difference between a data center Tier 3 and Tier 4?**
**A2:** Tier 3 provides N+1 redundancy with 99.982% availability (~1.6 hours downtime/year). It is concurrently maintainable — any component can be taken offline without disruption. Tier 4 provides 2N+1 fault-tolerant redundancy with 99.995% availability (~26 minutes downtime/year). Every component is dual-powered with automatic failover and zero single points of failure. Tier 4 costs 2-3x more per square foot than Tier 3.

#### 🟡 Intermediate Level (Mid-Level / 2-5 Yrs)

**Q1: How do you configure CloudFront to cache API responses?**
**A1:**
```hcl
resource "aws_cloudfront_distribution" "api" {
  default_cache_behavior {
    target_origin_id       = "APIOrigin"
    viewer_protocol_policy = "https-only"
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    compress               = true
    forwarded_values {
      query_string = true
      cookies {
        forward = "whitelist"
        whitelisted_names = ["session"]
      }
    }
    min_ttl     = 0
    default_ttl = 60        # Cache for 60 seconds
    max_ttl     = 3600      # Max cache 1 hour
  }
  # Origin Shield for request collapsing
  origin {
    origin_shield {
      enabled              = true
      origin_shield_region = "ap-south-1"
    }
  }
}
```

**Q2: Explain Cache-Control headers: public, private, no-cache, no-store, must-revalidate.**
**A2:**
- `public`: Can be cached by CDNs and browsers
- `private`: Cacheable only by the browser (not CDNs) — for personalized content
- `no-cache`: Must revalidate with origin before serving cached copy (can still cache, just checks freshness)
- `no-store`: Never cache at all — for sensitive data
- `must-revalidate`: Origin must be checked when cached content is stale
- `s-maxage`: CDN-specific max-age (overrides max-age for shared caches)
- Example: `Cache-Control: public, max-age=300, s-maxage=600, stale-while-revalidate=86400`

#### 🔴 Advanced Level (Senior / 5-8 Yrs)

**Q1: Design a multi-region active-active architecture with CDN and global database replication.**
**A1:**
```mermaid
flowchart TD
    R53_GA["Route53 Geolocation Routing"]
    subgraph NA["North America (us-east-1)"]
        CF_NA["CloudFront NA"]
        ALB_NA["ALB"]
        ECS_NA["ECS Fargate"]
        AURORA_NA["Aurora (Read/Write)"]
    end
    subgraph EU["Europe (eu-west-1)"]
        CF_EU["CloudFront EU"]
        ALB_EU["ALB"]
        ECS_EU["ECS Fargate"]
        AURORA_EU["Aurora (Read Replica)"]
    end
    subgraph APAC["Asia Pacific (ap-south-1)"]
        CF_APAC["CloudFront APAC"]
        ALB_APAC["ALB"]
        ECS_APAC["ECS Fargate"]
        AURORA_APAC["Aurora (Read Replica)"]
    end
    AURORA_NA <-->|Aurora Global Database<br/>Async replication| AURORA_EU
    AURORA_NA <-->|Aurora Global Database<br/>Async replication| AURORA_APAC

    USERS["Global Users"] --> R53_GA
    R53_GA --> CF_NA
    R53_GA --> CF_EU
    R53_GA --> CF_APAC

    style R53_GA fill:#e1f5fe,stroke:#01579b,color:#000
    style CF_NA fill:#fff3e0,stroke:#e65100,color:#000
    style CF_EU fill:#fff3e0,stroke:#e65100,color:#000
    style CF_APAC fill:#fff3e0,stroke:#e65100,color:#000
    style ALB_NA fill:#fce4ec,stroke:#c62828,color:#000
    style ECS_NA fill:#e8f5e9,stroke:#1b5e20,color:#000
    style AURORA_NA fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style ALB_EU fill:#fce4ec,stroke:#c62828,color:#000
    style ECS_EU fill:#e8f5e9,stroke:#1b5e20,color:#000
    style AURORA_EU fill:#f3e5f5,stroke:#6a1b9a,color:#000
    style ALB_APAC fill:#fce4ec,stroke:#c62828,color:#000
    style ECS_APAC fill:#e8f5e9,stroke:#1b5e20,color:#000
    style AURORA_APAC fill:#f3e5f5,stroke:#6a1b9a,color:#000
```

Key design decisions:
- Route53 geolocation routing directs users to the nearest region
- CloudFront for static caching + regional edge caches for API content
- Each region has its own read replica; writes go to the primary (us-east-1)
- Writes are async-replicated via Aurora Global Database
- Cache TTLs are staggered — shorter at edge, longer at regional cache
- `stale-while-revalidate` of 86400s provides resilience during origin failures

**Q2: How would you handle a cache stampede where thousands of CDN edges simultaneously request the same new content from origin?**
**A2:** Cache stampede (aka thundering herd) happens when popular content expires simultaneously across all edges. Mitigation strategies:

1. **Origin Shield with request collapsing:** Only the first edge's request reaches origin; subsequent edges wait for the same response. Reduces origin load by 10x+.
2. **Staggered TTLs:** Add random jitter to TTL (`s-maxage: 3600 ± 300`). Edges expire at different times, spreading the load.
3. **Stale-while-revalidate:** Serve stale content while fetching fresh version in background. Zero penalty for users.
4. **Pre-warming:** Before a flash sale, proactively push content to all CDN edges via CDN预热 APIs (supported by CloudFront, Akamai, Fastly).
5. **Proactive cache refresh:** Use Lambda@Edge to detect traffic spikes and pre-fetch popular content before mass expiration.

#### ⚫ Expert Level (Staff/Principal / 8+ Yrs)

**Q1: Explain how Anycast routing works at the BGP level for CloudFront and how traffic is steered to the correct edge location. What happens during a BGP hijack?**
**A1:** Anycast routing works by advertising the same IP prefix (e.g., 13.224.0.0/14) from multiple geographically distributed locations simultaneously.

**BGP mechanics:**
1. CloudFront edge locations (350+ PoPs) each announce the IP prefix to their upstream ISPs via eBGP.
2. Each announcement has a different AS-path length depending on the ISP's routing table. The ISP selects the route with the shortest AS-path.
3. The user's ISP propagates the best route to its customers. The user's packets are routed to the nearest edge location in BGP terms (which may differ from geographic proximity due to peering/policy).

**Connection flow:**
- `traceroute` shows the path: User → ISP router → Transit → CloudFront edge router
- The edge router terminates the TCP connection and performs DSR (Direct Server Return) — response packets go directly to the user without backhauling to origin
- The router uses ECMP to distribute connections across edge servers within the PoP

**BGP hijack scenario:**
An attacker advertises a more specific prefix (e.g., 13.224.0.0/24) or the same prefix with a shorter AS-path. Traffic is redirected to the attacker's infrastructure. Mitigations:
- **RPKI (Resource Public Key Infrastructure):** Route Origin Authorization (ROA) cryptographically signs the origin AS that is authorized to announce a prefix. ISPs implementing RPKI validation (e.g., Cloudflare, Comcast) reject invalid announcements. As of 2026, ~60% of the internet validates RPKI.
- **BGP Flowspec:** Drops traffic from the hijacked prefix at the network edge.
- **Anycast inherently limits blast radius:** Only users routed to the affected ISP are impacted.

**Q2: Describe the full request flow through a multi-tier cache hierarchy (CloudFront Edge → Regional Cache → Origin Shield → Origin). How does request collapsing prevent the thundering herd problem?**
**A2:** The cache hierarchy has four tiers:

```mermaid
flowchart TD
    USER2["User Request"]
    EDGE_LOCAL["Edge Location<br/>Local cache (SSD/NVMe, 1-10TB)<br/>Fastest, smallest cache"]
    REGIONAL2["Regional Edge Cache<br/>Larger cache (100s of GB)<br/>Serves multiple edge locations"]
    SHIELD2["Origin Shield<br/>Single-region cache<br/>Request collapsing<br/>Shock absorber for origin"]
    ORIGIN2["Origin Server<br/>ALB → ECS → Database"]

    USER2 -->|1. Route to nearest edge| EDGE_LOCAL
    EDGE_LOCAL -->|2. Cache miss| REGIONAL2
    REGIONAL2 -->|3. Cache miss| SHIELD2
    SHIELD2 -->|4. First request only| ORIGIN2
    ORIGIN2 -->|5. Response| SHIELD2
    SHIELD2 -->|6. Cache + respond| REGIONAL2
    REGIONAL2 -->|7. Cache + respond| EDGE_LOCAL
    EDGE_LOCAL -->|8. Serve user| USER2

    style USER2 fill:#e1f5fe,stroke:#01579b,color:#000
    style EDGE_LOCAL fill:#fff3e0,stroke:#e65100,color:#000
    style REGIONAL2 fill:#fce4ec,stroke:#c62828,color:#000
    style SHIELD2 fill:#e8f5e9,stroke:#1b5e20,color:#000
    style ORIGIN2 fill:#f3e5f5,stroke:#6a1b9a,color:#000
```

**Request collapsing internals:**
When 100 edges simultaneously request the same object and all miss:
1. Each edge sends a request to the regional cache. If all miss there too:
2. Each regional cache forwards to Origin Shield.
3. Origin Shield maintains a **pending request hash table**. The first request creates an entry; subsequent requests **park** in a wait-list (implemented as a condition variable or completion queue).
4. When the first response arrives from origin, the response is written to the Origin Shield cache, then all parked connections are unblocked and served the cached response.
5. The origin receives exactly **1 request** instead of 100.

**Performance characteristics:**
- Edge cache: ~1ms hit, 50-100ms miss (to regional)
- Regional cache: ~5ms hit, 100-200ms miss (to Origin Shield)
- Origin Shield: ~10ms hit, 200-500ms miss (to origin)
- Request collapsing: Saves 99% of origin load during stampede events

**Memory architecture of Varnish (used in Origin Shield):**
- Shared memory hash table with RCU (lock-free reads)
- Busy object pattern: creates an in-flight marker atomically
- Worker threads use non-blocking hash table lookups (~50M ops/sec)
- LRU eviction runs in dedicated thread with stop-the-world for structural changes only
