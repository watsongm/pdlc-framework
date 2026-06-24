---
product_name: ""
sprint: ""                # e.g., "Sprint 1" or "Phase 1 — Foundation"
spec_ref: "spec/spec.md"
design_ref: "spec/design.md"
generated_by: ""          # e.g., "architect-agent@1.0" or "human"
date: ""                  # YYYY-MM-DD
last_updated: ""
total_tasks: 0
completed_tasks: 0
---

# Agent Task Backlog: {{PRODUCT_NAME}}

> **Spec:** [spec/spec.md](../spec.md)  
> **Design:** [spec/design.md](../design.md)  
> **ADRs:** [adrs/](../../adrs/)  
> **Sprint:** {{SPRINT}}

---

## Task Format Reference

Each task uses the following structure. All fields are mandatory unless marked optional.

```
## TASK-NNN: [Task Title]
- Type: implementation | test | infrastructure | documentation
- Service Type: api | microservice | monolith | data-pipeline
- Priority: P1 (blocking) | P2 (important) | P3 (nice-to-have)
- Agent: dev-agent | test-agent | deploy-agent | doc-agent
- Autonomy: HITL | HOTL | HOOTL
- Spec Reference: [link to spec section]
- ADR References: [ADR IDs this task must respect]
- Acceptance Criteria: [numbered list — each item independently testable]
- Context Files: [files agent MUST load before starting this task]
- Dependencies: [TASK-NNN IDs that must be complete before this task starts]
- Estimated Complexity: S | M | L | XL
- Status: pending | in-progress | review | done
- Assignee: [optional — agent instance or human name]
- Notes: [optional — design clarifications, known edge cases]
```

**Autonomy tier definitions:**
- `HITL` — Human must review and approve the agent's output before it is committed
- `HOTL` — Human monitors; agent commits autonomously but human reviews async; can roll back
- `HOOTL` — Fully autonomous; output goes to audit log only; human is not in the loop unless alert triggered

**Complexity definitions:**
- `S` — ≤ 2 hours of agent work (simple CRUD, config change, documentation update)
- `M` — 2–6 hours (moderate logic, integration, multi-file change)
- `L` — 6–16 hours (complex feature, cross-cutting concern, multiple integration points)
- `XL` — >16 hours — **this task must be split into sub-tasks before starting**

---

## Task Decomposition Order

Tasks are listed in execution order. Do not start a task before its dependencies are resolved.

```
Infrastructure → Domain Model → Service Layer → API Layer → Integration Tests → Contract Tests → Documentation
```

---

## Tasks

---

## TASK-001: Create Flyway Baseline Schema Migration

- **Type:** infrastructure
- **Service Type:** api
- **Priority:** P1
- **Agent:** deploy-agent
- **Autonomy:** HITL
- **Spec Reference:** [spec.md — Data Model section](../spec.md#data-model); [spec.md — NFR-SEC](../spec.md#nfr-sec-security)
- **ADR References:** ADR-0003 (PostgreSQL as primary data store), ADR-0005 (Flyway for schema migrations)
- **Acceptance Criteria:**
  1. `src/main/resources/db/migration/V1__baseline.sql` exists and creates all tables defined in the Data Model section of the spec
  2. Each table includes: UUID primary key generated via `gen_random_uuid()`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`, `updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`, `version BIGINT NOT NULL DEFAULT 0` (optimistic locking)
  3. Appropriate indexes exist for all foreign keys and status columns
  4. `updated_at` auto-update trigger is created for each table
  5. All column constraints are explicit (NOT NULL, CHECK constraints for enum-like columns)
  6. `mvn flyway:migrate` runs to completion against a clean PostgreSQL 15 Testcontainers instance
  7. Flyway schema history table shows V1 as applied and successful
  8. `mvn flyway:validate` passes (schema matches Hibernate entity mapping)
- **Context Files:**
  - `spec/spec.md` (Data Model section)
  - `spec/design.md` (Data Layer Design section)
  - `adrs/ADR-0003-use-postgresql.md`
  - `adrs/ADR-0005-use-flyway-migrations.md`
  - `runtime-standards.md`
- **Dependencies:** None
- **Estimated Complexity:** M
- **Status:** pending
- **Notes:** Do NOT add columns that are not in the spec Data Model. If the spec is ambiguous about a column type, choose the most conservative type (e.g., `VARCHAR(255)` over `TEXT`) and flag the ambiguity in a comment in the SQL file.

---

## TASK-002: Implement Core Domain Model Entities and Value Objects

- **Type:** implementation
- **Service Type:** api
- **Priority:** P1
- **Agent:** dev-agent
- **Autonomy:** HOTL
- **Spec Reference:** [spec.md — Data Model section](../spec.md#data-model); [spec.md — FR-001 through FR-003](../spec.md#functional-requirements)
- **ADR References:** ADR-0001 (Java 21 + Spring Boot 3.x), ADR-0004 (Hexagonal architecture)
- **Acceptance Criteria:**
  1. Domain entity classes exist in `com.example.{{product}}.domain.model` package
  2. Entities use Java records for value objects (immutable, no setters)
  3. JPA entity classes (`@Entity`) exist separately in `com.example.{{product}}.infrastructure.persistence.entity` — domain model has NO JPA annotations
  4. `PersistenceMapper` exists to translate between domain objects and JPA entities
  5. All domain entities have: `id` (UUID), `createdAt` (Instant), `updatedAt` (Instant), `version` (long) for optimistic locking
  6. Domain entities implement `equals()` and `hashCode()` based on `id` only (not mutable fields)
  7. Unit tests exist for: entity creation, value object equality, any domain invariant methods
  8. Unit test coverage ≥ 80% for all classes in `domain.model` package
  9. No Spring annotations in `domain.model` or `domain.service` packages (only in `infrastructure` and `api` packages)
- **Context Files:**
  - `spec/spec.md` (Data Model and Functional Requirements sections)
  - `spec/design.md` (Spring Boot Application Structure section)
  - `adrs/ADR-0001-use-java21-spring-boot3.md`
  - `adrs/ADR-0004-hexagonal-architecture.md`
  - `src/main/resources/db/migration/V1__baseline.sql` (must exist — TASK-001 dependency)
- **Dependencies:** TASK-001
- **Estimated Complexity:** M
- **Status:** pending
- **Notes:** If a domain concept appears in the spec but has no corresponding database table, flag it — either TASK-001 is incomplete or the spec is ambiguous. Do not create entities for concepts not in the spec.

---

## TASK-003: Implement Service Layer with Business Logic

- **Type:** implementation
- **Service Type:** api
- **Priority:** P1
- **Agent:** dev-agent
- **Autonomy:** HOTL
- **Spec Reference:** [spec.md — Functional Requirements FR-001 to FR-NNN](../spec.md#functional-requirements); [spec.md — Error Handling Strategy](../spec.md#error-handling-strategy)
- **ADR References:** ADR-0001 (Java 21 + Spring Boot 3.x), ADR-0004 (Hexagonal architecture), ADR-0006 (Resilience4j for fault tolerance)
- **Acceptance Criteria:**
  1. Service interface exists in `com.example.{{product}}.domain.service` — no implementation details, only domain-language method signatures
  2. Service implementation exists in same package, annotated with `@Service`
  3. Each functional requirement (FR-NNN) in the spec has at least one corresponding method in the service interface
  4. All `@Transactional` annotations are on service methods only (not repository or controller)
  5. Write operations use `@Transactional`; read operations use `@Transactional(readOnly = true)`
  6. Custom exception classes exist in `domain.exception` package for each error case listed in the spec Error Handling Strategy
  7. Circuit breaker (`@CircuitBreaker`) is applied to all methods that call external services (per design.md Resilience Patterns section)
  8. Service methods emit domain events via `ApplicationEventPublisher` for state changes that the spec marks as significant
  9. Unit tests: all service methods tested with Mockito-mocked repository and client dependencies
  10. Test coverage ≥ 85% for service implementation class
  11. All edge cases from spec acceptance criteria have corresponding unit test methods
- **Context Files:**
  - `spec/spec.md` (Functional Requirements, Error Handling Strategy sections)
  - `spec/design.md` (Resilience Patterns section)
  - `adrs/ADR-0001-use-java21-spring-boot3.md`
  - `adrs/ADR-0004-hexagonal-architecture.md`
  - `adrs/ADR-0006-use-resilience4j.md`
  - `src/main/java/com/example/{{product}}/domain/` (TASK-002 output)
- **Dependencies:** TASK-002
- **Estimated Complexity:** L
- **Status:** pending
- **Notes:** If a functional requirement's business logic is ambiguous, implement the simplest interpretation, document the assumption in a code comment, and flag in the PR. Do not make business rule decisions autonomously.

---

## TASK-004: Implement REST API Controllers and DTOs

- **Type:** implementation
- **Service Type:** api
- **Priority:** P1
- **Agent:** dev-agent
- **Autonomy:** HOTL
- **Spec Reference:** [spec.md — API Contract section](../spec.md#api-contract); [spec.md — NFR-SEC](../spec.md#nfr-sec-security); [spec.md — Error Handling](../spec.md#error-handling-strategy)
- **ADR References:** ADR-0001 (Java 21 + Spring Boot 3.x), ADR-0002 (REST with JSON, URL versioning), ADR-0007 (OAuth 2.0 Bearer Token auth), ADR-0008 (No field injection — constructor injection only)
- **Acceptance Criteria:**
  1. One `@RestController` class per resource in `com.example.{{product}}.api.v1` package
  2. All endpoints listed in spec API Contract section are implemented with correct HTTP method, path, and status codes
  3. Request DTOs use Java records with Bean Validation annotations (`@NotNull`, `@NotBlank`, `@Size`, etc.) matching spec constraints
  4. Response DTOs use Java records; include `_links` section for HATEOAS (Spring HATEOAS)
  5. `@Valid` annotation on all `@RequestBody` parameters
  6. `GlobalExceptionHandler` (`@RestControllerAdvice`) maps all domain exceptions to the standard error response schema defined in the spec
  7. All controllers use constructor injection (no `@Autowired` field injection — verified by ADR-0008 compliance rule)
  8. `SecurityConfig` restricts all `/api/**` endpoints to authenticated requests; actuator endpoints follow spec NFR-SEC requirements
  9. CORS configuration is explicit — no wildcard origins in staging or production profiles
  10. OpenAPI annotations (`@Operation`, `@ApiResponse`, `@Tag`) are present on all controller methods
  11. Spring MVC Test (`MockMvc`) tests exist for: each endpoint happy path, each validation failure case, each 404 case, auth failure (401), and forbidden (403)
  12. Controller test coverage ≥ 90%
- **Context Files:**
  - `spec/spec.md` (API Contract, NFR-SEC, Error Handling sections)
  - `spec/design.md` (API Design Detail, Security Design sections)
  - `adrs/ADR-0002-rest-api-with-url-versioning.md`
  - `adrs/ADR-0007-oauth2-bearer-token.md`
  - `adrs/ADR-0008-constructor-injection-only.md`
  - `src/main/java/com/example/{{product}}/domain/service/` (TASK-003 output)
- **Dependencies:** TASK-003
- **Estimated Complexity:** L
- **Status:** pending
- **Notes:** Controllers must be thin — no business logic. If implementing a controller method requires more than a simple service delegation + mapping, stop and flag. The service layer needs more decomposition.

---

## TASK-005: Write Integration and Contract Tests

- **Type:** test
- **Service Type:** api
- **Priority:** P1
- **Agent:** test-agent
- **Autonomy:** HOTL
- **Spec Reference:** [spec.md — Test Strategy section](../spec.md#test-strategy); [spec.md — API Contract section](../spec.md#api-contract); [spec.md — Functional Requirements](../spec.md#functional-requirements)
- **ADR References:** ADR-0001 (Java 21 + Spring Boot 3.x), ADR-0009 (Testcontainers for integration tests), ADR-0010 (Pact for contract tests)
- **Acceptance Criteria:**
  1. Integration test base class `BaseIntegrationTest` uses `@SpringBootTest(webEnvironment = RANDOM_PORT)` with `@Testcontainers`
  2. PostgreSQL Testcontainer is declared as a `@Container` static field and registered via `DynamicPropertySource`
  3. Integration tests exist for: each service method that interacts with the database, verifying actual data persistence and retrieval
  4. Integration tests use `@Sql` annotation to seed test data and `@Transactional` (with rollback) for test isolation where appropriate
  5. Pact consumer test exists for each upstream service this product calls — defines the expected response contract
  6. Pact provider test exists — verifies that this service's API matches the contracts declared by downstream consumers
  7. All integration tests pass against a clean Testcontainer database (no leftover state between test classes)
  8. Contract tests produce Pact JSON files in `target/pacts/`
  9. `mvn test -Pintegration` runs all integration tests; `mvn test -Pcontract` runs all contract tests
  10. Integration test coverage adds ≥ 15 percentage points over unit test coverage for the service layer
  11. All FR-NNN acceptance criteria from the spec are covered by at least one integration or contract test
- **Context Files:**
  - `spec/spec.md` (Test Strategy, API Contract, Functional Requirements sections)
  - `spec/design.md` (Data Layer Design section)
  - `adrs/ADR-0009-testcontainers-integration-tests.md`
  - `adrs/ADR-0010-pact-contract-tests.md`
  - `src/main/java/com/example/{{product}}/` (all TASK-001 to TASK-004 outputs)
  - `src/main/resources/db/migration/` (for understanding schema)
- **Dependencies:** TASK-001, TASK-002, TASK-003, TASK-004
- **Estimated Complexity:** L
- **Status:** pending
- **Notes:** Contract tests are not optional. If an upstream service does not yet have a Pact broker, generate the Pact JSON and include instructions for publishing. Do not mock database calls in integration tests — that defeats the purpose. All DB interactions must use the real Testcontainer.

---

## Adding New Tasks

When adding tasks beyond these examples, follow the decomposition order:
1. Infrastructure tasks (DB, config, pipeline) — always P1
2. Domain model — always P1
3. Service layer — P1 for core features, P2 for secondary features
4. API layer — P1 for core endpoints
5. Integration tests — P1
6. Contract tests — P1
7. Documentation — P2 or P3

**XL tasks must be split.** If a task is estimated XL, break it into 2–4 sub-tasks, each ≤ L complexity, and add them to this file with a parent reference.

---

## Task Status Summary

| Status | Count |
|---|---|
| Pending | 5 |
| In Progress | 0 |
| Review | 0 |
| Done | 0 |
| **Total** | **5** |

---

*Template version: 1.0 | Framework: PDLC 1.0*
