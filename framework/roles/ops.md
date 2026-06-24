# Role: Ops / Platform

> **Headcount:** 0–1 people | **Typical title:** Platform Engineer, DevOps Engineer, SRE | **Autonomy tier scope:** Owns all HITL gates for production deployments, schema migrations, and secret rotation. At 0-person headcount, these responsibilities are shared by the Orchestrator and Architect.

---

## Role Purpose

The Ops/Platform role is the **reliability and delivery infrastructure owner** of the AI-native pod. In a 3–5 person team, this role may be filled by a dedicated Platform Engineer or distributed across the Orchestrator and Architect. At minimum, the responsibilities described here must be owned — they cannot be delegated to agents autonomously.

The Ops role in an AI-native pod has a new dimension beyond traditional DevOps: **agent observability**. This means instrumenting and monitoring not just the product services, but the agent pipelines themselves — tracking agent run costs, failure rates, context quality signals, and multi-step agent failures.

### Scope of Accountability

| Domain | Ops Accountable | Ops Consulted | Ops Informed |
|---|---|---|---|
| Harness CI/CD pipeline health | Yes | Engineer | All |
| Production deployment approvals | Yes | Orchestrator | All |
| GCP GKE namespace governance | Yes | Architect | — |
| VMware VCF VM/container scheduling | Yes | Architect | — |
| Secret management | Yes | — | Architect |
| Agent API cost monitoring | Yes | Orchestrator | All |
| Infrastructure cost optimisation | Yes | — | Orchestrator |
| Incident response coordination | Yes | All | — |
| Harness delegate health | Yes | — | — |
| Observability stack (metrics/logs/traces) | Yes | Engineer | All |

---

## Day-to-Day Responsibilities

### Weekly Rhythm

**Monday — Pipeline health check:**
- Verify all Harness delegates are healthy and connected
- Review pipeline failure rates from the previous sprint
- Check Harness template library for drift from canonical templates
- Confirm GKE node pool health and Autopilot budget utilisation

**Daily — Deployment queue:**
- Review pending deployment approvals in Harness (HITL gate)
- Execute or reject deployments with written rationale in the Harness approval step
- Monitor GKE workload health post-deployment (rolling update progress, pod restart counts)
- Check VMware VCF scheduler queue for any blocked on-prem deployments

**Wednesday — Agent cost review:**
- Pull agent run cost report from the cost attribution dashboard
- Flag any agent run exceeding the per-task cost threshold
- Review context window sizes for bloat (a growing context window is often a sign of poor CLAUDE.md curation)
- Check Harness build minute consumption against monthly budget

**Friday — Observability review:**
- Review alert fatigue: disable or tune any alert that fired more than 10 times without action
- Confirm all new services deployed this sprint have Micrometer metrics, health endpoints, and structured logging
- Review any distributed tracing anomalies (GCP Cloud Trace for GKE, on-prem equivalent for VCF)

---

## Harness-Specific Responsibilities

### Pipeline Management

The Ops role owns the Harness pipeline definitions in `ci/pipelines/`. All pipeline YAML is version-controlled and reviewed before promotion.

**Pipeline governance rules:**
- Production pipeline changes require a PR reviewed by Ops + Orchestrator (HITL)
- Staging pipeline changes require Ops review (HOTL)
- Dev pipeline changes may be self-approved by Engineer (HOOTL)
- No inline secrets in pipeline YAML — all secrets via Harness Secret Manager references (`<+secrets.getValue("secret-name")>`)
- All pipelines must have a timeout configured; no open-ended pipeline runs

**Standard pipeline stages (all services):**

```yaml
stages:
  - stage:
      name: Build and Unit Test
      type: CI
      spec:
        execution:
          steps:
            - step:
                type: Run
                name: Maven Build
                spec:
                  command: mvn -B clean verify -Dsurefire.failIfNoSpecifiedTests=false
            - step:
                type: Run
                name: JaCoCo Coverage Gate
                spec:
                  command: mvn -B jacoco:check
  - stage:
      name: Architecture Lint
      type: CI
      spec:
        execution:
          steps:
            - step:
                type: Run
                name: ADR Compliance Check
                spec:
                  command: ./ci/scripts/adr-lint.sh
            - step:
                type: Run
                name: OpenAPI Contract Lint
                spec:
                  command: ./ci/scripts/openapi-lint.sh
  - stage:
      name: Integration Test
      type: CI
      spec:
        execution:
          steps:
            - step:
                type: Run
                name: Testcontainers Integration Tests
                spec:
                  command: mvn -B verify -P integration-tests
  - stage:
      name: OWASP Dependency Check
      type: CI
      spec:
        execution:
          steps:
            - step:
                type: Run
                name: Dependency Vulnerability Scan
                spec:
                  command: mvn -B org.owasp:dependency-check-maven:check -DfailBuildOnCVSS=7
  - stage:
      name: Build Docker Image
      type: CI
  - stage:
      name: Deploy to Staging
      type: CD
      spec:
        deploymentType: Kubernetes
  - stage:
      name: Staging Smoke Test
      type: CI
  - stage:
      name: Production Approval
      type: Approval
      spec:
        execution:
          steps:
            - step:
                type: HarnessApproval
                name: Ops Production Gate
                spec:
                  approvers:
                    userGroups:
                      - ops-team
                  approvalMessage: "Review staging results before approving production deployment"
  - stage:
      name: Deploy to Production
      type: CD
```

### Delegate Health

Harness Delegates run inside each target environment (GKE cluster, VCF environment). The Ops role monitors:
- Delegate connectivity status in Harness Manager UI
- Delegate memory and CPU utilisation (alert if >80% sustained for 10 minutes)
- Delegate version alignment with Harness SaaS (auto-upgrade enabled; Ops monitors for failed upgrades)
- Delegate log anomalies (connection refused, authentication failures)

**Delegate placement:**
- GKE: one Delegate deployment per cluster, Kubernetes mode, namespace `harness-delegate`
- VCF: one Delegate per environment (dev/staging/prod), Docker mode on a dedicated VM

### Secret Management

All secrets are stored in Harness Secret Manager (backed by GCP Secret Manager for cloud, CyberArk for on-prem). The Ops role:
- Provisions new secrets following the secret naming convention: `<env>/<product>/<secret-name>`
- Rotates secrets on the defined rotation schedule (see `framework/governance/hitl-model.md`)
- Audits secret access monthly from GCP Secret Manager audit logs
- Never permits secrets in pipeline YAML, Kubernetes manifests, or Helm values files

**Secret rotation HITL gate:** Any secret rotation in production requires Ops approval. The rotation process:
1. New secret version created in Secret Manager
2. Harness pipeline variable updated to reference new version
3. Rolling restart of affected deployments (blue/green preferred for zero downtime)
4. Old secret version invalidated after 24-hour verification window

---

## GCP GKE Responsibilities

### Namespace Governance

Each product gets a dedicated namespace per environment following the convention `<product>-<env>` (e.g. `order-service-prod`, `order-service-staging`).

**Namespace resource quotas (standard; adjust per product based on observed usage):**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: default-quota
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
```

**Network policies:** Each namespace has a default-deny ingress policy; explicit ingress policies per service. Ops provisions and reviews network policies; agents cannot modify them.

### Workload Identity

GKE Workload Identity is configured for all service accounts. The Ops role:
- Creates GCP service accounts with minimum required IAM roles
- Binds GKE Kubernetes service accounts to GCP service accounts using Workload Identity annotation
- Reviews IAM bindings quarterly and removes unused permissions
- IAM changes are a HITL gate — no agent or automated process modifies IAM bindings

### Autopilot vs Standard Node Management

**Autopilot (preferred for new workloads):**
- GKE manages node pools; Ops monitors pod scheduling success and cost
- Alert on pod pending >5 minutes (Autopilot provisioning failure)
- Review Autopilot cost reports weekly; compare to Standard equivalent for cost optimisation

**Standard (required for data pipelines and GPU workloads):**
- Node pool sizes defined in Terraform (version-controlled in `ci/infra/`)
- Node auto-scaling configured: min/max defined per environment
- Ops reviews node pool utilisation weekly; right-sizes pools monthly
- Node OS patching: GKE manages; Ops monitors for failed node upgrades

### Health Check Standard

All services deployed to GKE must expose (enforced by Helm chart template):
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 5
  failureThreshold: 3
```

---

## VMware VCF Responsibilities

### VM and Container Scheduling

VMware VCF supports both traditional VM workloads (monoliths) and Kubernetes-based container workloads (via Tanzu Kubernetes Grid, TKG). The Ops role:

**For VM workloads (monoliths, data pipeline schedulers):**
- VM templates maintained in vSphere Content Library; Ops manages template versions
- VM sizing based on profiling data; over-provisioning tracked as a cost metric
- VM snapshots before any major deployment (HITL gate for production VMs)
- Automated startup order groups for dependent services (e.g. database before application)

**For TKG container workloads:**
- TKG cluster lifecycle managed by Ops (upgrades, scale operations)
- Namespace and resource quota governance mirrors GKE model above
- NSX-T network policies for micro-segmentation; Ops provisions, agents cannot modify

### Network Policies (NSX-T)

```
Application Tier → Database Tier: TCP 5432 (PostgreSQL), TCP 6379 (Redis)
External LB → Application Tier: TCP 443 (HTTPS)
Application Tier → Message Bus: TCP 9092 (Kafka)
Management → All: SSH (22) from jump host CIDR only
```

Ops reviews NSX-T security group rules monthly. Any new inter-service communication requires explicit NSX-T rule approval (HITL gate).

### Storage Classes

VCF storage classes available (defined by Ops, referenced in app Helm charts):
- `vcf-ssd-retain` — SSD-backed, retain policy (databases, stateful services)
- `vcf-hdd-retain` — HDD-backed, retain policy (data pipeline output, archives)
- `vcf-ssd-delete` — SSD-backed, delete policy (ephemeral cache, scratch space)

Ops governs which storage class a workload may use; declared in `products/<product>/CLAUDE.md` infrastructure section.

---

## Agent Observability

This is the new dimension of the Ops role that does not exist in traditional DevOps. The Ops role instruments and monitors agent pipelines.

### Tracing Agent Runs

Each agent run is tagged with a structured trace ID that flows through:
- The agent invocation log (Claude API request/response)
- The Harness pipeline stage that triggered the agent
- The Git commit or PR created by the agent
- The audit log entry (see `framework/governance/audit-log-spec.md`)

**Trace ID format:** `{product_id}-{agent_id}-{phase}-{YYYYMMDD}-{UUID4_short}`
Example: `order-svc-impl-agent-implementation-20260615-a3f9`

### Debugging Multi-Step Agent Failures

When an agent run fails mid-way (partial output, incomplete PR, or corrupted context), the Ops role:

1. **Retrieve the agent run log** from GCP Cloud Logging using the trace ID
2. **Identify the failure step** — which tool call or reasoning step failed
3. **Check context completeness** — was the CLAUDE.md token limit exceeded? Were ADRs injected?
4. **Check rate limits** — was the Claude API returning 429s? Check cost guardrails thresholds
5. **Replay or escalate** — if recoverable, restart the agent with corrected context; if not, escalate to Engineer with full context

### Cost Per Agent Run

The Ops role monitors agent cost using the cost attribution tags applied by each agent run:

```json
{
  "product_id": "order-service",
  "agent_id": "implementation-agent",
  "phase": "implementation",
  "task_id": "TASK-042",
  "sprint": "2026-S23",
  "input_tokens": 12450,
  "output_tokens": 3200,
  "cost_usd": 0.089
}
```

**Cost alert thresholds (see `framework/governance/cost-guardrails.md` for full details):**
- Per agent run: >$0.50 triggers a HOTL alert to Ops
- Per task: >$2.00 triggers a HITL review
- Per sprint: >$200 across all agent runs triggers Orchestrator review

### Observability Stack

**GCP GKE:**
- Metrics: Micrometer → Cloud Monitoring; dashboards in Cloud Monitoring
- Logs: Logback JSON → Cloud Logging; log-based alerts configured
- Traces: Spring Boot Actuator + Cloud Trace; sampling rate 10% production, 100% staging
- Alerts: Cloud Monitoring alerting policies → PagerDuty for P1, email for P2/P3

**VMware VCF:**
- Metrics: Micrometer → Prometheus → Grafana (deployed on VCF management cluster)
- Logs: Logback JSON → Fluentd → Splunk (or syslog for simpler setups)
- Traces: Zipkin (on-prem) or forward to GCP Cloud Trace via VPN
- Alerts: Alertmanager → PagerDuty for P1, email for P2/P3

---

## HITL Gates Owned by Ops

| Gate | Trigger | What Must Happen | SLA |
|---|---|---|---|
| **Production Deployment** | Any deployment to production environment | Ops reviews staging test results, approves in Harness | 2h from staging pass |
| **Schema Migration (Production)** | Flyway migration in production pipeline | Ops co-approves with Architect; verifies backup exists | Before migration runs |
| **Secret Rotation** | Scheduled rotation or incident-triggered | Ops executes rotation process; verifies service continuity | Per rotation schedule |
| **IAM Change** | Any change to GCP IAM bindings or NSX-T security groups | Ops reviews and approves; logs in audit system | 4h from request |
| **Delegate Configuration Change** | Any change to Harness Delegate deployment | Ops reviews; tests in staging delegate first | 24h |
| **Node Pool Resize (Production)** | Scaling GKE node pools or TKG cluster in production | Ops approves after capacity analysis | 2h from request |
| **DR Test Execution** | Quarterly disaster recovery test | Ops coordinates; requires Orchestrator awareness | Scheduled |

---

## Incident Escalation Paths

### Severity Levels

| Severity | Definition | Response Time | Escalation |
|---|---|---|---|
| **P1 — Critical** | Production down, data loss, security breach | 15 minutes | PagerDuty → Ops → Orchestrator → All hands |
| **P2 — High** | Production degraded, >10% error rate, deployment blocked | 1 hour | PagerDuty → Ops; brief Orchestrator |
| **P3 — Medium** | Staging issue, non-critical feature broken, agent cost spike | 4 hours | Email → Ops; self-assigned |
| **P4 — Low** | Cosmetic, documentation, minor improvement | Next sprint | Backlog |

### Incident Response Steps

**P1 Response:**
1. Ops acknowledges PagerDuty within 15 minutes
2. Ops assesses: is this a deployment issue (roll back in Harness), infrastructure issue (GKE/VCF), or application issue (Orchestrator + Engineer engaged)?
3. Communication in incident Slack channel (or equivalent): status every 15 minutes
4. If agent caused the incident: immediately pause all agent runs for the affected product (HOOTL → HITL escalation)
5. Post-incident review within 24 hours; blameless retrospective; ADR or spec update to prevent recurrence

**When Agents Cause Incidents:**
The Ops role has authority to immediately pause all autonomous agent activity for a product by:
- Removing the product's Harness pipeline trigger webhooks
- Revoking the agent's repository write token
- Posting a P1 escalation to the full pod

Resuming agent activity after an agent-caused incident requires Orchestrator + Architect sign-off.

---

*See also: [`framework/governance/hitl-model.md`](../governance/hitl-model.md) | [`framework/governance/cost-guardrails.md`](../governance/cost-guardrails.md) | [`framework/governance/audit-log-spec.md`](../governance/audit-log-spec.md)*
