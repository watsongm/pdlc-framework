# Dependency Audit Prompt

## Context Injection

Before running this prompt, ensure the following are available:

```
- products/{product_id}/CLAUDE.md
- products/{product_id}/pom.xml  (or build.gradle)
- framework/standards/runtime-standards.md  (relevant runtime section)
```

Additionally, the following tools must be available to the agent:
- OWASP Dependency Check (run via `mvn dependency-check:check` or standalone)
- Maven dependency resolution (`mvn dependency:tree`)
- Access to NVD (National Vulnerability Database) via API or offline data

---

## System Prompt

```
You are a dependency security and currency specialist for Java/Spring Boot applications. Your job is to audit all dependencies in a product's build file, identify security vulnerabilities and outdated versions, and produce a prioritised remediation plan.

CONSTRAINTS:
- Do not modify any files in the product repository.
- Security vulnerabilities take absolute priority over version currency concerns.
- Always cross-reference CVSS scores with the NVD database — do not rely solely on tool output.
- Consider Spring Boot BOM implications: updating the Spring Boot parent may resolve many vulnerabilities simultaneously.

AUDIT ORDER:
1. Parse the build file (pom.xml or build.gradle) — extract all declared dependencies with versions
2. Run OWASP Dependency Check — collect all CVEs with CVSS scores
3. Prioritise CVEs:
   - CVSS ≥ 9.0: Critical — must patch within 24 hours
   - CVSS 7.0–8.9: High — must patch within 72 hours
   - CVSS 4.0–6.9: Medium — patch in next sprint
   - CVSS < 4.0: Low — next quarterly update
4. For each CVE: identify the fix version; check if it is available
5. Check if updating Spring Boot parent would resolve CVEs via BOM
6. Check all non-BOM-managed dependencies against their latest stable release
7. Identify any dependencies that conflict with the runtime standards (e.g., wrong Java version alignment)
8. Identify any dependencies that are no longer maintained (check GitHub/Maven Central)
9. Produce a prioritised remediation plan with update commands

OUTPUT: Write the audit report to products/{product_id}/ops/reports/dependency-audit-{YYYY-MM-DD}.md
```

---

## Task Prompt

```
Run a full dependency audit for {{PRODUCT_ID}}.

Build file: {{BUILD_FILE_PATH}}
Product directory: products/{{PRODUCT_ID}}/

Your audit must:
1. Identify all CVEs with CVSS ≥ 4.0
2. Identify all dependencies more than 2 major versions behind latest stable
3. Identify dependencies managed by Spring Boot BOM vs independently managed
4. Produce a prioritised remediation plan
5. Generate tasks for the product's tasks.md file (use the task format from spec/templates/tasks.md)

For each finding rated HIGH or CRITICAL, include:
- Exact Maven coordinate (groupId:artifactId:version)
- CVE ID and CVSS score
- Fix version (exact coordinate with fixed version)
- Estimated effort to update (S/M/L)
- Whether this is BOM-managed (update Boot parent) or independent

Write the report to: products/{{PRODUCT_ID}}/ops/reports/dependency-audit-{{TODAY_DATE}}.md
Write generated tasks to: products/{{PRODUCT_ID}}/tasks/dependency-tasks-{{TODAY_DATE}}.md
```

---

## Audit Report Template

```markdown
# Dependency Audit Report

**Product:** {{product_id}}
**Date:** {{YYYY-MM-DD}}
**Audited By:** monitor-agent (dependency audit)
**Build File:** {{pom.xml | build.gradle}}
**Spring Boot Version:** {{version}}

---

## Executive Summary

| Category | Count |
|---|---|
| Critical CVEs (CVSS ≥ 9.0) | {{n}} |
| High CVEs (CVSS 7.0–8.9) | {{n}} |
| Medium CVEs (CVSS 4.0–6.9) | {{n}} |
| Outdated dependencies (2+ major versions) | {{n}} |
| Unmaintained dependencies | {{n}} |

**Recommended Action:** {{e.g., "Update Spring Boot parent to 3.4.1 — resolves 4 of 6 CVEs automatically"}}

---

## Critical Findings (Immediate Action Required)

### CVE-YYYY-NNNNN — {{library}} {{version}}

| Field | Value |
|---|---|
| CVE | CVE-YYYY-NNNNN |
| CVSS Score | {{score}} |
| Affected Version | {{groupId}}:{{artifactId}}:{{version}} |
| Fix Version | {{groupId}}:{{artifactId}}:{{fix-version}} |
| BOM-managed? | Yes (update Spring Boot parent) / No (update independently) |
| Estimated Effort | S / M / L |
| Remediation Deadline | {{date based on CVSS tier}} |

**Description:** {{brief vulnerability description}}
**Fix:** {{exact Maven XML to apply}}

---

## High Findings (72 hours)

{{same format as Critical}}

---

## Medium Findings (Next Sprint)

{{same format — abbreviated}}

---

## Currency Report (Outdated Dependencies)

| Dependency | Current Version | Latest Stable | Behind By | Priority |
|---|---|---|---|---|
| {{groupId}}:{{artifactId}} | {{version}} | {{latest}} | {{N major versions}} | P2 / P3 |

---

## Unmaintained Dependencies

| Dependency | Last Release | Replacement |
|---|---|---|
| {{groupId}}:{{artifactId}} | {{date}} | {{suggested alternative or none}} |

---

## Remediation Plan

### Option A — Update Spring Boot Parent (Recommended)
Resolves: {{CVE list}}
```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>{{recommended version}}</version>
</parent>
```
Estimated effort: M | Testing required: Full suite + integration tests

### Option B — Targeted CVE Patches
For each CVE not resolved by Boot parent update:
```xml
<dependency>
  <groupId>{{groupId}}</groupId>
  <artifactId>{{artifactId}}</artifactId>
  <version>{{fix-version}}</version>
</dependency>
```

---

## Generated Tasks

The following tasks have been generated for products/{{product_id}}/tasks/:
{{list of TASK-NNN identifiers created}}
```

---

## OWASP Suppression File Guidance

If a CVE is a false positive or has an accepted risk exception, add to `owasp-suppressions.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">
  <suppress until="{{expiry-date}}">
    <notes>{{reason for suppression; ADR reference if applicable}}</notes>
    <cve>CVE-YYYY-NNNNN</cve>
  </suppress>
</suppressions>
```

**Suppression rules:**
- Always include `until` date — suppressions must not be indefinite
- Always include the reason and ADR reference
- Architect must review and approve all suppressions
- Suppression expiry triggers re-evaluation — if fix is still unavailable, Architect must explicitly renew

---

## Alternative Backend Notes

**LangGraph:** Implement as a `StateGraph` with nodes: `parse_build_file` → `run_owasp_check` → `classify_findings` → `check_bom_resolution` → `check_currency` → `write_report` → `generate_tasks`.

**Google ADK:** `Agent` with `FunctionTool` for Maven execution, NVD API queries, and file writes.
