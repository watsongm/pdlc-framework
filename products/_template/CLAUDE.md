# CLAUDE.md — {{PRODUCT_NAME}}

> **IMPORTANT FOR AGENTS:** This is the primary context file for {{PRODUCT_NAME}}. Load this file first. It provides the product overview, architecture summary, active ADRs, and agent instructions. Keep this file current — stale context is the leading cause of agent output degradation.
>
> **Last Updated:** {{YYYY-MM-DD}} by {{author}}
> **Context Freshness Review Due:** {{YYYY-MM-DD + 30 days}}

---

## Product Overview

**Name:** {{PRODUCT_NAME}}
**Service Type:** `api` | `microservice` | `monolith` | `data-pipeline`  ← choose one
**Runtime:** `java-springboot` | `other` ← choose one; if other, specify version
**Framework Version:** Java {{version}} + Spring Boot {{version}}
**Status:** `onboarding` | `spec` | `build` | `ops`  ← current phase
**Entry Point:** `discovery` | `greenfield`  ← which onboarding path was used
**Horizon Level:** `H1` | `H2` | `H3` | `H4`  ← current McKinsey Horizon

**One-line description:**
{{What this service does and for whom — maximum 1 sentence}}

**Repo:** {{GitHub/GitLab URL}}
**Primary language:** Java {{version}}
**Build tool:** Maven {{version}} | Gradle {{version}}

---

## Team

| Role | Name | Responsibilities |
|---|---|---|
| Orchestrator | {{name}} | Spec approval, stakeholder communication, agent governance |
| Architect | {{name}} | ADR ownership, design review, spec validation |
| Engineer | {{name}} | PR review, context engineering, prompt refinement |
| Ops | {{name}} (or shared) | Deployment, monitoring, cost tracking |

**On-call rotation:** {{link to PagerDuty/OpsGenie or "N/A"}}
**Escalation contact:** {{Orchestrator phone/Slack for P1}}

---

## Architecture Summary

> This section is the most important for agents. Keep it current. If this drifts from the code, agent output quality degrades immediately.

### What This Service Does
{{2-5 sentences describing the business domain this service covers}}

### Service Topology
{{Brief description: what calls this service, what this service calls}}
- **Called by:** {{upstream services or clients}}
- **Calls:** {{downstream services and their protocols}}
- **Owns data in:** {{database name, type}}
- **Publishes events to:** {{topic/queue names or "N/A"}}
- **Consumes events from:** {{topic/queue names or "N/A"}}

### Key Domain Concepts
{{List 3-5 core domain entities that agents must understand to work with this service}}
- **{{Entity1}}:** {{1-line definition}}
- **{{Entity2}}:** {{1-line definition}}

### Important Business Rules
{{List any business rules agents must never violate when generating code}}
1. {{Rule 1}}
2. {{Rule 2}}

---

## Environment Configuration

### GCP GKE (if applicable)
- **Project:** {{gcp-project-id}}
- **Cluster:** {{cluster-name}} ({{region}})
- **Staging namespace:** `{{product-name}}-staging`
- **Production namespace:** `{{product-name}}-prod`
- **Workload Identity SA:** `{{k8s-sa}}@{{project}}.iam.gserviceaccount.com`
- **Change freeze:** `false`  ← set to `true` during freeze windows

### VMware VCF (if applicable)
- **vCenter:** {{vcenter-host}}
- **Staging environment:** {{vcf-staging-env-name}}
- **Production environment:** {{vcf-production-env-name}}
- **Change freeze:** `false`

### Harness
- **Account:** {{account-id}}
- **Org:** {{org-id}}
- **Project:** {{project-id}}
- **Pipeline — staging deploy:** {{pipeline-id}}
- **Pipeline — production deploy:** {{pipeline-id}}
- **Pipeline — incident triage:** {{pipeline-id}}

---

## Active Architecture Decisions

> Agents: you MUST read these ADRs before generating any code. Non-compliance will cause the PR to fail the CI check.

| ADR ID | Decision | Status | File |
|---|---|---|---|
| ADR-0001 | {{Runtime: Java 21 + Spring Boot 3.x}} | Accepted | [ADR-0001](adrs/ADR-0001-java-springboot-runtime.md) |
| ADR-0002 | {{e.g., Database: PostgreSQL via Spring Data JPA}} | Accepted | [ADR-0002](adrs/ADR-0002-postgresql.md) |
| ADR-0003 | {{e.g., Auth: JWT Bearer via Spring Security}} | Accepted | [ADR-0003](adrs/ADR-0003-jwt-auth.md) |

**Compliance rules summary:** [adrs/compliance-rules-summary.md](adrs/compliance-rules-summary.md) ← read this for CI lint rules

---

## Agent Instructions

### For All Agents
- Always load this CLAUDE.md first
- Always load ALL ADRs from the `adrs/` directory before generating code
- Always check `tasks/tasks.md` to understand what has already been done
- Never commit or push directly to `main` — always create a branch and PR
- Never hardcode any credentials, API keys, or secrets — use environment variable references only
- When uncertain, STOP and escalate — do not guess

### For the Discovery Agent
- Repo path: {{local repo path}}
- Known tech stack: `java-springboot`
- Service type: {{service_type}}
- Output directory: `products/{{product_name}}/discovery/`

### For the Architect Agent
- Proposal location: `products/{{product_name}}/spec/proposal.md`
- Discovery outputs: `products/{{product_name}}/discovery/`
- ADR directory: `products/{{product_name}}/adrs/`
- Output: `products/{{product_name}}/spec/`

### For the Dev Agent
- Tasks are in: `products/{{product_name}}/tasks/tasks.md`
- Always implement tasks in dependency order
- Default package structure: `com.{{org}}.{{product_name}}.{controller|service|repository|domain|config}`
- Spring profile for development: `default` (H2 in-memory)
- Spring profile for staging: `staging`
- Spring profile for production: `production`

### For the Deploy Agent
- Harness pipeline IDs: see Environment Configuration above
- Staging: HOTL — proceed after health check passes
- Production: HITL — always wait for human approval in Harness

### For the Incident Agent
- Check deployment history in Harness project {{project-id}}
- GCP Cloud Logging filter prefix: `resource.labels.namespace_name="{product_name}-prod"`
- vROps resource group: `{{vcf-resource-group}}` (if VCF)
- SLA targets: p99 latency < {{latency_sla_ms}}ms, error rate < {{error_rate_sla}}%, availability {{availability_sla}}%

---

## Non-Functional Requirements

| NFR | Target | SLA |
|---|---|---|
| Availability | {{e.g., 99.9%}} | {{monthly}} |
| p99 Latency | {{e.g., 200ms}} | {{per request}} |
| Throughput | {{e.g., 500 RPS}} | {{sustained}} |
| RTO | {{e.g., 30 min}} | {{per P1 incident}} |
| RPO | {{e.g., 1 hour}} | {{data loss tolerance}} |

---

## Known Technical Debt

> Updated by agents and humans during discovery and ongoing development.

| ID | Description | Impact | Priority |
|---|---|---|---|
| DEBT-001 | {{e.g., No contract tests for downstream payment-service}} | Medium | P2 |

---

## Context Refresh History

| Date | Updated By | What Changed |
|---|---|---|
| {{YYYY-MM-DD}} | {{Human/Agent}} | Initial CLAUDE.md created |

---

*Context owner: Engineer + Architect | Review cadence: Monthly or after significant architecture changes*
