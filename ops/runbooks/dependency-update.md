# Dependency Update Runbook

## Purpose
This runbook defines how dependencies are managed across the lifecycle of a Java/Spring Boot product managed by the AI-Native PDLC Framework — from automated scanning through update, test, and production deployment.

---

## Dependency Categories

| Category | Examples | Criticality | Update Cadence |
|---|---|---|---|
| **Spring Boot parent version** | `3.3.x → 3.4.x` | High | Quarterly (minor); Immediately for security patches |
| **Spring dependencies** | Spring Security, Spring Data, Spring Cloud | High | Managed by Boot BOM — update with Boot |
| **Core third-party libraries** | Flyway, Hibernate, Jackson, Resilience4j | Medium-High | Quarterly |
| **Secondary libraries** | Lombok, MapStruct, Testcontainers | Medium | Bi-annually |
| **Docker base image** | `gcr.io/distroless/java21-debian12` | High | Monthly (check for OS patches) |
| **JDK (on VCF VMs)** | Eclipse Temurin 21.x.y | High | Quarterly (patch releases) |

---

## Automated Scanning

The Monitor Agent runs OWASP Dependency Check on a weekly schedule for all active products:

```bash
# Run via Harness dependency-audit pipeline
# Triggered: weekly Sunday 02:00 UTC
# Agent: monitor-agent + dependency-audit prompt

mvn org.owasp:dependency-check-maven:check \
  -DfailBuildOnCVSS=7 \
  -Dformats=JSON,HTML \
  -DsuppressFile=owasp-suppressions.xml
```

Results are written to `products/<name>/ops/reports/dependency-audit-{date}.md` and reviewed by the Monitor Agent, which:
- Classifies findings by CVSS score
- Cross-references with NVD for patches
- Creates tasks in `tasks.md` for each finding that requires action
- Notifies Ops via Slack for CVSS ≥ 7.0 findings

---

## Update Process

### Step 1 — Discovery (Monitor Agent, HOOTL)
Monitor Agent produces a dependency audit report:
- CVE findings (CVSS score, affected version, fix version, remediation deadline)
- Outdated dependencies (current version vs latest stable)
- Breaking change warnings (major version bumps)
- Recommended update order (dependency on each other's versions)

### Step 2 — Task Creation (Engineer, HOTL)
Engineer reviews the audit report and creates tasks in `tasks.md`:
```markdown
## TASK-NNN: Update Spring Boot from 3.3.2 to 3.4.0
- Type: implementation
- Priority: P2
- Agent: dev-agent
- Autonomy: HOTL
- ADR References: ADR-0001-java-springboot-runtime
- Acceptance Criteria:
  1. pom.xml parent version updated to 3.4.0
  2. All tests pass (mvn verify)
  3. No new OWASP findings introduced
  4. Application starts successfully in staging
```

### Step 3 — Implementation (Dev Agent, HOTL)
Dev Agent updates `pom.xml`:
```xml
<!-- Before -->
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.3.2</version>
</parent>

<!-- After -->
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.4.0</version>
</parent>
```

Agent then runs `mvn dependency:tree` and checks for version conflicts introduced by the update.

### Step 4 — Test (Test Agent, HOTL)
Full test suite run:
```bash
mvn verify \
  -Pdependency-check \
  -Dfailsafe.failIfNoSpecifiedTests=false
```

### Step 5 — Review (Review Agent, HOOTL)
Review Agent checks:
- No new OWASP CVEs introduced
- No API breaking changes (check springdoc-openapi output diff)
- All existing tests pass
- Dependency tree has no version conflicts

### Step 6 — Human Approval
| Update Type | Gate | Approver |
|---|---|---|
| Patch version (3.3.2 → 3.3.3) | HOTL | Engineer sign-off |
| Minor version (3.3.x → 3.4.x) | HITL | Engineer + Architect review |
| Major version (3.x → 4.x) | HITL | Full pod review; new ADR required |
| Critical CVE patch (CVSS ≥ 9.0) | HITL (expedited) | Architect authority; same day |

### Step 7 — Staging Deployment & Validation
Deploy Agent triggers staging pipeline; integration tests run against staging with updated dependencies.

### Step 8 — Production Deployment
Standard production deployment process (HITL gate in Harness).

---

## Maven BOM Management

The Spring Boot BOM manages versions for all Spring ecosystem dependencies. Never override Spring dependency versions manually unless an ADR justifies it.

```xml
<!-- DO: use the BOM, declare no version -->
<dependency>
  <groupId>org.springframework.security</groupId>
  <artifactId>spring-security-test</artifactId>
  <scope>test</scope>
  <!-- NO <version> tag — managed by Boot BOM -->
</dependency>

<!-- DON'T: override BOM-managed versions -->
<dependency>
  <groupId>org.springframework.security</groupId>
  <artifactId>spring-security-test</artifactId>
  <version>6.1.0</version> <!-- This will cause version conflicts -->
</dependency>
```

When the Spring Boot parent version is updated, all Spring dependencies update automatically through the BOM. This is the primary benefit of BOM management.

---

## CVE Handling

| CVSS Score | Classification | Action | SLA |
|---|---|---|---|
| 9.0 – 10.0 | Critical | Immediate patch; emergency process if needed | 24 hours |
| 7.0 – 8.9 | High | Expedited patch | 72 hours |
| 4.0 – 6.9 | Medium | Planned update in next sprint | 2 weeks |
| 0.1 – 3.9 | Low | Next quarterly update | Next quarter |

**CVE exception process:** If a fix is not yet available (zero-day), the Architect must:
1. Create an OWASP suppression rule with a time-bound expiry
2. Write an ADR documenting the accepted risk
3. Implement compensating controls where feasible (WAF rule, input filtering)
4. Set a calendar reminder for weekly check on fix availability

---

## Quarterly Audit Cadence

On the first Monday of each quarter:
1. Monitor Agent runs full dependency audit across all products
2. Ops produces a portfolio-wide report: number of outdated deps, CVE count, by product
3. Orchestrator prioritises dependency update tasks in the quarterly planning session
4. Architect reviews any BOM-managed dependency that has drifted from the framework standard

---

*Owner: Ops/Platform + Engineer | Triggered by: Monitor Agent (weekly) + Manual (quarterly)*
