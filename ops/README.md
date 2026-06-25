# Ops Phase

## Overview

The Operations phase is the ongoing lifecycle of a product once it has been through onboarding, spec, and initial build. It covers continuous deployment, monitoring, incident response, dependency maintenance, and the change management process for evolving the product over time.

The ops phase never ends — it runs for the lifetime of the product.

---

## The Three-Tier Model Applied to Operations

| Operation Type | Autonomy | Examples |
|---|---|---|
| **HITL** | Agent halts; human must approve | Production deployments, schema migrations, secret rotation, P1 incident remediation, emergency rollbacks |
| **HOTL** | Agent runs; human monitors | Staging deployments, dependency update PRs, post-deploy health monitoring, P2 incident triage, config changes |
| **HOOTL** | Autonomous; async audit | Log analysis, metric collection, dependency vulnerability scanning, PR description generation, daily health reports |

---

## The Operational Agent Team

| Agent | Role | Autonomy |
|---|---|---|
| **Deploy Agent** | Triggers Harness pipelines; monitors deployments; auto-rollback on failure | HOTL (staging) / HITL (production) |
| **Monitor Agent** | Continuous metrics/log monitoring; cost tracking; daily reports; incident triggering | HOOTL (monitoring) / HOTL (alerts) / HITL (incidents) |
| **Incident Agent** | First-responder triage; evidence gathering; RCA drafting; remediation proposal | HOTL (triage) / HITL (remediation) |

See individual agent definitions in [framework/agents/ops/](../../framework/agents/ops/).

---

## Change Management Flow

All changes to a product — regardless of size — flow through a consistent process. See [change-management.md](change-management.md) for full detail.

```
Change Request
    ↓
Is this a spec-altering change?
  YES → Mini spec pipeline (proposal delta → spec update → ADR update if needed → tasks delta)
  NO  → Create task directly, referencing existing spec section
    ↓
Build phase (Dev Agent → Test Agent → Review Agent → PR)
    ↓
Harness CI: test → ADR lint → Docker → staging deploy
    ↓
Human review (Engineer — HOTL gate)
    ↓
HITL approval gate (Orchestrator or Architect for spec changes; Engineer for bugfixes)
    ↓
Production deployment (Deploy Agent → HITL approval → production)
    ↓
Post-deploy monitoring (Monitor Agent — enhanced for 30 min)
```

---

## Harness as the Operational Backbone

All deployments and many operational actions flow through Harness:

- **Pipeline templates** — reusable stage templates for Java/Spring Boot services in `ci/templates/`
- **Approval gates** — all HITL gates are Harness approval steps assigned to named user groups
- **Rollback stages** — every production pipeline has an automatic rollback stage triggered by health check failure
- **Delegates** — one delegate per environment (GKE cluster, VCF environment); delegates must be healthy before any deployment
- **Secret management** — all credentials stored in Harness Secrets Manager; never in environment variables or files

---

## GCP GKE Operational Patterns

| Concern | Approach |
|---|---|
| **Scaling** | Horizontal Pod Autoscaler (HPA) targeting 70% CPU; Autopilot handles node provisioning |
| **Workload Identity** | Each service has a dedicated K8s ServiceAccount bound to a GCP service account; no shared accounts |
| **Resource quotas** | Namespace-level quotas enforce per-service resource limits; prevent noisy-neighbour problems |
| **Monitoring** | Micrometer → Cloud Monitoring; Cloud Logging with structured JSON; Cloud Trace for distributed tracing |
| **Alerting** | Cloud Monitoring alert policies → Pub/Sub → Harness webhook → Monitor Agent |
| **Certificate management** | cert-manager with Google Certificate Authority; auto-renewal |

---

## VMware VCF Operational Patterns

| Concern | Approach |
|---|---|
| **VM lifecycle** | Managed by Ansible playbooks; vSphere tags identify all PDLC-managed VMs |
| **Network policy** | NSX-T micro-segmentation; each service in its own security group; zero-trust between segments |
| **Scaling** | Manual VM provisioning via Ansible (no auto-scaling); capacity planning owned by Ops |
| **Monitoring** | JVM metrics via Micrometer → Prometheus (on-prem) → Grafana; logs via rsyslog → Splunk |
| **Alerting** | vROps alerts → email/webhook → Ops team; Splunk alerts → Monitor Agent |
| **Backup** | VM snapshots via vSAN before each production deployment; retention: 7 days |

---

## Cost Monitoring

The Monitor Agent produces monthly cost reports covering:
- Claude API token costs per product (attributed to tasks via tagging)
- Harness build minutes per product
- GCP GKE compute costs per namespace (via GCP billing export to BigQuery)
- VMware VCF internal chargeback (manual entry from vROps)

See [framework/governance/cost-guardrails.md](../../framework/governance/cost-guardrails.md) for limits and alerts.

---

## Runbooks

| Runbook | When to Use |
|---|---|
| [incident-response.md](runbooks/incident-response.md) | Production incident triggered (P1/P2/P3/P4) |
| [dependency-update.md](runbooks/dependency-update.md) | Quarterly dependency audit or CVE alert |
| [game-day.md](runbooks/game-day.md) | Quarterly practice of HITL intervention |

---

*Phase Owner: Ops/Platform role | Duration: Ongoing for product lifetime*
