# Topic 11: Production Case Study — Project Tracking Application (Jira/Trello Clone)

## 1. Theoretical Foundation & System Mechanics

### The Core Concept

This case study walks through the complete system design of a **Project Tracking Application** — a SaaS platform similar to Jira, Trello, or Asana. The system supports organizations managing projects, tasks, sprints, and teams with real-time collaboration, file attachments, and an AI assistant.

**Requirements Gathering:**

**Functional Requirements:**
- User authentication and authorization (RBAC: Admin, Project Manager, Developer, Viewer)
- Project CRUD with configurable workflows (Kanban, Scrum, custom)
- Task management with status transitions, priority, assignees, due dates
- Real-time collaboration (comments, mentions, notifications)
- File attachments (images, documents, screenshots)
- Search across projects and tasks (full-text + filters)
- AI Assistant for task suggestions, sprint planning, and status summaries
- Audit logging for compliance

**Non-Functional Requirements:**
- Availability: 99.95% (downtime < 4.38 hours/year)
- Latency: p95 < 200ms for API calls, p99 < 500ms
- Throughput: 50K DAU, 500K tasks created/day, 5M API calls/day
- Consistency: Strong consistency for task state (no lost updates), eventual for search and analytics
- Durability: Zero data loss (RPO = 0), RTO < 15 minutes
- Security: SOC 2 compliance, data encryption at rest and in transit
- Multi-tenancy: Data isolation between organizations

### The "Why"

The bottleneck in project tracking applications is **contention on shared resources**. When 50 developers are simultaneously updating tasks in the same project, we need:

1. **Optimistic concurrency control** — prevent lost updates without pessimistic locking
2. **Real-time sync** — all collaborators see changes within seconds
3. **Search that stays fresh** — newly created tasks appear in search results immediately
4. **Scalable notifications** — 500K task updates/day generate 2M+ notification events

### Trade-offs

| Decision | Trade-off | Rationale |
|----------|-----------|-----------|
| Microservices | Operational complexity over monolith | Independent scaling of search, notifications, AI |
| PostgreSQL per service | Joins across services harder | Avoid shared-database anti-pattern |
| WebSocket for real-time | Connection overhead vs polling | <100ms notification latency required |
| Eventual consistency for search | Search may lag 500ms | Acceptable; full consistency would kill throughput |
| S3 for files | Latency vs cost | Files are read infrequently; CDN mitigates latency |

---

## 2. Production Implementation (Full Stack & Cloud)

### Backend & Code Architecture

**Microservices Architecture:**

```mermaid
flowchart TD
    GW["API Gateway (Kong)<br/>Rate limiting: 1000 req/s<br/>Auth: JWT validation<br/>Routing: /api/v1/*"]:::gw
    AUTH["Auth Service<br/>JWT/OAuth2<br/>/auth/*"]:::svc
    PS["Project Service<br/>REST API<br/>/projects/*"]:::svc
    TS["Task Service<br/>REST + WebSocket<br/>/tasks/*"]:::svc
    PDB[("Project DB<br/>PostgreSQL")]:::db
    TDB[("Task DB<br/>PostgreSQL")]:::db
    NS["Notification Service<br/>WebSocket/SSE<br/>Email/SMS/Push"]:::svc
    SS["Search Service<br/>Elasticsearch<br/>/search/*"]:::svc
    AI["AI Assistant<br/>LLM Service<br/>/ai/*"]:::svc

    GW --> AUTH & PS & TS
    PS --> PDB
    TS --> TDB
    AUTH --> NS & SS
    NS --> AI

    classDef gw fill:#f3e5f5,stroke:#4a148c,color:#000,stroke-width:2px
    classDef svc fill:#e1f5fe,stroke:#01579b,color:#000
    classDef db fill:#fce4ec,stroke:#b71c1c,color:#000


**API Gateway Configuration (Spring Cloud Gateway):**

```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("auth-service", r -> r
                .path("/api/v1/auth/**")
                .filters(f -> f
                    .circuitBreaker(config -> config
                        .setName("authCircuitBreaker")
                        .setFallbackUri("forward:/fallback/auth"))
                    .retry(config -> config
                        .setRetries(3)
                        .setStatuses(HttpStatus.SERVICE_UNAVAILABLE))
                    .requestRateLimiter(config -> config
                        .setRateLimiter(redisRateLimiter())))
                .uri("lb://auth-service"))
            .route("project-service", r -> r
                .path("/api/v1/projects/**")
                .filters(f -> f
                    .circuitBreaker(config -> config
                        .setName("projectCircuitBreaker"))
                    .addRequestHeader("X-Gateway-Request", "true"))
                .uri("lb://project-service"))
            .route("task-service", r -> r
                .path("/api/v1/tasks/**")
                .filters(f -> f
                    .circuitBreaker(config -> config
                        .setName("taskCircuitBreaker"))
                    .addResponseHeader("X-API-Version", "v1"))
                .uri("lb://task-service"))
            .route("notification-service", r -> r
                .path("/api/v1/notifications/**")
                .uri("lb://notification-service"))
            .route("ai-service", r -> r
                .path("/api/v1/ai/**")
                .filters(f -> f
                    .circuitBreaker(config -> config
                        .setName("aiCircuitBreaker")
                        .setFallbackUri("forward:/fallback/ai")))
                .uri("lb://ai-service"))
            .build();
    }
}
```

**Auth Service with JWT + OAuth2:**

```java
@Service
public class AuthenticationService {

    private final UserRepository userRepository;
    private final JwtTokenProvider tokenProvider;

    public AuthResponse authenticate(LoginRequest request) {
        User user = userRepository.findByEmail(request.getEmail())
            .orElseThrow(() -> new AuthenticationException("Invalid credentials"));

        if (!passwordEncoder.matches(request.getPassword(), user.getPasswordHash())) {
            throw new AuthenticationException("Invalid credentials");
        }

        // Generate access token (15 min) and refresh token (7 days)
        String accessToken = tokenProvider.generateAccessToken(
            user.getId(), user.getOrganizationId(), user.getRoles());
        String refreshToken = tokenProvider.generateRefreshToken(user.getId());

        // Store refresh token hash in Redis for revocation
        redisTemplate.opsForValue().set(
            "refresh:" + user.getId(),
            hashRefreshToken(refreshToken),
            Duration.ofDays(7));

        return new AuthResponse(accessToken, refreshToken, tokenProvider.getExpiry());
    }

    public AuthResponse refreshAccessToken(String refreshToken) {
        Claims claims = tokenProvider.validateRefreshToken(refreshToken);
        String userId = claims.getSubject();

        // Verify refresh token in Redis
        String storedHash = redisTemplate.opsForValue().get("refresh:" + userId);
        if (storedHash == null || !storedHash.equals(hashRefreshToken(refreshToken))) {
            throw new AuthenticationException("Refresh token revoked or expired");
        }

        User user = userRepository.findById(UUID.fromString(userId))
            .orElseThrow(() -> new AuthenticationException("User not found"));

        String newAccessToken = tokenProvider.generateAccessToken(
            user.getId(), user.getOrganizationId(), user.getRoles());

        return new AuthResponse(newAccessToken, refreshToken, tokenProvider.getExpiry());
    }

    @PreAuthorize("hasRole('ADMIN')")
    public void revokeUserSessions(UUID userId) {
        redisTemplate.delete("refresh:" + userId);
        // Add to JWT blacklist
        redisTemplate.opsForSet().add("jwt-blacklist:" + userId, "*");
        redisTemplate.expire("jwt-blacklist:" + userId, Duration.ofHours(24));
    }
}
```

**Task Service with Optimistic Locking:**

```java
@Entity
@Table(name = "tasks")
public class Task {
    @Id
    private UUID id;

    @Column(nullable = false)
    private String title;

    @Enumerated(EnumType.STRING)
    private TaskStatus status;

    @Column(name = "assignee_id")
    private UUID assigneeId;

    @Column(name = "project_id", nullable = false)
    private UUID projectId;

    @Column(name = "sprint_id")
    private UUID sprintId;

    @Version
    private Long version;  // Optimistic lock

    @CreationTimestamp
    private Instant createdAt;

    @UpdateTimestamp
    private Instant updatedAt;
}

@Service
@Transactional
public class TaskService {

    private final TaskRepository taskRepository;
    private final KafkaTemplate<String, TaskEvent> kafka;
    private final RedisTemplate<String, String> redis;

    public Task updateTaskStatus(UUID taskId, TaskStatus newStatus, UUID userId, Long expectedVersion) {
        Task task = taskRepository.findById(taskId)
            .orElseThrow(() -> new NotFoundException("Task not found"));

        // Validate state transition
        if (!isValidTransition(task.getStatus(), newStatus)) {
            throw new InvalidTransitionException(
                "Cannot transition from " + task.getStatus() + " to " + newStatus);
        }

        try {
            task.setStatus(newStatus);
            task.setUpdatedAt(Instant.now());
            taskRepository.save(task);  // Hibernate checks @Version on flush
        } catch (OptimisticLockException e) {
            // Retry: re-fetch and apply
            throw new ConcurrentModificationException(
                "Task was modified by another user. Please refresh and try again.",
                e);
        }

        // Publish event for real-time sync
        TaskStatusChangedEvent event = new TaskStatusChangedEvent(
            taskId, task.getProjectId(), task.getStatus(), newStatus, userId, Instant.now());
        kafka.send("task-events", taskId.toString(), event);

        // Invalidate cache
        redis.delete("task:" + taskId);
        redis.delete("project-tasks:" + task.getProjectId());

        return task;
    }

    private boolean isValidTransition(TaskStatus from, TaskStatus to) {
        // TODO: Load workflow config from Project Service
        return switch (from) {
            case TODO -> to == TaskStatus.IN_PROGRESS;
            case IN_PROGRESS -> to == TaskStatus.IN_REVIEW || to == TaskStatus.BLOCKED;
            case IN_REVIEW -> to == TaskStatus.DONE || to == TaskStatus.IN_PROGRESS;
            case BLOCKED -> to == TaskStatus.TODO || to == TaskStatus.IN_PROGRESS;
            case DONE -> false;  // Terminal state
        };
    }
}
```

**Real-Time Notification Service with WebSocket:**

```java
@Component
public class WebSocketNotificationHandler {

    private final SimpMessagingTemplate messagingTemplate;
    private final Map<UUID, List<SessionInfo>> projectSubscriptions = new ConcurrentHashMap<>();

    public void subscribeToProject(UUID userId, UUID projectId, String sessionId) {
        projectSubscriptions
            .computeIfAbsent(projectId, k -> new CopyOnWriteArrayList<>())
            .add(new SessionInfo(userId, sessionId));
    }

    public void unsubscribe(String sessionId) {
        projectSubscriptions.values().forEach(list ->
            list.removeIf(s -> s.sessionId().equals(sessionId)));
    }

    @KafkaListener(topics = "task-events")
    public void onTaskEvent(TaskStatusChangedEvent event) {
        List<SessionInfo> subscribers = projectSubscriptions
            .getOrDefault(event.getProjectId(), Collections.emptyList());

        // Send to all subscribers in the project
        subscribers.forEach(session -> {
            messagingTemplate.convertAndSendToUser(
                session.sessionId(),
                "/queue/task-updates",
                event
            );
        });

        // Also send email/push for important updates
        if (event.getNewStatus() == TaskStatus.BLOCKED ||
            event.getNewStatus() == TaskStatus.IN_REVIEW) {
            notificationService.sendEmailNotification(event);
        }
    }
}
```

**AI Assistant Service Integration:**

```java
@Service
public class AiAssistantService {

    private final RestClient llmClient;  // OpenAI / Anthropic / LLAMA

    @Retryable(
        retryFor = {TimeoutException.class, ServerErrorException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public SprintRecommendation suggestSprintPlan(UUID projectId, UUID sprintId) {
        // Gather context
        List<Task> backlogTasks = taskRepository.findByProjectAndSprintIsNull(projectId);
        List<TeamMember> teamMembers = teamService.getTeamMembers(projectId);
        Sprint currentSprint = sprintRepository.findById(sprintId).orElseThrow();

        // Build LLM prompt
        String prompt = buildSprintPrompt(backlogTasks, teamMembers, currentSprint);

        // Call LLM with structured output
        var request = new LlmRequest(prompt, 0.3, 2000);
        var response = llmClient.post()
            .uri("/v1/completions")
            .body(request)
            .retrieve()
            .body(LlmResponse.class);

        return parseSprintRecommendation(response.getText());
    }

    @CircuitBreaker(name = "llmService")
    public TaskSummary summarizeTask(UUID taskId) {
        Task task = taskService.getTask(taskId);
        List<Comment> comments = commentService.getComments(taskId);

        String prompt = String.format("""
            Summarize the following task and its comments:
            Title: %s
            Description: %s
            Status: %s
            Comments: %s

            Provide a 2-3 sentence summary.
            """, task.getTitle(), task.getDescription(), task.getStatus(), comments);

        var request = new LlmRequest(prompt, 0.5, 500);
        var response = llmClient.post()
            .uri("/v1/completions")
            .body(request)
            .retrieve()
            .body(LlmResponse.class);

        return new TaskSummary(taskId, task.getTitle(), response.getText());
    }
}
```

### DevOps & Infrastructure

**Dockerfile for Java Microservice:**

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY gradlew build.gradle.kts settings.gradle.kts ./
COPY gradle gradle
RUN ./gradlew dependencies --no-daemon
COPY src src
RUN ./gradlew bootJar --no-daemon -x test

FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S app && adduser -S app -G app
USER app
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 8080
EXPOSE 9090  # Metrics port

# JVM tuning for containerized environment
ENV JAVA_OPTS="-XX:+UseZGC \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+ZGenerational \
               -Xlog:gc*:file=/dev/stdout:time,tags:filecount=0 \
               -Djava.security.egd=file:/dev/./urandom"

HEALTHCHECK --interval=15s --timeout=5s --retries=3 \
    CMD wget -qO- http://localhost:9090/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

**Kubernetes Deployment for Task Service:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: task-service
  namespace: project-tracker
  labels:
    app: task-service
    version: "2.1.0"
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  selector:
    matchLabels:
      app: task-service
  template:
    metadata:
      labels:
        app: task-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - task-service
                topologyKey: topology.kubernetes.io/zone
      terminationGracePeriodSeconds: 60
      containers:
        - name: task-service
          image: myregistry/task-service:2.1.0
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 9090
              name: metrics
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: DB_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
            - name: KAFKA_BOOTSTRAP_SERVERS
              value: "kafka-cluster.kafka:9092"
            - name: REDIS_HOST
              value: "redis-cluster.redis"
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 9090
            initialDelaySeconds: 30
            periodSeconds: 15
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 9090
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
          startupProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 9090
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 30
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: task-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: task-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: kafka_consumer_lag
        target:
          type: AverageValue
          averageValue: 100
```

**CI/CD Pipeline (GitHub Actions):**

```yaml
name: Deploy Task Service

on:
  push:
    branches: [main]
    paths:
      - 'services/task-service/**'
      - 'libs/common/**'
  pull_request:
    branches: [main]

env:
  REGISTRY: myregistry.azurecr.io
  IMAGE_NAME: task-service
  K8S_NAMESPACE: project-tracker

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - name: Run tests
        run: |
          cd services/task-service
          ./gradlew test --no-daemon
      - name: Integration tests
        run: |
          docker compose -f docker-compose.test.yml up -d
          ./gradlew integrationTest --no-daemon

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: |
          docker build -t $REGISTRY/$IMAGE_NAME:${{ github.sha }} \
            -t $REGISTRY/$IMAGE_NAME:latest \
            -f services/task-service/Dockerfile .
      - name: Push to registry
        run: |
          docker push $REGISTRY/$IMAGE_NAME:${{ github.sha }}
          docker push $REGISTRY/$IMAGE_NAME:latest

  deploy-staging:
    needs: build-and-push
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to staging
        run: |
          kubectl set image deployment/task-service \
            task-service=$REGISTRY/$IMAGE_NAME:${{ github.sha }} \
            -n $K8S_NAMESPACE
          kubectl rollout status deployment/task-service \
            -n $K8S_NAMESPACE --timeout=5m
      - name: Smoke tests
        run: |
          curl -f https://staging-api.example.com/actuator/health
          curl -f https://staging-api.example.com/api/v1/tasks?limit=1

  deploy-production:
    needs: deploy-staging
    environment: production
    runs-on: ubuntu-latest
    steps:
      - name: Canary deployment (10%)
        run: |
          kubectl set image deployment/task-service-canary \
            task-service=$REGISTRY/$IMAGE_NAME:${{ github.sha }} \
            -n $K8S_NAMESPACE
      - name: Wait for canary validation (15 min)
        run: sleep 900
      - name: Rollout to all instances
        run: |
          kubectl set image deployment/task-service \
            task-service=$REGISTRY/$IMAGE_NAME:${{ github.sha }} \
            -n $K8S_NAMESPACE
          kubectl rollout status deployment/task-service \
            -n $K8S_NAMESPACE --timeout=10m
      - name: Notify team
        run: |
          curl -X POST $SLACK_WEBHOOK \
            -H 'Content-Type: application/json' \
            -d '{"text": "Task Service v2.1.0 deployed to production 🚀"}'
```

### Database Design

**PostgreSQL Schemas:**

```sql
-- ========================================
-- ORGANIZATIONS (Multi-tenant root)
-- ========================================
CREATE TABLE organizations (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    slug        VARCHAR(100) UNIQUE NOT NULL,
    plan        VARCHAR(50) NOT NULL DEFAULT 'free',
    settings    JSONB DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ========================================
-- USERS (Across organizations)
-- ========================================
CREATE TABLE users (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email             VARCHAR(255) UNIQUE NOT NULL,
    password_hash     VARCHAR(255) NOT NULL,
    display_name      VARCHAR(255) NOT NULL,
    avatar_url        VARCHAR(500),
    default_org_id    UUID REFERENCES organizations(id),
    preferences       JSONB DEFAULT '{"theme": "light", "timezone": "UTC"}',
    is_active         BOOLEAN DEFAULT true,
    last_login_at     TIMESTAMPTZ,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ========================================
-- ORGANIZATION MEMBERSHIP + RBAC
-- ========================================
CREATE TABLE org_members (
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    -- Roles: admin, project_manager, developer, viewer
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (org_id, user_id)
);

-- ========================================
-- PROJECTS
-- ========================================
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    key             VARCHAR(10) NOT NULL,  -- e.g., "PROJ" for ticket IDs PROJ-101
    workflow_type   VARCHAR(50) NOT NULL DEFAULT 'kanban',
    -- kanban, scrum, custom
    workflow_config JSONB DEFAULT '{}',
    -- {"statuses": ["TODO","IN_PROGRESS","DONE"], "transitions": [...]}
    lead_id         UUID REFERENCES users(id),
    is_archived     BOOLEAN DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, key)
);

CREATE INDEX idx_projects_org ON projects(org_id) WHERE NOT is_archived;

-- ========================================
-- SPRINTS (Scrum)
-- ========================================
CREATE TABLE sprints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    goal            TEXT,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'planning',
    -- planning, active, completed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT valid_dates CHECK (end_date > start_date)
);

-- ========================================
-- TASKS (Core entity)
-- ========================================
CREATE TYPE task_status AS ENUM (
    'TODO', 'IN_PROGRESS', 'IN_REVIEW', 'DONE', 'BLOCKED'
);

CREATE TYPE task_priority AS ENUM (
    'LOWEST', 'LOW', 'MEDIUM', 'HIGH', 'CRITICAL'
);

CREATE TABLE tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    sprint_id       UUID REFERENCES sprints(id) ON DELETE SET NULL,
    parent_id       UUID REFERENCES tasks(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    status          task_status NOT NULL DEFAULT 'TODO',
    priority        task_priority NOT NULL DEFAULT 'MEDIUM',
    assignee_id     UUID REFERENCES users(id) ON DELETE SET NULL,
    reporter_id     UUID NOT NULL REFERENCES users(id),
    story_points    SMALLINT CHECK (story_points > 0),
    due_date        TIMESTAMPTZ,
    sort_order      FLOAT NOT NULL DEFAULT 0,
    -- For drag-and-drop ordering within a column
    labels          TEXT[] DEFAULT '{}',
    custom_fields   JSONB DEFAULT '{}',
    version         BIGINT NOT NULL DEFAULT 1,  -- Optimistic lock
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Core query indexes
CREATE INDEX idx_tasks_project ON tasks(project_id, status, sort_order);
CREATE INDEX idx_tasks_sprint ON tasks(sprint_id) WHERE sprint_id IS NOT NULL;
CREATE INDEX idx_tasks_assignee ON tasks(assignee_id, status) WHERE assignee_id IS NOT NULL;
CREATE INDEX idx_tasks_due_date ON tasks(project_id, due_date) WHERE due_date IS NOT NULL;
CREATE INDEX idx_tasks_labels ON tasks USING GIN(labels);
CREATE INDEX idx_tasks_fulltext ON tasks USING GIN(
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(description, ''))
);

-- ========================================
-- COMMENTS
-- ========================================
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES users(id),
    body            TEXT NOT NULL,
    mentions        UUID[] DEFAULT '{}',
    -- Array of user IDs mentioned with @
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ
);

CREATE INDEX idx_comments_task ON comments(task_id, created_at);

-- ========================================
-- ATTACHMENTS (S3 references)
-- ========================================
CREATE TABLE attachments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    uploader_id     UUID NOT NULL REFERENCES users(id),
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    s3_key          VARCHAR(1000) NOT NULL,
    -- e.g., "org/abc123/task/def456/img.jpg"
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_attachments_task ON attachments(task_id);

-- ========================================
-- AUDIT LOG
-- ========================================
CREATE TABLE audit_log (
    id              BIGSERIAL,
    org_id          UUID NOT NULL,
    actor_id        UUID NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(50) NOT NULL,
    -- created, updated, deleted, status_changed, assigned, etc.
    changes         JSONB NOT NULL,
    -- {"field": "status", "old": "TODO", "new": "IN_PROGRESS"}
    metadata        JSONB DEFAULT '{}',
    -- {"source_ip": "...", "user_agent": "..."}
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (occurred_at);

-- Monthly partitions
CREATE TABLE audit_log_2026_01 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE audit_log_2026_02 PARTITION OF audit_log
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_org ON audit_log(org_id, occurred_at);
```

### Cloud Architecture

```mermaid
flowchart TD
    CF["Cloudflare DNS + DDoS Protection"]:::net
    CDN["CloudFront CDN (Static Assets)"]:::cdn
    WAF["AWS WAF (Rate Limiting, SQLi, XSS)"]:::security
    KONG["Kong API Gateway (ECS Fargate)"]:::gw
    AUTH0["Auth0 / OAuth2"]:::auth
    ALB["ALB (Internal)"]:::lb
    AUTH["Auth Service"]:::svc
    PROJ["Project Service"]:::svc
    TASK["Task Service"]:::svc
    SEARCH["Search Service"]:::svc
    RDS_A[("RDS (Aurora)")]:::db
    RDS_P[("RDS (Aurora)")]:::db
    RDS_T[("RDS (Aurora)")]:::db
    ES[("Elasticsearch (7 nodes)")]:::db
    KAFKA["Kafka (MSK)"]:::kafka
    REDIS["Redis Cluster (ElastiCache)<br/>Session / Task Query / Rate Limit"]:::cache
    NOTIFY["Notification Service"]:::svc
    AI["AI Service"]:::svc
    SNS_SES["SNS / SES<br/>Email/SMS/Push"]:::out
    LLM["OpenAI / Anthropic"]:::out
    S3_FILES["S3 (Files)"]:::s3
    S3_LOGS["S3 (Logs)"]:::s3
    ATHENA["Athena"]:::s3

    CF --> CDN --> WAF --> KONG
    KONG --> AUTH0
    KONG --> ALB
    ALB --> AUTH & PROJ & TASK & SEARCH
    AUTH --> RDS_A
    PROJ --> RDS_P
    TASK --> RDS_T
    SEARCH --> ES
    AUTH & PROJ & TASK --> KAFKA
    KAFKA --> NOTIFY & AI
    NOTIFY --> SNS_SES
    AI --> LLM
    KAFKA --> REDIS
    S3_FILES --> CDN
    S3_LOGS --> ATHENA

    subgraph OBS["Observability"]
        CW["CloudWatch"]:::mon --> PROM["Prometheus"]:::mon --> GRAF["Grafana"]:::mon
        FB["Filebeat"]:::mon --> LS["Logstash"]:::mon --> ES_O["ES"]:::mon --> KIB["Kibana"]:::mon
        XR["X-Ray (Distributed Tracing)"]:::mon
    end

    subgraph CICD["CI/CD"]
        GH["GitHub"]:::ci --> ACT["Actions"]:::ci --> ECR["ECR"]:::ci --> ARGO["ArgoCD"]:::ci --> EKS["EKS"]:::ci
    end

    classDef net fill:#e1f5fe,stroke:#01579b,color:#000
    classDef cdn fill:#fff3e0,stroke:#e65100,color:#000
    classDef security fill:#fce4ec,stroke:#b71c1c,color:#000
    classDef gw fill:#f3e5f5,stroke:#4a148c,color:#000,stroke-width:2px
    classDef auth fill:#e8f5e9,stroke:#1b5e20,color:#000
    classDef lb fill:#e1f5fe,stroke:#01579b,color:#000
    classDef svc fill:#e8f5e9,stroke:#1b5e20,color:#000
    classDef db fill:#fce4ec,stroke:#b71c1c,color:#000
    classDef kafka fill:#f3e5f5,stroke:#4a148c,color:#000,stroke-width:2px
    classDef cache fill:#fff3e0,stroke:#e65100,color:#000
    classDef out fill:#e1f5fe,stroke:#01579b,color:#000
    classDef s3 fill:#fff3e0,stroke:#e65100,color:#000
    classDef mon fill:#e1f5fe,stroke:#01579b,color:#000
    classDef ci fill:#f3e5f5,stroke:#4a148c,color:#000


### Frontend Architecture

```javascript
// Simplified React component with real-time updates
// frontend/src/components/TaskBoard.jsx

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { useWebSocket } from '../hooks/useWebSocket';

export function TaskBoard({ projectId }) {
  const queryClient = useQueryClient();
  const { lastMessage } = useWebSocket(`/ws/projects/${projectId}`);

  // React Query for server state management
  const { data: tasks, isLoading } = useQuery({
    queryKey: ['tasks', projectId],
    queryFn: () => fetch(`/api/v1/tasks?projectId=${projectId}`).then(r => r.json()),
    staleTime: 30000,  // 30 seconds before refetch
    cacheTime: 300000, // 5 minutes in cache
  });

  // Real-time WebSocket updates
  useEffect(() => {
    if (lastMessage) {
      const event = JSON.parse(lastMessage.data);
      // Optimistically update cache
      queryClient.setQueryData(['tasks', projectId], (old) => {
        return old.map(t =>
          t.id === event.taskId
            ? { ...t, status: event.newStatus, version: event.version }
            : t
        );
      });
    }
  }, [lastMessage, projectId, queryClient]);

  // Drag and drop mutation with optimistic update
  const moveTask = useMutation({
    mutationFn: ({ taskId, newStatus, version }) =>
      fetch(`/api/v1/tasks/${taskId}/status`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ status: newStatus, expectedVersion: version }),
      }).then(r => {
        if (r.status === 409) throw new Error('Concurrent modification');
        return r.json();
      }),
    onMutate: async ({ taskId, newStatus }) => {
      await queryClient.cancelQueries(['tasks', projectId]);
      const previous = queryClient.getQueryData(['tasks', projectId]);
      queryClient.setQueryData(['tasks', projectId], old =>
        old.map(t => t.id === taskId ? { ...t, status: newStatus } : t)
      );
      return { previous };
    },
    onError: (err, vars, context) => {
      queryClient.setQueryData(['tasks', projectId], context.previous);
      toast.error('Failed to move task. Another user modified it.');
    },
    onSettled: () => {
      queryClient.invalidateQueries(['tasks', projectId]);
    },
  });

  if (isLoading) return <SkeletonBoard />;

  return (
    <DndContext onDragEnd={(result) => {
      const task = tasks.find(t => t.id === result.active.id);
      if (task && result.over) {
        moveTask.mutate({
          taskId: task.id,
          newStatus: result.over.id,
          version: task.version,
        });
      }
    }}>
      <div className="grid grid-cols-5 gap-4">
        {['TODO', 'IN_PROGRESS', 'IN_REVIEW', 'BLOCKED', 'DONE'].map(status => (
          <Droppable key={status} id={status}>
            <ColumnHeader status={status} count={
              tasks.filter(t => t.status === status).length
            } />
            {tasks.filter(t => t.status === status).map(task => (
              <Draggable key={task.id} id={task.id}>
                <TaskCard task={task} />
              </Draggable>
            ))}
          </Droppable>
        ))}
      </div>
    </DndContext>
  );
}
```

---

## 3. Real-World Scaling Scenarios

### The Bottleneck

**Scenario:** A large enterprise customer (10,000 users) imports 50,000 tasks via CSV upload. This triggers:

1. 50,000 database inserts
2. 50,000 search index updates
3. 200,000 notification events (mentions and assignments)
4. 50,000 WebSocket broadcasts to subscribed clients
5. 50,000 AI summary requests

**Failure Sequence:**

```mermaid
flowchart TD
    T0["T=0s: CSV upload begins<br/>50K tasks submitted"]:::fail
    T2["T=2s: Task Service inserts 500 tasks/s<br/>Each insert triggers Kafka event"]:::fail
    T5["T=5s: Kafka producer queue backs up<br/>Max batch size hit"]:::fail
    T10["T=10s: Notification WS buffer grows to 500MB"]:::fail
    T15["T=15s: Elasticsearch indexing saturates<br/>Queue: 100K documents"]:::fail
    T20["T=20s: AI Service — 50K LLM requests<br/>Rate limited by OpenAI"]:::fail
    T30["T=30s: Redis spikes 2GB → 8GB<br/>Eviction drops critical session data"]:::fail
    T45["T=45s: PostgreSQL pool exhausted"]:::fail
    T60["T=60s: API Gateway circuit breakers open<br/>503 for all endpoints"]:::fail
    T120["T=120s: Full system outage"]:::final

    T0 --> T2 --> T5 --> T10 --> T15 --> T20 --> T30 --> T45 --> T60 --> T120

    classDef fail fill:#fff3e0,stroke:#e65100,color:#000
    classDef final fill:#fce4ec,stroke:#b71c1c,color:#000,stroke-width:3px


**Root Causes:**
1. **No request throttling at the API level** — CSV import should be async with progress tracking
2. **Synchronous event processing** — each insert waited for Kafka ack before returning
3. **No backpressure** — all downstream services were overwhelmed simultaneously
4. **Missing bulk APIs** — 50K individual requests instead of batch inserts

### The Solution

**Step 1: Async CSV Import with Job Queue**

```mermaid
sequenceDiagram
    participant Client
    participant API as API Server
    participant S3 as S3 Storage
    participant DB as Database
    participant Q as SQS Queue
    participant W as Import Worker
    participant K as Kafka
    participant ES as Elasticsearch

    Client->>API: POST /api/v1/import/csv (multipart)
    API->>S3: Upload CSV file
    API->>DB: Create ImportJob (PENDING)
    API-->>Client: { jobId, statusUrl }

    Client->>API: GET /api/v1/import/123
    API-->>Client: { status: "PROCESSING", progress: 45% }

    Note over Q,W: Background Processing
    Q->>W: Dequeue batch
    W->>DB: Batch INSERT 1000 rows
    W->>K: 1 event per batch
    W->>ES: bulk_index(1000 documents)
    W->>Q: bulk_enqueue(notifications)


**Step 2: Implement Backpressure at Every Layer**

```java
// Task Service — Reactive backpressure with Kafka
@Service
public class ImportService {

    private static final int MAX_CONCURRENT_BATCHES = 5;

    private final Semaphore backpressure = new Semaphore(MAX_CONCURRENT_BATCHES);

    @Transactional
    public void processImportBatch(List<Task> tasks, UUID importJobId) {
        if (!backpressure.tryAcquire(10, TimeUnit.SECONDS)) {
            // Backpressure threshold hit — slow down
            updateImportProgress(importJobId, ImportStatus.THROTTLED);
            throw new BackpressureException("System at capacity, retrying batch");
        }

        try {
            // Batch insert
            taskRepository.saveAll(tasks);

            // Single Kafka event for the batch
            kafka.send("task-import-events", importJobId.toString(),
                new TaskBatchEvent(tasks, Instant.now()));
        } finally {
            backpressure.release();
        }
    }
}

// Notification Service — Adaptive rate limiting
@Component
public class AdaptiveNotificationLimiter {

    private final double targetDurationMs = 100.0;  // Target: 100ms per notification
    private final double smoothingFactor = 0.9;
    private double currentRate = 100.0;  // Start at 100 notifications/second

    @Scheduled(fixedRate = 1000)
    public void adaptRate() {
        double actualDuration = metrics.getAverageNotificationDuration();
        if (actualDuration > targetDurationMs * 1.2) {
            // We're too slow — reduce rate by 20%
            currentRate *= 0.8;
        } else if (actualDuration < targetDurationMs * 0.8) {
            // We have headroom — increase rate by 10%
            currentRate *= 1.1;
        }
        metrics.setCurrentRateLimit(currentRate);
    }

    public boolean allowNotification() {
        return rateLimiter.tryAcquire(1, currentRate);
    }
}
```

**Step 3: Add Capacity Planning and Auto-Scaling**

```mermaid
flowchart TD
    subgraph TASK_SCALE["Task Service Auto-Scaling"]
        T1["CPU > 70% for 5 min → +2 replicas (max 20)"]:::trig
        T2["Kafka consumer lag > 1000 → +4 replicas"]:::trig
        T3["Query latency > 200ms p99 → +2 replicas"]:::trig
    end

    subgraph SEARCH_SCALE["Search Service Auto-Scaling"]
        S1["ES indexing queue > 5000 → +3 data nodes"]:::trig
        S2["Search latency > 500ms p95 → +1 data node"]:::trig
    end

    subgraph AI_SCALE["AI Service"]
        A1["LLM queue depth > 100 → HTTP 429"]:::trig
        A2["OpenAI rate limit → cached fallback"]:::trig
    end

    subgraph DB_SCALE["Database Scaling"]
        D1["Aurora read replicas: 1 → 5 based on latency"]:::dscale
        D2["PgBouncer: 200 connections/replica"]:::dscale
        D3["Redis cluster: 3 shards, 2 replicas/shard"]:::dscale
        D4["Auto-shard when memory > 75%"]:::dscale
        D5["Kafka: 32 partitions/topic, 7d retention"]:::dscale
    end

    classDef trig fill:#e1f5fe,stroke:#01579b,color:#000
    classDef dscale fill:#e8f5e9,stroke:#1b5e20,color:#000


---

## 4. Senior-Level Interview Deep Dive

### System Design Challenge

**Question:** Design the "Real-Time Collaborative View" feature for the Project Tracking App. Multiple users (up to 100) should be able to view and modify the same task simultaneously. When User A changes the status, User B should see the update within 500ms. Handle conflict resolution when two users modify the same field simultaneously.

**Optimal Blueprint:**

```mermaid
flowchart TD
    UA["User A"]:::user
    UB["User B"]:::user
    UC["User C"]:::user
    UD["User D"]:::user
    TCS["Task Collaboration Service"]:::svc
    IMDS["In-Memory Document Store<br/>- Last-Writer-Wins per field<br/>- CRDT for multi-value fields<br/>- Version vector tracking"]:::store
    KAFKA["Kafka: task-events"]:::kafka
    PG[("PostgreSQL<br/>(Source of Truth)")]:::db

    UA & UC -->|WebSocket| TCS
    UB & UD -->|WebSocket| TCS
    TCS --> IMDS
    IMDS --> KAFKA
    KAFKA --> PG

    subgraph CONFLICT["Conflict Resolution: Field-Level LWW"]
        LWW["Write accepted only if version matches<br/>Higher timestamp wins on conflict<br/>User A: status=IN_PROGRESS (v=5, T1)<br/>User B: status=BLOCKED (v=5, T2)<br/>T2 > T1 → BLOCKED wins<br/>User A notified of change"]:::lww
    end

    subgraph WS_MSG["WebSocket Message Format"]
        MSG["{<br/>  type: 'FIELD_UPDATED',<br/>  taskId: 'abc-123',<br/>  field: 'status',<br/>  value: 'IN_PROGRESS',<br/>  actorId: 'user-456',<br/>  version: 6,<br/>  timestamp: '2026-06-24T10:30:00Z'<br/>}"]:::msg
    end

    IMDS --> CONFLICT

    classDef user fill:#e1f5fe,stroke:#01579b,color:#000,stroke-width:2px
    classDef svc fill:#fff3e0,stroke:#e65100,color:#000,stroke-width:2px
    classDef store fill:#e8f5e9,stroke:#1b5e20,color:#000
    classDef kafka fill:#f3e5f5,stroke:#4a148c,color:#000
    classDef db fill:#fce4ec,stroke:#b71c1c,color:#000
    classDef lww fill:#e8f5e9,stroke:#1b5e20,color:#000,stroke-dasharray: 5 5
    classDef msg fill:#f3e5f5,stroke:#4a148c,color:#000,stroke-dasharray: 5 5
```

### Deep Technical QA

#### 4. Interview Preparation: Multi-Level QA

##### 🟢 Basic Level (Jr. Engineer / 0-2 Yrs)

**Q1: What is the difference between functional and non-functional requirements in system design?**

**A1:** Functional requirements describe **what** the system should do — specific features and behaviors. Example: "Users can create, update, and delete tasks; assign tasks to team members; search across projects." Non-functional requirements describe **how** the system should perform — quality attributes. Example: "p95 latency < 200ms, 99.95% availability, RPO = 0 (zero data loss), SOC 2 compliance." Non-functional requirements drive architectural decisions (caching, replication, multi-region deployment).

**Q2: What are the key considerations for multi-tenant data isolation in a SaaS application?**

**A2:** Three main approaches:
1. **Database per tenant** — strongest isolation, easiest to restore, but costly for many tenants.
2. **Schema per tenant** — shared database, separate schemas; good balance for mid-size deployments.
3. **Shared schema with tenant_id column** — most cost-effective, but requires careful indexing (`WHERE tenant_id = ?`) and row-level security policies to prevent cross-tenant data leaks.

For the Project Tracking App, **shared schema with tenant_id** is chosen because 50K DAU doesn't justify per-tenant databases. All tables include `org_id` (organization ID) as a partitioning key.

##### 🟡 Intermediate Level (Mid-Level / 2-5 Yrs)

**Q1: Implement an authentication service with JWT access tokens and refresh tokens. Show the refresh flow.**

**A1:**
```java
public AuthResponse authenticate(LoginRequest request) {
    User user = userRepository.findByEmail(request.getEmail())
        .orElseThrow(() -> new AuthenticationException("Invalid credentials"));
    if (!passwordEncoder.matches(request.getPassword(), user.getPasswordHash())) {
        throw new AuthenticationException("Invalid credentials");
    }
    String accessToken = tokenProvider.generateAccessToken(
        user.getId(), user.getOrganizationId(), user.getRoles());
    String refreshToken = tokenProvider.generateRefreshToken(user.getId());
    redisTemplate.opsForValue().set(
        "refresh:" + user.getId(),
        hashRefreshToken(refreshToken),
        Duration.ofDays(7));
    return new AuthResponse(accessToken, refreshToken, tokenProvider.getExpiry());
}

public AuthResponse refreshAccessToken(String refreshToken) {
    Claims claims = tokenProvider.validateRefreshToken(refreshToken);
    String userId = claims.getSubject();
    String storedHash = redisTemplate.opsForValue().get("refresh:" + userId);
    if (storedHash == null || !storedHash.equals(hashRefreshToken(refreshToken))) {
        throw new AuthenticationException("Refresh token revoked or expired");
    }
    User user = userRepository.findById(UUID.fromString(userId)).orElseThrow();
    String newAccessToken = tokenProvider.generateAccessToken(
        user.getId(), user.getOrganizationId(), user.getRoles());
    return new AuthResponse(newAccessToken, refreshToken, tokenProvider.getExpiry());
}
```
Design decisions: refresh token hash stored in Redis (enables revocation), short-lived access tokens (15 min), auto-extending refresh tokens.

**Q2: Design the PostgreSQL schema for a task management system with support for drag-and-drop ordering. How do you handle sorting?**

**A2:**
```sql
CREATE TABLE tasks (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id  UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    title       VARCHAR(500) NOT NULL,
    status      task_status NOT NULL DEFAULT 'TODO',
    sort_order  FLOAT NOT NULL DEFAULT 0,  -- For drag-and-drop
    version     BIGINT NOT NULL DEFAULT 1,  -- Optimistic lock
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_tasks_project ON tasks(project_id, status, sort_order);
```
For drag-and-drop, use floating-point sort_order: when moving between two tasks with `order=1.0` and `order=2.0`, assign `order=1.5`. Periodic rebalancing (`UPDATE tasks SET sort_order = row_number() OVER (...)`) keeps precision under control. Optimistic locking via `@Version` prevents lost updates when two users drag simultaneously.

**Q3:** How do you configure HikariCP connection pooling for optimal database performance in the Task Service?

**A3:**
```yaml
spring.datasource.hikari:
  maximum-pool-size: 50
  minimum-idle: 10
  connection-timeout: 5000
  idle-timeout: 300000
  max-lifetime: 600000
  leak-detection-threshold: 60000
  pool-name: TaskServicePool
```
The formula for optimal pool size: `pool_size = Tn × (Cm - 1) + 1` where Tn = thread count and Cm = core count. For PostgreSQL with 8 cores and 50 application threads, ~50 connections is optimal — going higher would cause context-switching overhead. Configure `connection-timeout` to fail fast (5s) instead of queuing indefinitely. Set `max-lifetime` below any firewall/router idle timeout (e.g., 600s for AWS NLB). Enable `leak-detection-threshold` to identify connection leaks in development. Monitor via Micrometer + Prometheus: `hikaricp_connections_active` (active connections), `hikaricp_connections_pending` (threads waiting for a connection), `hikaricp_connections_timeout_total` (connection acquisition failures). A spike in `hikaricp_connections_pending` indicates the pool is too small or the database is slow.

##### 🔴 Advanced Level (Senior / 5-8 Yrs)

**Q1: Design the "task activity feed" that shows chronological events across all tasks in a project. How do you handle pagination, event ordering, and storage efficiency for 10M+ events per project?**

**A1:** Use **Event Sourcing with Materialized Aggregates**:

```sql
-- Raw event store (append-only, partitioned by month)
CREATE TABLE task_events (
    id              BIGSERIAL,
    task_id         UUID NOT NULL,
    project_id      UUID NOT NULL,
    event_type      VARCHAR(50) NOT NULL,
    actor_id        UUID NOT NULL,
    payload         JSONB NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, occurred_at)
) PARTITION BY RANGE (occurred_at);

CREATE TABLE task_events_2026_06 PARTITION OF task_events
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_task_events_project_time
    ON task_events(project_id, occurred_at DESC);

-- Cursor-based pagination query
SELECT task_id, event_type, actor_id, payload, occurred_at
FROM task_events
WHERE project_id = $1 AND occurred_at < $2
ORDER BY occurred_at DESC LIMIT 50;
```
Storage: ~500 bytes/event × 10M = 5GB/month/project. Retention: 90d hot, 1y warm (S3), indefinite cold (Glacier). Ordering: Hybrid Logical Clocks (HLC) for clock skew tolerance. Pagination: cursor-based using `(occurred_at DESC, id DESC)` — stable under concurrent inserts.

**Q2: Implement the WebSocket connection manager for 50K concurrent connections across 10 instances. Handle reconnection and state recovery after a crash.**

**A2:** Netty EventLoop model — each connection is bound to one thread for its lifetime (lock-free per connection):
```java
@Component
public class WebSocketConnectionManager {
    private final ConcurrentHashMap<String, WebSocketSession> localSessions = new ConcurrentHashMap<>();
    private final RedisTemplate<String, String> globalRegistry;

    public void registerSession(WebSocketSession session, UUID userId, UUID projectId) {
        localSessions.put(session.getId(), session);
        String key = "ws:user:" + userId + ":project:" + projectId;
        globalRegistry.opsForValue().set(key, getLocalInstanceId() + ":" + session.getId());
        globalRegistry.expire(key, Duration.ofMinutes(30));
    }

    public void handleReconnection(String oldSessionId, WebSocketSession newSession) {
        List<String> missedEvents = redisTemplate.opsForList()
            .range("session:events:" + oldSessionId, 0, -1);
        missedEvents.forEach(e -> newSession.sendMessage(new TextMessage(e)));
        redisTemplate.delete("session:events:" + oldSessionId);
    }
}
```
Crash recovery: service registry (Consul/etcd) with 15s TTL, heartbeat every 5s. Clients auto-reconnect via `reconnecting-websocket` library, send last received event ID, and replay missed events from Redis buffer (TTL: 5 min). Hard failures fall back to full REST API state reload.

##### ⚫ Expert Level (Staff/Principal / 8+ Yrs)

**Q1: Walk through the performance characteristics of the search service. How do you handle partial updates, indexing lag, and search result freshness with Elasticsearch, while maintaining write throughput to PostgreSQL?**

**A1:** Search architecture uses CDC-based indexing: Debezium captures PostgreSQL WAL changes → Kafka → Elasticsearch (~500ms end-to-end).

**Partial updates** — Elasticsearch `upsert` with `doc_as_upsert` (only changed fields):
```java
public void updateTaskSearchIndex(TaskStatusChangedEvent event) {
    UpdateRequest request = new UpdateRequest("tasks", "_doc", event.getTaskId().toString());
    request.doc(jsonBuilder().startObject()
        .field("status", event.getNewStatus().name())
        .field("updated_at", event.getTimestamp().toString())
    .endObject());
    request.setRetryOnConflict(3);
    bulkProcessor.add(request);
}
```

**Performance tuning:**
- **Bulk indexing:** 500 docs/bulk request, 1 thread per data node
- **Refresh interval:** `index.refresh_interval: 30s` (10× I/O reduction; 30s max staleness)
- **Translog:** `index.translog.durability: async` (fsync every 5s)
- **Two-tier freshness:** critical fields (status, assignee) use `?refresh=wait_for` for <2s visibility; standard fields use 30s refresh
- **Circuit breaker:** JVM heap > 85% → reject new indexing, retry via Kafka DLQ

**Consistency:** at-least-once delivery via Kafka Connect + deduplication by `{lsn, seq}`. Nightly reconciliation job compares counts and reindexes missing documents if discrepancy > 0.1%.

**Q2: Explain the optimistic concurrency control strategy used in the Task Service to prevent lost updates. How does it interact with Kafka event publishing for real-time sync?**

**A2:** The Task entity uses JPA `@Version` (optimistic locking). When two users update the same task simultaneously, Hibernate checks the version column on flush — the second writer gets `OptimisticLockException`:

```java
public Task updateTaskStatus(UUID taskId, TaskStatus newStatus, UUID userId, Long expectedVersion) {
    Task task = taskRepository.findById(taskId).orElseThrow();
    if (!isValidTransition(task.getStatus(), newStatus)) {
        throw new InvalidTransitionException(...);
    }
    try {
        task.setStatus(newStatus);
        taskRepository.save(task);  // Hibernate checks @Version
    } catch (OptimisticLockException e) {
        throw new ConcurrentModificationException(
            "Task was modified by another user. Please refresh.", e);
    }
    // Publish event AFTER successful DB write
    kafka.send("task-events", taskId.toString(),
        new TaskStatusChangedEvent(taskId, newStatus, ...));
    redis.delete("task:" + taskId);
    return task;
}
```

The critical pattern: **DB write first, then publish event**. This ensures the event is only emitted if the DB transaction succeeds. If the event publish fails, a background reconciliation job detects the inconsistency and re-publishes. The WebSocket notification system consumes from Kafka, so all subscribers eventually see the update even if the initial push fails.

---

*Reference: TrainWithShubham — Computer Networking & System Design For DevOps in OneShot (YouTube)*
