# Runtime Standards

This document defines the technical standards that govern how services are built within the AI-Native PDLC Framework. Each product declares its runtime in `proposal.md`; agents use this file to apply the correct standards when generating, reviewing, and testing code.

---

## How Runtime Standards Are Used

1. **Proposal declaration** — `proposal.md` frontmatter field `runtime: java-springboot` (or other) selects the active standard block.
2. **Agent context injection** — all agents load the relevant runtime section from this file via CLAUDE.md's context injection list.
3. **Discovery heuristics** — the Discovery Agent uses the runtime-specific section to know what to look for in an existing codebase.
4. **Code generation constraints** — the Dev Agent uses this document to make technology decisions (dependencies, patterns, tooling) that are consistent across the portfolio.
5. **Review Agent checks** — the Review Agent uses compliance rules from this document alongside ADRs.

---

## Runtime: `java-springboot` (Primary)

### Overview
Java 21 LTS + Spring Boot 3.x is the primary and default runtime for all services in this framework. It is the most mature, best-supported runtime across the portfolio and provides strong observability, security, and deployment tooling for both GCP GKE and VMware VCF targets.

---

### Language & Framework Versions

| Component | Version | Notes |
|---|---|---|
| Java | **21 LTS** | Minimum. Virtual threads (Project Loom) enabled by default. |
| Spring Boot | **3.3.x** (latest patch) | Minimum 3.0. Use Spring Boot BOM for all dependency management. |
| Spring Framework | Managed by Boot BOM | Do not declare independently. |
| Build Tool | **Maven 3.9+** (primary) / Gradle 8+ (supported) | Maven preferred for predictable, agent-generatable configs. |

---

### Required Spring Boot Starters (Baseline)

All services must include these starters unless an ADR explicitly justifies an exception:

```xml
<!-- pom.xml — baseline dependencies -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<!-- Observability -->
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-stackdriver</artifactId>
</dependency>
```

**Optional starters** (declared in `design.md` and justified by ADR):
- `spring-boot-starter-data-jpa` — relational data access
- `spring-boot-starter-data-r2dbc` — reactive relational data access
- `spring-boot-starter-data-mongodb` — document store
- `spring-boot-starter-amqp` — RabbitMQ messaging
- `spring-cloud-starter-openfeign` — declarative HTTP clients
- `spring-batch-core` — batch processing (data pipeline service type)

---

### Code Style & Quality

| Concern | Standard | Enforcement |
|---|---|---|
| Code formatting | **Google Java Format** | Maven plugin in `verify` phase; Harness CI fails on violation |
| Static analysis | **SpotBugs** + **PMD** | Maven plugin; warnings-as-errors for high severity |
| Dependency vulnerabilities | **OWASP Dependency Check** | Harness CI gate; CVSS ≥7.0 blocks deployment |
| Unused dependencies | **Maven Dependency Analyzer** | Advisory; flagged in PR review |

---

### Coding Patterns (Agent Compliance Rules)

These are the machine-readable constraints agents must follow. They are also registered as ADRs.

| Rule ID | Constraint | Rationale |
|---|---|---|
| `JAVA-001` | Use **constructor injection** only. No `@Autowired` field injection. | Testability; immutability; makes dependencies explicit |
| `JAVA-002` | Use `@ConfigurationProperties` for all configuration. No `@Value` for multi-value config. | Type safety; IDE support; testability |
| `JAVA-003` | Use `@Valid` on all controller method parameters that accept request bodies. | Fail-fast input validation |
| `JAVA-004` | No business logic in `@RestController` classes. Controllers delegate to `@Service` layer. | Separation of concerns; testability |
| `JAVA-005` | No raw SQL in service or controller layers. Use Spring Data repositories or named queries. | Maintainability; SQL injection prevention |
| `JAVA-006` | All `@Service` classes must emit at least one Micrometer `Counter` or `Timer` for business operations. | Observability |
| `JAVA-007` | Use SLF4J for all logging. Log with structured fields (MDC). Never log secrets or PII. | Structured observability; compliance |
| `JAVA-008` | `@Transactional` on service methods only, never on controllers or repositories. | Correct transaction boundary |
| `JAVA-009` | All exceptions must be handled; never swallow exceptions silently. Use `@ControllerAdvice` for global error handling. | Reliability; debuggability |
| `JAVA-010` | Use Java 21 virtual threads (`spring.threads.virtual.enabled=true`) for thread-per-request model. | Performance under load |

---

### API Contract Standard

- **Specification format:** OpenAPI 3.1 (generated via `springdoc-openapi`)
- **Spec endpoint:** `/v3/api-docs` (JSON), `/v3/api-docs.yaml` (YAML)
- **UI endpoint:** `/swagger-ui.html` (disabled in production; enabled in dev/staging)
- **Versioning:** URI versioning (`/api/v1/...`); breaking changes require new version + ADR
- **Response envelope:** Consistent error response structure using `ProblemDetail` (RFC 9457, built into Spring 6)

```java
// Standard error response — use Spring's built-in ProblemDetail
@ControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    // Override handleMethodArgumentNotValid, etc.
}
```

---

### Configuration Management

```yaml
# application.yml structure — standard layout
spring:
  application:
    name: ${SERVICE_NAME}  # Injected via env var
  datasource:
    url: ${DB_URL}         # Always from env/secret; never hardcoded
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:default}

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      probes:
        enabled: true       # Enables /actuator/health/liveness and /readiness
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

**Profile conventions:**
- `default` — local development (H2 in-memory DB, no auth)
- `staging` — staging environment (real DB, auth enabled, verbose logging)
- `production` — production (all security enabled, JSON logging, reduced actuator exposure)

Kubernetes ConfigMaps and Secrets are mapped to environment variables and consumed via Spring's environment abstraction. Never mount secrets as files unless security policy requires it.

---

### Testing Standards

| Test Type | Framework | Scope | Coverage Target |
|---|---|---|---|
| Unit | JUnit 5 + Mockito | Single class, all dependencies mocked | Line ≥80%, Branch ≥70% |
| Controller slice | `@WebMvcTest` + MockMvc | Single controller; mock service layer | All endpoints covered |
| Repository slice | `@DataJpaTest` + H2 or Testcontainers | Repository queries | All custom queries |
| Integration | `@SpringBootTest` + Testcontainers | Full application context, real DB | Critical user journeys |
| Contract | Spring Cloud Contract or Pact | Consumer/producer contracts | All public API contracts |

**Key testing rules:**
- Each `@Service` class has a corresponding `*Test.java` unit test class
- Each `@RestController` has a corresponding `@WebMvcTest` slice test
- Integration tests use **Testcontainers** (not mocked infrastructure)
- Test class naming: `MyServiceTest` (unit), `MyServiceIT` (integration)
- Maven Failsafe plugin runs `*IT.java` tests in the `integration-test` phase

---

### Observability

**Metrics (Micrometer):**
```java
// Standard metric naming: {service}.{domain}.{operation}
Counter.builder("orders.placed")
    .description("Total orders placed")
    .tag("status", "success")
    .register(meterRegistry)
    .increment();

Timer.builder("payments.processed")
    .description("Payment processing duration")
    .register(meterRegistry)
    .record(() -> processPayment(order));
```

**Logging (SLF4J + Logback JSON):**
```xml
<!-- logback-spring.xml — JSON format for GCP Cloud Logging -->
<springProfile name="staging,production">
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
  </appender>
</springProfile>
```

Required MDC fields per log statement: `traceId`, `spanId`, `userId` (if authenticated), `serviceVersion`.

**Health indicators:**
- `/actuator/health/liveness` — used by Kubernetes liveness probe
- `/actuator/health/readiness` — used by Kubernetes readiness probe
- Custom `HealthIndicator` beans for each external dependency (DB, message broker, downstream services)

**Distributed tracing:**
- Spring Boot 3 auto-configures Micrometer Tracing with Brave/OpenTelemetry
- Export to GCP Cloud Trace: `micrometer-tracing-bridge-brave` + `zipkin-reporter-sender-stackdriver`
- Sampling rate: 10% production, 100% staging

---

### Containerisation

**Base image:** `gcr.io/distroless/java21-debian12` (production). Never use `openjdk:21` or similar full JDK images in production.

**Multi-stage Dockerfile:**
```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Stage 2: Runtime
FROM gcr.io/distroless/java21-debian12
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**JVM flags (GKE):**
```
-XX:MaxRAMPercentage=75.0
-XX:+UseZGC
-XX:+ZGenerational
-Djava.security.egd=file:/dev/./urandom
```

---

### GKE Deployment Standard

**Helm chart structure (per service):**
```
helm/
├── Chart.yaml
├── values.yaml          ← defaults
├── values-staging.yaml  ← staging overrides
├── values-prod.yaml     ← production overrides
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    ├── configmap.yaml
    └── serviceaccount.yaml
```

**Standard resource requests/limits:**

| Service Type | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---|---|---|---|---|
| REST API | 250m | 1000m | 256Mi | 512Mi |
| Microservice | 100m | 500m | 128Mi | 256Mi |
| Data Pipeline (batch) | 500m | 2000m | 512Mi | 2Gi |

**HPA configuration:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**Workload Identity:** Each service must have a dedicated Kubernetes ServiceAccount bound to a GCP service account with least-privilege IAM roles. No shared service accounts.

**Probes:**
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 5
```

---

### VMware VCF Deployment Standard

For monoliths and data pipelines deployed on-premise via VMware VCF:

- **VM sizing:** Minimum 4 vCPU, 8GB RAM for monoliths; 2 vCPU, 4GB RAM for microservices
- **OS:** RHEL 9 or Ubuntu 22.04 LTS (standardise per organisation policy)
- **JVM:** Eclipse Temurin 21 (from Adoptium) installed via Ansible
- **Deployment:** Systemd service unit; managed by Ansible playbooks
- **NSX-T Network Policy:** Each service in its own security group; zero-trust between segments
- **Storage:** NFS-mounted volumes for batch pipeline input/output; local SSD for temp files
- **Health check:** Custom script calling `/actuator/health` exposed on internal network only
- **vSphere Tags:** Mandatory tags — `environment`, `service-name`, `team`, `runtime`, `pdlc-managed=true`

---

### Discovery Heuristics (for Discovery Agent)

When the Discovery Agent encounters a Java project, it should look for:

| Signal | Location | Meaning |
|---|---|---|
| `pom.xml` | root | Maven project; extract `<parent>` Spring Boot version, all `<dependency>` entries |
| `build.gradle` | root | Gradle project; extract `dependencies {}` block |
| `@SpringBootApplication` | `src/main/java/**` | Application entry point class |
| `@RestController` | `src/main/java/**` | REST endpoint class; extract all `@RequestMapping`, `@GetMapping`, etc. |
| `@FeignClient` | `src/main/java/**` | Downstream service dependency; extract `url` or `name` |
| `RestTemplate` / `WebClient` bean | `src/main/java/**` | HTTP client; map to downstream service |
| `application.yml` / `application.properties` | `src/main/resources/` | Config; extract keys (redact values matching secret patterns) |
| `flyway/` or `db/migration/` | `src/main/resources/` | Schema migrations; count files, identify current version |
| `V{n}__*.sql` files | `src/main/resources/db/` | Flyway migration; last version = current schema version |
| `@SpringBootTest` | `src/test/java/**` | Integration tests exist |
| `@WebMvcTest` | `src/test/java/**` | Controller slice tests exist |
| `Testcontainers` import | `src/test/java/**` | Real infrastructure in tests |
| `/actuator/info`, `/actuator/health` | HTTP (if running) | Live application metadata |
| `/v3/api-docs` | HTTP (if running) | OpenAPI spec; fetch and include in discovery report |
| `Dockerfile` | root or `docker/` | Containerised; inspect base image |
| `helm/` or `k8s/` | root | Kubernetes deployment; extract service topology |
| `Jenkinsfile` / `harness/` / `.harness/` | root | Existing CI/CD pipeline |

---

## Adding a New Runtime Standard

To add support for a new runtime (e.g., Python/FastAPI, Node/TypeScript, Go):

1. **Add a new section** to this file following the Java/Spring Boot template structure:
   - Overview
   - Versions
   - Required libraries/frameworks
   - Coding patterns & compliance rules
   - Testing standards
   - Observability
   - Containerisation
   - GKE/VCF deployment specifics
   - Discovery heuristics

2. **Update agent prompts** — add runtime-specific instructions to:
   - `onboarding/prompts/discovery-prompt.md` (new discovery heuristics section)
   - `build/prompts/implement-task.md` (new coding pattern guidance)
   - `build/prompts/generate-tests.md` (new test framework guidance)
   - `build/prompts/review-pr.md` (new anti-pattern checklist)

3. **Create a product template variant** — `products/_template-{runtime}/` with runtime-specific CLAUDE.md defaults

4. **Write an ADR** in `framework/adrs/` documenting the decision to add this runtime and its governance implications

5. **Update `framework/standards/agent-interface.md`** if the new runtime requires adapter changes

---

*Last updated: 2026-06 | Owner: Framework Architect | Review cadence: Quarterly or on major Spring Boot / Java LTS release*
