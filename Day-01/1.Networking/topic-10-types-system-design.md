# Topic 10: Types, Patterns & Architectures in System Design

## 1. Theoretical Foundation & System Mechanics

### The Core Concept

System design is not monolithic — it encompasses multiple levels of abstraction (HLD vs LLD), multiple communication paradigms (sync vs async, request-response vs event-driven), and multiple API styles (REST, gRPC, GraphQL). Mastering these categories allows an architect to select the right tool for each problem.

**High-Level Design (HLD):**

HLD is the **blueprint** of the system. It answers: "What are the components, and how do they interact?"

```mermaid
flowchart TD
    CL["Client Apps"]:::client
    CDN["CDN"]:::cdn
    GW["API Gateway"]:::gw
    AUTH["Auth Service"]:::svc
    LB["Load Balancer"]:::lb
    PS["Product Service"]:::svc
    OS["Order Service"]:::svc
    PAY["Payment Service"]:::svc
    PDB[("Product DB")]:::db
    ODB[("Order DB")]:::db
    PAYDB[("Payment DB")]:::db
    MQ["Message Queue (Kafka)"]:::mq
    NS["Notification Service"]:::svc
    OUT["Email / SMS / Push"]:::out

    CL --> CDN --> GW --> AUTH --> LB
    LB --> PS & OS & PAY
    PS --> PDB
    OS --> ODB
    PAY --> PAYDB
    PS & OS & PAY --> MQ
    MQ --> NS
    NS --> OUT

    classDef client fill:#e1f5fe,stroke:#01579b,color:#000,stroke-width:2px
    classDef cdn fill:#fff3e0,stroke:#e65100,color:#000
    classDef gw fill:#f3e5f5,stroke:#4a148c,color:#000
    classDef svc fill:#e8f5e9,stroke:#1b5e20,color:#000
    classDef lb fill:#e1f5fe,stroke:#01579b,color:#000
    classDef db fill:#fce4ec,stroke:#b71c1c,color:#000
    classDef mq fill:#fff3e0,stroke:#e65100,color:#000
    classDef out fill:#f3e5f5,stroke:#4a148c,color:#000


HLD deliverables:
- Architecture diagram (boxes and arrows)
- Component descriptions (1-2 paragraphs each)
- Data flow diagrams
- Technology stack decisions
- API contract overview
- Scaling strategy

**Low-Level Design (LLD):**

LLD is the **specification** of each component. It answers: "How exactly does each component implement its responsibilities?"

```mermaid
classDiagram
    class OrderService {
        +createOrder()
        +getOrder(id)
        +cancelOrder(id)
        +updateStatus()
    }
    class OrderRepository {
        +save(Order)
        +findById(id)
        +findByUser(id)
    }
    class JPA {
        <<interface>>
    }
    class Order {
        -id: UUID
        -userId: UUID
        -items: List
        -status: Enum
        -total: BigDecimal
        -createdAt: Instant
        -version: Long
    }
    class order_table {
        id: UUID (PK)
        user_id: UUID (FK)
        status: VARCHAR
        total: DECIMAL
        created_at: TS
        version: INT
    }

    OrderService --> OrderRepository
    OrderRepository ..|> JPA
    OrderRepository --> order_table
    OrderService --> Order
    JPA --> order_table


LLD deliverables:
- Class diagrams (UML)
- Database schemas (SQL)
- API contracts (OpenAPI 3.0)
- Sequence diagrams for key flows
- Error handling specifications
- State machine diagrams

**Request-Response vs Event-Driven vs Streaming:**

```mermaid
flowchart LR
    subgraph REQRES["Request-Response (Synchronous)"]
        direction LR
        C1["Client"]:::client -->|"REQ"| S1["Server"]:::server
        S1 -->|"RESP"| C1
        B1["Client blocks until response"]:::note
    end

    subgraph EVDR["Event-Driven (Asynchronous)"]
        direction LR
        P["Producer"]:::prod -->|"Event"| EB["Event Bus"]:::bus
        EB -->|"Event"| C2["Consumer(s)"]:::cons
        N1["Producer does not wait"]:::note
    end

    subgraph STREAM["Streaming (Continuous)"]
        direction LR
        P2["Producer"]:::prod -->|"Stream of Events"| C3["Consumer"]:::cons
        N2["Persistent connection<br/>Kafka / WebSocket / SSE"]:::note
    end

    classDef client fill:#e1f5fe,stroke:#01579b,color:#000,stroke-width:2px
    classDef server fill:#fff3e0,stroke:#e65100,color:#000
    classDef prod fill:#e8f5e9,stroke:#1b5e20,color:#000
    classDef bus fill:#f3e5f5,stroke:#4a148c,color:#000
    classDef cons fill:#fce4ec,stroke:#b71c1c,color:#000
    classDef note fill:#e1f5fe,stroke:#01579b,color:#000,stroke-dasharray: 5 5


**Synchronous vs Asynchronous Communication:**

```mermaid
flowchart LR
    subgraph Sync["Synchronous"]
        direction LR
        SA["Service A"]:::svc
        SB["Service B"]:::svc
        SA -->|"HTTP/gRPC<br/>Wait + Block"| SB
        SB -->|"Response"| SA
    end

    subgraph Async["Asynchronous"]
        direction LR
        AS["Service A"]:::svc
        Q[("Queue (Kafka)")]:::queue
        BS["Service B"]:::svc
        AS -->|Publish| Q
        Q -->|Consume| BS
    end

    classDef svc fill:#e1f5fe,stroke:#01579b,color:#000,stroke-width:2px
    classDef queue fill:#fff3e0,stroke:#e65100,color:#000,stroke-width:2px


| Aspect | Synchronous | Asynchronous |
|--------|-------------|--------------|
| Latency | Lower (single hop) | Higher (queue overhead) |
| Coupling | Tight (both must be up) | Loose (queue buffers) |
| Error handling | Immediate | Retry with DLQ |
| Debugging | Easier (linear flow) | Harder (distributed tracing) |
| Throughput | Limited by slowest service | Decoupled, higher |

### The "Why"

These patterns solve the **coupling bottleneck**. In a traditional monolithic request-response system, every component must be available for the system to function. Event-driven and asynchronous patterns allow components to fail independently, scale independently, and evolve independently.

**The bottleneck of synchronous chains:**

```mermaid
flowchart LR
    subgraph SyncChain["Synchronous Chain"]
        SA["Req → A<br/>2ms"]:::sync
        SB["B<br/>5ms"]:::sync
        SC["C<br/>10ms"]:::sync
        SD["D<br/>3ms"]:::sync
        SR["Response"]:::sync
        SA --> SB --> SC --> SD --> SR
        NOTE1["Total: 20ms<br/>If B goes down → 500 Error<br/>Throughput = 0"]:::fail
    end

    subgraph AsyncChain["Asynchronous Chain"]
        AA["Req → A<br/>200ms"]:::async
        AQ1[("Queue")]:::queue
        AB["B<br/>(eventual)"]:::async
        AQ2[("Queue")]:::queue
        AC["C<br/>(eventual)"]:::async
        AQ3[("Queue")]:::queue
        AD["D<br/>(eventual)"]:::async
        AA --> AQ1 --> AB --> AQ2 --> AC --> AQ3 --> AD
        NOTE2["Throughput = independent per service<br/>B down? Queue buffers. B recovers? Queue drains."]:::ok
    end

    classDef sync fill:#fce4ec,stroke:#b71c1c,color:#000
    classDef fail fill:#fce4ec,stroke:#b71c1c,color:#000,stroke-dasharray: 5 5
    classDef async fill:#e8f5e9,stroke:#1b5e20,color:#000
    classDef queue fill:#fff3e0,stroke:#e65100,color:#000
    classDef ok fill:#e8f5e9,stroke:#1b5e20,color:#000,stroke-dasharray: 5 5


### Trade-offs

| Pattern | Hidden Cost | Mitigation |
|---------|-------------|------------|
| **Event-Driven** | Eventual consistency, duplicate events | Idempotency keys, exactly-once semantics |
| **gRPC** | Binary protocol (hard to debug), HTTP/2 complexity | gRPC reflection, Envoy sidecar |
| **GraphQL** | N+1 queries, complex caching | DataLoader, persisted queries |
| **CQRS** | Two models to maintain, eventual consistency | Event sourcing for audit trail |
| **Event Sourcing** | Event store growth, replay time | Snapshotting, compaction |
| **SAGA** | Compensating transactions complexity | Choreography-based SAGA for simplicity |

---

## 2. Production Implementation (Full Stack & Cloud)

### Backend & Code Architecture

**RESTful API Design (OpenAPI 3.0):**

```yaml
openapi: 3.0.3
info:
  title: Project Management API
  version: 2.1.0
  description: Enterprise project tracking with AI assistant integration

paths:
  /api/v2/projects:
    get:
      summary: List projects with pagination and filtering
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: size
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
        - name: status
          in: query
          schema:
            type: string
            enum: [ACTIVE, ARCHIVED, COMPLETED]
        - name: cursor
          in: query
          schema:
            type: string
          description: Cursor-based pagination token
      responses:
        '200':
          description: Paginated list of projects
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Project'
                  pagination:
                    type: object
                    properties:
                      nextCursor:
                        type: string
                      total:
                        type: integer
            application/x-protobuf:
              schema:
                $ref: '#/components/schemas/ProjectList'
    post:
      summary: Create a new project
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateProjectRequest'
      responses:
        '201':
          description: Project created
          headers:
            Location:
              schema:
                type: string
                format: uri

components:
  schemas:
    Project:
      type: object
      required: [id, name, status]
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
          minLength: 3
          maxLength: 255
        description:
          type: string
          maxLength: 5000
        status:
          type: string
          enum: [ACTIVE, ARCHIVED, COMPLETED]
        ownerId:
          type: string
          format: uuid
        createdAt:
          type: string
          format: date-time
        version:
          type: integer
          description: Optimistic locking version
```

**gRPC Service Definition:**

```protobuf
syntax = "proto3";

package taskmanager.v1;

option java_package = "com.example.taskmanager.grpc";
option java_multiple_files = true;

service TaskService {
    rpc GetTask (GetTaskRequest) returns (Task);
    rpc ListTasks (ListTasksRequest) returns (ListTasksResponse);
    rpc CreateTask (CreateTaskRequest) returns (Task);
    rpc UpdateTask (UpdateTaskRequest) returns (Task);
    rpc WatchTaskUpdates (WatchTaskRequest) returns (stream TaskEvent);
}

message Task {
    string id = 1;
    string project_id = 2;
    string title = 3;
    TaskStatus status = 4;
    int32 priority = 5;
    string assignee_id = 6;
    google.protobuf.Timestamp due_date = 7;
    map<string, string> custom_fields = 8;
    int32 version = 9;
}

enum TaskStatus {
    TASK_STATUS_UNSPECIFIED = 0;
    TASK_STATUS_TODO = 1;
    TASK_STATUS_IN_PROGRESS = 2;
    TASK_STATUS_IN_REVIEW = 3;
    TASK_STATUS_DONE = 4;
    TASK_STATUS_BLOCKED = 5;
}

message WatchTaskRequest {
    string project_id = 1;
    repeated TaskStatus filter_status = 2;
}

message TaskEvent {
    EventType type = 1;
    Task task = 2;
    google.protobuf.Timestamp timestamp = 3;

    enum EventType {
        EVENT_TYPE_CREATED = 0;
        EVENT_TYPE_UPDATED = 1;
        EVENT_TYPE_DELETED = 2;
        EVENT_TYPE_STATUS_CHANGED = 3;
    }
}

service HealthService {
    rpc Check (HealthCheckRequest) returns (HealthCheckResponse);
}
```

**CQRS Pattern Implementation:**

```java
// COMMAND SIDE (Writes)
@RestController
@RequestMapping("/api/commands/tasks")
public class TaskCommandController {

    private final CommandBus commandBus;

    @PostMapping
    public CompletableFuture<UUID> createTask(@Valid @RequestBody CreateTaskCommand cmd) {
        return commandBus.dispatch(cmd);
    }

    @PostMapping("/{id}/assign")
    public CompletableFuture<Void> assignTask(@PathVariable UUID id,
                                               @RequestBody AssignTaskCommand cmd) {
        cmd.setTaskId(id);
        return commandBus.dispatch(cmd);
    }
}

// QUERY SIDE (Reads)
@RestController
@RequestMapping("/api/queries/tasks")
public class TaskQueryController {

    private final TaskReadRepository readRepo;

    @GetMapping("/{id}")
    public TaskDetail getTaskDetail(@PathVariable UUID id) {
        return readRepo.findById(id)
            .orElseThrow(() -> new NotFoundException("Task not found"));
    }

    @GetMapping
    public Page<TaskSummary> searchTasks(TaskSearchCriteria criteria, Pageable pageable) {
        return readRepo.search(criteria, pageable);
    }
}

// EVENT HANDLER — Syncs read model from command events
@Component
public class TaskEventHandler {

    private final TaskReadRepository readRepo;

    @EventListener
    public void on(TaskCreatedEvent event) {
        readRepo.save(new TaskDetail(
            event.getTaskId(),
            event.getTitle(),
            event.getStatus(),
            event.getAssigneeId(),
            event.getCreatedAt(),
            event.getVersion()
        ));
    }

    @EventListener
    public void on(TaskAssignedEvent event) {
        readRepo.updateAssignee(event.getTaskId(), event.getAssigneeId());
    }
}
```

**SAGA Pattern for Distributed Transactions:**

```java
// Choreography-based SAGA for order processing
@Service
public class OrderSagaOrchestrator {

    @Autowired
    private KafkaTemplate<String, Object> kafka;

    @Transactional
    public void createOrder(CreateOrderCommand cmd) {
        // Step 1: Create order in PENDING status
        Order order = orderRepository.save(new Order(cmd));

        // Step 2: Publish event to start saga
        kafka.send("order-events", new OrderCreatedEvent(order.getId(), order.getItems()));
    }
}

@Component
public class InventoryServiceSaga {

    @KafkaListener(topics = "order-events")
    public void handle(OrderCreatedEvent event) {
        try {
            // Reserve inventory
            inventoryService.reserve(event.getOrderId(), event.getItems());
            // If successful, emit next event
            kafka.send("order-events", new InventoryReservedEvent(event.getOrderId()));
        } catch (InsufficientInventoryException e) {
            // Compensating action
            kafka.send("order-events", new InventoryReservationFailedEvent(
                event.getOrderId(), e.getMessage()));
        }
    }
}

@Component
public class PaymentServiceSaga {

    @KafkaListener(topics = "order-events")
    public void handle(InventoryReservedEvent event) {
        try {
            paymentService.charge(event.getOrderId());
            kafka.send("order-events", new PaymentCompletedEvent(event.getOrderId()));
        } catch (PaymentException e) {
            // Compensating action: release inventory
            kafka.send("inventory-commands", new ReleaseInventoryCommand(event.getOrderId()));
            kafka.send("order-events", new PaymentFailedEvent(event.getOrderId()));
        }
    }
}
```

### DevOps & Infrastructure

**Kubernetes for Microservices Deployment:**

```yaml
# Service definition with canary strategy
apiVersion: v1
kind: Service
metadata:
  name: task-service
spec:
  selector:
    app: task-service
  ports:
    - name: grpc
      port: 9090
      targetPort: 9090
    - name: metrics
      port: 9091
      targetPort: 9091
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: task-service-stable
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: task-service
      track: stable
  template:
    metadata:
      labels:
        app: task-service
        track: stable
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: task-service
          image: myregistry/task-service:2.1.0
          ports:
            - containerPort: 9090
            - containerPort: 9091
          readinessProbe:
            grpc:
              port: 9090
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 9091
            initialDelaySeconds: 30
            periodSeconds: 15
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          env:
            - name: DB_CONNECTION_STRING
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: connection-string
            - name: KAFKA_BOOTSTRAP_SERVERS
              value: "kafka-cluster:9092"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: task-service-canary
spec:
  replicas: 2
  selector:
    matchLabels:
      app: task-service
      track: canary
  template:
    metadata:
      labels:
        app: task-service
        track: canary
    spec:
      containers:
        - name: task-service
          image: myregistry/task-service:2.2.0-rc.1
          # ... same probe and resource config ...
---
# Service mesh traffic splitting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: task-service-routing
spec:
  hosts:
    - task-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: task-service
            subset: canary
          weight: 100
    - route:
        - destination:
            host: task-service
            subset: stable
          weight: 95
        - destination:
            host: task-service
            subset: canary
          weight: 5
```

**Terraform for Event-Driven Infrastructure:**

```hcl
resource "aws_msk_cluster" "kafka" {
  cluster_name           = "production-events"
  kafka_version          = "3.5.1"
  number_of_broker_nodes = 3

  broker_node_group_info {
    instance_type   = "kafka.m5.large"
    client_subnets  = aws_subnet.private[*].id
    security_groups = [aws_security_group.kafka.id]
    storage_info {
      ebs_storage_info {
        volume_size = 1000
      }
    }
  }

  configuration_info {
    arn      = aws_msk_configuration.kafka_config.arn
    revision = aws_msk_configuration.kafka_config.latest_revision
  }

  encryption_info {
    encryption_in_transit {
      client_broker = "TLS"
      in_cluster    = true
    }
  }

  open_monitoring {
    prometheus {
      jmx_exporter {
        enabled_in_broker = true
      }
      node_exporter {
        enabled_in_broker = true
      }
    }
  }

  logging_info {
    broker_logs {
      cloudwatch_logs {
        enabled   = true
        log_group = "msk-broker-logs"
      }
    }
  }

  tags = {
    Environment = "production"
    Team        = "platform"
  }
}

resource "aws_msk_configuration" "kafka_config" {
  name           = "production-config"
  kafka_versions = ["3.5.1"]

  server_properties = <<PROPERTIES
auto.create.topics.enable = false
default.replication.factor = 3
min.insync.replicas = 2
num.io.threads = 8
num.network.threads = 8
log.retention.hours = 168
log.segment.bytes = 1073741824
log.retention.bytes = 107374182400
PROPERTIES
}

resource "aws_sqs_queue" "dlq" {
  name                        = "event-dlq"
  delay_seconds               = 90
  max_message_size            = 262144
  message_retention_seconds   = 1209600
  receive_wait_time_seconds   = 20
  visibility_timeout_seconds  = 300

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 5
  })
}
```

### Cloud Architecture

```mermaid
flowchart TD
    CL["Client Apps"]:::client
    GW["API Gateway"]:::gw
    CMD["Command Service"]:::cmd
    WDB[("Write DB (Event Store)"):::wdb]
    KAFKA["Kafka Event Bus"]:::kafka
    CU["Cache Updater"]:::proc
    SI["Search Indexer"]:::proc
    NTF["Notification"]:::proc
    ALY["Analytics"]:::proc
    RMB["Read Model Builder"]:::proc
    REDIS[("Redis")]:::store
    ES[("Elasticsearch")]:::store
    SNS["SES/SNS"]:::store
    S3[("S3 / Athena")]:::store
    RDB[("Read DB")]:::store
    GQL["GraphQL API"]:::api
    CL2["Client Apps"]:::client

    CL --> GW --> CMD
    CMD --> WDB
    WDB --> KAFKA
    KAFKA --> CU & SI & NTF & ALY & RMB
    CU --> REDIS
    SI --> ES
    NTF --> SNS
    ALY --> S3
    RMB --> RDB --> GQL --> CL2

    classDef client fill:#e1f5fe,stroke:#01579b,color:#000,stroke-width:2px
    classDef gw fill:#fff3e0,stroke:#e65100,color:#000
    classDef cmd fill:#f3e5f5,stroke:#4a148c,color:#000,stroke-width:2px
    classDef wdb fill:#fce4ec,stroke:#b71c1c,color:#000
    classDef kafka fill:#e8f5e9,stroke:#1b5e20,color:#000,stroke-width:2px
    classDef proc fill:#e1f5fe,stroke:#01579b,color:#000
    classDef store fill:#fff3e0,stroke:#e65100,color:#000
    classDef api fill:#f3e5f5,stroke:#4a148c,color:#000


---

## 3. Real-World Scaling Scenarios

### The Bottleneck

**Scenario:** A SaaS analytics platform ingests 500K events/second from customer integrations. The original synchronous REST API cannot keep up. Database writes bottleneck at 50K inserts/second.

```mermaid
flowchart TD
    EP["Event Producer"]:::prod
    API["API Server"]:::api
    DB[("PostgreSQL")]:::db
    RETRY["Client retries<br/>(already saturated)"]:::prod
    FAIL["503 Service Unavailable"]:::fail

    EP -->|"POST → Wait for 200 OK<br/>Timeout after 5s"| API
    API -->|"INSERT"| DB
    DB -->|"DB at 100% CPU<br/>Connection pool full"| API
    API -->|"⛔ Saturated"| RETRY
    RETRY -->|"Back to saturated server"| API
    API --> FAIL

    subgraph OBSERVED["Observed Failure Mode"]
        O1["• 50% error rate on producers"]:::obs
        O2["• API servers at 200% CPU"]:::obs
        O3["• DB deadlocks from concurrent inserts"]:::obs
        O4["• Backpressure causes cascading failures"]:::obs
    end

    FAIL --> OBSERVED

    classDef prod fill:#fce4ec,stroke:#b71c1c,color:#000
    classDef api fill:#fff3e0,stroke:#e65100,color:#000
    classDef db fill:#fce4ec,stroke:#b71c1c,color:#000
    classDef fail fill:#b71c1c,stroke:#b71c1c,color:#fff,stroke-width:3px
    classDef obs fill:#fce4ec,stroke:#b71c1c,color:#000,stroke-dasharray: 5 5


### The Solution

**Step 1: Buffer writes with Kafka**

```mermaid
flowchart LR
    EP["Event Producer"]:::prod
    API["API Server"]:::api
    KAFKA["Kafka (Partition 0-31)<br/>Retention: 7 days"]:::kafka
    CG["Kafka Consumer Group"]:::cg
    TDB["TimescaleDB<br/>50K → 500K writes/sec<br/>via batching"]:::db

    EP -->|"POST"| API
    API -->|"202 Accepted (immediate)"| EP
    API -->|"PUBLISH"| KAFKA
    KAFKA -->|"Kafka Buffer"| CG
    CG -->|"Batch insert every 5s"| TDB

    classDef prod fill:#e1f5fe,stroke:#01579b,color:#000
    classDef api fill:#fff3e0,stroke:#e65100,color:#000
    classDef kafka fill:#f3e5f5,stroke:#4a148c,color:#000,stroke-width:2px
    classDef cg fill:#e8f5e9,stroke:#1b5e20,color:#000
    classDef db fill:#fce4ec,stroke:#b71c1c,color:#000,stroke-width:2px


**Step 2: Implement backpressure and circuit breakers**

```java
@Component
public class EventIngestionService {

    private final KafkaTemplate<String, Event> kafka;

    @CircuitBreaker(name = "kafkaPublisher", fallbackMethod = "publishFallback")
    public CompletableFuture<Void> publishEvent(Event event) {
        return kafka.send("raw-events", event.getCustomerId(), event)
            .completable()
            .thenAccept(metadata -> {
                if (metadata.hasError()) {
                    Metrics.counter("kafka.publish.error").increment();
                }
            });
    }

    // If Kafka is down, buffer to S3
    public CompletableFuture<Void> publishFallback(Event event, Throwable t) {
        return CompletableFuture.runAsync(() -> {
            s3Client.putObject(
                "event-buffer",
                "backlog/" + Instant.now().toString() + "-" + event.getId() + ".json",
                event.toJson()
            );
        });
    }
}
```

**Step 3: Move to CQRS with materialized views**

```mermaid
flowchart TD
    TOPIC["Kafka Topic: customer-analytics"]:::topic
    P0["Partition 0: Customer A-D"]:::part
    P1["Partition 1: Customer E-H"]:::part
    PX["... 32 partitions total"]:::part
    RV["Real-time Aggregations<br/>(Redis)<br/>TTL: 1 hour"]:::view
    HV["Hourly Aggregations<br/>(ClickHouse)<br/>Retention: 30d"]:::view
    DV["Daily Aggregations<br/>(S3 / Athena)<br/>Retention: 365d"]:::view

    TOPIC --> P0 & P1 & PX
    P0 & P1 & PX --> RV & HV & DV

    classDef topic fill:#fce4ec,stroke:#b71c1c,color:#000,stroke-width:3px
    classDef part fill:#fff3e0,stroke:#e65100,color:#000
    classDef view fill:#e8f5e9,stroke:#1b5e20,color:#000,stroke-width:2px


---

## 4. Senior-Level Interview Deep Dive

### System Design Challenge

**Question:** Design a real-time collaborative document editing platform (like Google Docs) supporting 10,000 concurrent users on a single document with sub-second sync latency. Must handle conflicts, offline edits, and provide version history.

**Optimal Blueprint:**

```mermaid
flowchart TD
    CA["Client A"]:::client
    CB["Client B"]:::client
    CC["Client C"]:::client
    CS["Collaboration Service"]:::svc
    OPLOG["Operation Log (Kafka)"]:::kafka
    OT["OT Transform Engine"]:::engine
    DS[("Document Store (DynamoDB)")]:::db
    SS[("Snapshot Service (S3)")]:::s3

    CA & CB & CC -->|"WebSocket"| CS
    CS --> OPLOG --> OT --> DS
    DS --> SS

    subgraph CONFLICT["Conflict Resolution"]
        OT_METHOD["OT: Transform concurrent ops<br/>CRDT: Mergeable data structures"]:::method
    end

    subgraph SCALING["Scaling"]
        S1["Document sharded by doc_id"]:::scale
        S2["Each node: ~100 concurrent docs"]:::scale
        S3["In-memory state (Redis) + DynamoDB"]:::scale
        S4["WebSocket: sticky sessions"]:::scale
    end

    OT --> CONFLICT
    CS --> SCALING

    classDef client fill:#e1f5fe,stroke:#01579b,color:#000,stroke-width:2px
    classDef svc fill:#fff3e0,stroke:#e65100,color:#000,stroke-width:2px
    classDef kafka fill:#f3e5f5,stroke:#4a148c,color:#000
    classDef engine fill:#e8f5e9,stroke:#1b5e20,color:#000,stroke-width:2px
    classDef db fill:#fce4ec,stroke:#b71c1c,color:#000
    classDef s3 fill:#fff3e0,stroke:#e65100,color:#000
    classDef method fill:#e8f5e9,stroke:#1b5e20,color:#000,stroke-dasharray: 5 5
    classDef scale fill:#e1f5fe,stroke:#01579b,color:#000,stroke-dasharray: 5 5
```

### Deep Technical QA

#### 4. Interview Preparation: Multi-Level QA

##### 🟢 Basic Level (Jr. Engineer / 0-2 Yrs)

**Q1: What is the difference between High-Level Design (HLD) and Low-Level Design (LLD)?**

**A1:** HLD is the architectural blueprint — it defines system components, their interactions, data flow, and tech stack choices. Audience: architects and PMs. LLD is the implementation specification — it defines classes, methods, database schemas, API contracts, and error handling. Audience: developers. HLD answers "what components exist and how they talk?" LLD answers "how does each component work internally?"

**Q2: What is the difference between synchronous and asynchronous communication in distributed systems?**

**A2:** In synchronous communication, the caller blocks waiting for a response. Example: HTTP/REST — Service A calls Service B and waits. Tight coupling — both must be up. In asynchronous communication, the caller publishes a message to a queue/bus and continues immediately. Example: Kafka — Producer publishes, Consumer processes later. Loose coupling — queue buffers failures. Sync is simpler to debug (linear flow); async is harder (needs distributed tracing) but offers higher throughput and resilience.

**Q3:** What is API versioning and what strategies exist?

**A3:** API versioning ensures backward compatibility when APIs evolve. **URI versioning** (`/api/v1/users`, `/api/v2/users`) — simplest to understand and route, but clutters URLs. **Header versioning** (`Accept: application/vnd.myapp.v1+json`) — keeps URLs clean but harder to test. **Query parameter versioning** (`?version=1`) — easy to implement but allows clients to stick indefinitely to old versions. Best practice: use header versioning for internal APIs (cleaner), URI versioning for public APIs (more explicit). Always communicate deprecation via the `Sunset` HTTP header (e.g., `Sunset: Sat, 31 Dec 2026 23:59:59 GMT`) and the `Deprecation` header. Versioning matters for distributed teams because different services evolve at different speeds — a breaking change in one service should not force all consumers to upgrade simultaneously.

##### 🟡 Intermediate Level (Mid-Level / 2-5 Yrs)

**Q1: Design a RESTful API for a project management system. Show the OpenAPI 3.0 schema for projects with pagination.**

**A1:**
```yaml
openapi: 3.0.3
info:
  title: Project Management API
  version: 2.1.0
paths:
  /api/v2/projects:
    get:
      summary: List projects with pagination
      parameters:
        - name: cursor
          in: query
          schema:
            type: string
          description: Cursor-based pagination token
        - name: size
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
        - name: status
          in: query
          schema:
            type: string
            enum: [ACTIVE, ARCHIVED, COMPLETED]
      responses:
        '200':
          description: Paginated list of projects
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Project'
                  pagination:
                    type: object
                    properties:
                      nextCursor:
                        type: string
                      total:
                        type: integer
```
Key design decisions: cursor-based pagination (stable under writes), max page size of 100, enum validation for status.

**Q2: Implement the CQRS pattern for a task management system. Show the command side vs query side separation.**

**A2:**
```java
// COMMAND SIDE (Writes)
@RestController
@RequestMapping("/api/commands/tasks")
public class TaskCommandController {
    private final CommandBus commandBus;
    @PostMapping
    public CompletableFuture<UUID> createTask(@Valid @RequestBody CreateTaskCommand cmd) {
        return commandBus.dispatch(cmd);
    }
}

// QUERY SIDE (Reads)
@RestController
@RequestMapping("/api/queries/tasks")
public class TaskQueryController {
    private final TaskReadRepository readRepo;
    @GetMapping("/{id}")
    public TaskDetail getTaskDetail(@PathVariable UUID id) {
        return readRepo.findById(id)
            .orElseThrow(() -> new NotFoundException("Task not found"));
    }
}

// EVENT HANDLER — Syncs read model from command events
@Component
public class TaskEventHandler {
    @EventListener
    public void on(TaskCreatedEvent event) {
        readRepo.save(new TaskDetail(event.getTaskId(), event.getTitle(), ...));
    }
}
```
CQRS separates read and write models so each can be optimized independently: writes use domain-driven aggregates, reads use denormalized projections.

##### 🔴 Advanced Level (Senior / 5-8 Yrs)

**Q1: Design a SAGA pattern for distributed transaction spanning Order, Inventory, and Payment services. How do you handle compensation?**

**A1:** Use **Choreography-based SAGA** with Kafka:
```java
// Order Service — start saga
@Transactional
public void createOrder(CreateOrderCommand cmd) {
    Order order = orderRepository.save(new Order(cmd));
    kafka.send("order-events", new OrderCreatedEvent(order.getId(), order.getItems()));
}

// Inventory Service — participate in saga
@KafkaListener(topics = "order-events")
public void handle(OrderCreatedEvent event) {
    try {
        inventoryService.reserve(event.getOrderId(), event.getItems());
        kafka.send("order-events", new InventoryReservedEvent(event.getOrderId()));
    } catch (InsufficientInventoryException e) {
        kafka.send("order-events", new InventoryReservationFailedEvent(event.getOrderId()));
    }
}

// Payment Service — with compensatory action
@KafkaListener(topics = "order-events")
public void handle(InventoryReservedEvent event) {
    try {
        paymentService.charge(event.getOrderId());
        kafka.send("order-events", new PaymentCompletedEvent(event.getOrderId()));
    } catch (PaymentException e) {
        kafka.send("inventory-commands", new ReleaseInventoryCommand(event.getOrderId()));
        kafka.send("order-events", new PaymentFailedEvent(event.getOrderId()));
    }
}
```
If Payment fails, Inventory releases the reservation (compensating transaction). The saga state is tracked via event sequence; a `SagaInstance` entity persists current step, status, and retry count for observability.

**Q2: Compare gRPC streaming vs WebSocket for real-time collaboration. What are the performance and protocol-level trade-offs?**

**A2:**

| Aspect | gRPC Stream | WebSocket |
|--------|-------------|-----------|
| Protocol | HTTP/2 (binary) | HTTP/1.1 upgrade (binary or text) |
| Flow control | Built-in (HTTP/2 credit-based window) | Application-level (manual) |
| Multiplexing | Native (one TCP, many streams) | Separate connections per channel |
| Browser support | Requires gRPC-Web proxy | Native (browser API) |
| Backpressure | Automatic via `WINDOW_UPDATE` frames | Must implement manually |

For collaborative editing: use WebSocket for browser clients (native support) with a custom backpressure layer. Use gRPC for server-to-server sync (automatic flow control, multiplexing). Hybrid approach: WebSocket between client→proxy, gRPC between proxy→collaboration service.

##### ⚫ Expert Level (Staff/Principal / 8+ Yrs)

**Q1: Explain the memory model of a WebSocket server handling 100K concurrent connections. How do you manage per-connection buffers without exhausting heap?**

**A1:** Netty-based approach (Spring WebFlux, Play Framework):

Each connection: TCP receive buffer (default 64KB) + send buffer (64KB) = 128KB. After tuning (`tcp_rmem = 4096 16384 65536`), reduce to ~16KB per direction = 1.6GB for 100K connections. Netty Channel overhead: ~4KB = 400MB. Application state: ~2KB = 200MB. **Total ~2.2GB — feasible on a 4GB heap.**

Key strategies:
- **Direct ByteBuffers** (off-heap) — reduces GC pressure
- **Reference counting** — Netty's `ByteBuf` uses `refCnt()` to track lifecycle
- **Epoll ET (Edge-Triggered)** — reduces event loop wakeups
- **Watermark flow control:** `setWriteBufferWaterMark(low=32KB, high=64KB)` — when buffer exceeds high watermark, channel becomes unwritable, creating backpressure
- **Idle timeout:** Connections idle >60s are closed via `IdleStateHandler`

At 100K connections, the EventLoop model binds each connection to a single thread for its lifetime → no locking per connection. TCP buffer tuning is the critical knob — default 128KB/conn → 12.8GB (impossible); tuned to 16KB/conn → 1.6GB (feasible).

**Q2: How do you prevent event storms and cyclic dependencies in a SAGA-based distributed system with 50 microservices?**

**A2:** Full mesh SAGA is an anti-pattern beyond ~10 services. Use **Centralized Orchestrator SAGA + Topic-per-Domain**:

1. **Deduplication by idempotency key:** Every event has an `idempotencyKey` (UUID). Consumers store processed keys in Redis with TTL = 7 days. Duplicates are silently dropped.

2. **Saga state machine with timeouts:**
   ```java
   @Entity
   @Table(name = "saga_state")
   public class SagaInstance {
       @Id private UUID sagaId;
       private String currentStep;
       private String status;  // RUNNING | COMPLETED | FAILED | COMPENSATING
       private String payload;
       private int retryCount;
       private Instant nextRetryAt;
   }
   ```

3. **Exponential backoff:** Retry 3 times (1s, 4s, 16s). After max retries → COMPENSATING state.

4. **Dead letter queue:** Events unconsumed after 24h → DLQ. Scheduled job alerts on-call engineers.

5. **Circuit breaker per saga type:** If >5% failures in 5 minutes, reject new sagas of that type immediately.

6. **Strict topic isolation:** One Kafka topic per bounded context. Services never emit to topics outside their domain. Event schema registry enforces evolution.

---

*Reference: TrainWithShubham — Computer Networking & System Design For DevOps in OneShot (YouTube)*
