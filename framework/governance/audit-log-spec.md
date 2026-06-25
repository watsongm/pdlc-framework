# Audit Log Specification

This document defines the audit log schema, retention policy, and compliance requirements for all agent actions within the AI-Native PDLC Framework. Every agent action at every autonomy tier (HITL/HOTL/HOOTL) must produce an audit log entry.

---

## Purpose

The audit log serves four functions:
1. **Governance** — proves that human oversight is real, not symbolic (critical for regulated environments)
2. **Debugging** — traces exactly what an agent did and why when something goes wrong
3. **Improvement** — feeds back into prompt refinement, context quality reviews, and escalation calibration
4. **Compliance** — satisfies regulatory requirements for demonstrable oversight of automated systems

---

## Log Schema

Every audit log entry is a single JSON object. All fields are required unless marked optional.

```json
{
  "schema_version": "1.0",
  "log_id": "uuid-v4",
  "timestamp": "2026-06-24T20:30:00Z",
  "agent_id": "dev-agent",
  "product_id": "order-service",
  "phase": "build",
  "task_id": "TASK-042",
  "action_type": "code_generation",
  "autonomy_tier": "HOTL",
  "input_summary": {
    "files_loaded": [
      "products/order-service/CLAUDE.md",
      "products/order-service/spec/spec.md",
      "products/order-service/adrs/ADR-0003-rest-api.md"
    ],
    "task_description": "Implement POST /api/v1/orders endpoint per spec section 3.2",
    "context_token_count": 12400
  },
  "output_summary": {
    "files_created": ["src/main/java/com/example/orders/controller/OrderController.java"],
    "files_modified": ["src/main/java/com/example/orders/service/OrderService.java"],
    "lines_added": 147,
    "lines_removed": 12,
    "pr_url": "https://github.com/org/repo/pull/234"
  },
  "human_reviewer": "gwatson",
  "decision": "approved",
  "decision_timestamp": "2026-06-24T21:15:00Z",
  "escalation_triggered": false,
  "escalation_trigger": null,
  "escalation_urgency": null,
  "escalation_resolution": null,
  "cost_tokens_input": 12400,
  "cost_tokens_output": 3800,
  "cost_usd": 0.18,
  "duration_seconds": 142,
  "adr_compliance_check": "passed",
  "adr_violations": [],
  "test_result": "passed",
  "notes": "Constructor injection confirmed; Micrometer counter added to OrderService"
}
```

### Field Definitions

| Field | Type | Required | Description |
|---|---|---|---|
| `schema_version` | string | Yes | Audit log schema version; increment on breaking changes |
| `log_id` | UUID v4 | Yes | Globally unique identifier for this log entry |
| `timestamp` | ISO 8601 | Yes | When the agent action started |
| `agent_id` | string | Yes | Agent identifier from agent contract |
| `product_id` | string | Yes | Product this action relates to |
| `phase` | enum | Yes | `onboarding`, `spec`, `build`, `ops` |
| `task_id` | string | Yes | Task identifier from tasks.md; `SYSTEM` for automated monitoring |
| `action_type` | string | Yes | See action type taxonomy below |
| `autonomy_tier` | enum | Yes | `HITL`, `HOTL`, `HOOTL` |
| `input_summary.files_loaded` | array | Yes | All context files loaded |
| `input_summary.task_description` | string | Yes | Brief description of what was requested |
| `input_summary.context_token_count` | number | Yes | Total input tokens |
| `output_summary.files_created` | array | Yes | Files created by this action |
| `output_summary.files_modified` | array | Yes | Files modified by this action |
| `output_summary.lines_added` | number | Yes | Lines of code added |
| `output_summary.lines_removed` | number | Yes | Lines of code removed |
| `output_summary.pr_url` | string | No | Pull request URL if applicable |
| `human_reviewer` | string | No | Username of human who reviewed (HITL/HOTL) |
| `decision` | enum | No | `approved`, `rejected`, `modified`, `pending` |
| `decision_timestamp` | ISO 8601 | No | When human made their decision |
| `escalation_triggered` | boolean | Yes | Whether an escalation was triggered |
| `escalation_trigger` | string | No | The specific trigger condition |
| `escalation_urgency` | enum | No | `P1`, `P2`, `P3` |
| `escalation_resolution` | string | No | How the escalation was resolved |
| `cost_tokens_input` | number | Yes | Input tokens consumed |
| `cost_tokens_output` | number | Yes | Output tokens produced |
| `cost_usd` | number | Yes | Estimated USD cost of this run |
| `duration_seconds` | number | Yes | Wall-clock time for the agent run |
| `adr_compliance_check` | enum | Yes | `passed`, `failed`, `skipped` |
| `adr_violations` | array | Yes | List of ADR IDs violated (empty if none) |
| `test_result` | enum | No | `passed`, `failed`, `not_run` |
| `notes` | string | No | Human or agent notes on notable aspects of this run |

---

## Action Type Taxonomy

Use these standardised values for the `action_type` field:

| Action Type | Description | Typical Autonomy Tier |
|---|---|---|
| `codebase_discovery` | Discovery Agent scanning a repository | HOTL |
| `documentation_ingestion` | Doc Agent reading external documentation | HOTL |
| `dependency_mapping` | Dependency Agent building architecture map | HOTL |
| `spec_generation` | Architect Agent generating spec.md | HOTL |
| `adr_generation` | ADR Agent generating Architecture Decision Records | HOTL |
| `design_generation` | Architect Agent generating design.md | HOTL |
| `task_decomposition` | Architect Agent decomposing spec into tasks | HOTL |
| `code_generation` | Dev Agent implementing a task | HOTL |
| `test_generation` | Test Agent generating test classes | HOTL |
| `code_review` | Review Agent checking spec/ADR compliance | HOOTL |
| `deployment_staging` | Deploy Agent deploying to staging | HOTL |
| `deployment_production` | Deploy Agent deploying to production | HITL |
| `incident_triage` | Incident Agent investigating an alert | HOTL |
| `dependency_audit` | Monitor Agent scanning for outdated/vulnerable deps | HOOTL |
| `context_refresh` | Any agent updating CLAUDE.md | HOTL |
| `monitoring` | Monitor Agent collecting metrics/logs | HOOTL |
| `adr_compliance_check` | CI pipeline ADR linting step | HOOTL |

---

## Detail Level by Autonomy Tier

| Autonomy Tier | Required Detail Level |
|---|---|
| **HITL** | Full schema; all optional fields must be populated; human decision and timestamp mandatory |
| **HOTL** | Full schema; `human_reviewer` optional if human reviewed asynchronously; decision required |
| **HOOTL** | Abbreviated schema; `input_summary`, `output_summary`, `cost_*`, `adr_compliance_check`, `escalation_triggered` required; human fields optional |

---

## Retention Policy

| Environment | Retention Period | Storage |
|---|---|---|
| Production (GKE) | 2 years | GCP Cloud Logging (bucket-backed, long-term retention) |
| Production (VCF) | 2 years | Splunk (index: `pdlc_audit`) |
| Staging | 90 days | GCP Cloud Logging / Splunk |
| Development | 30 days | Local file or short-retention log sink |

**Legal hold:** If a product is subject to a regulatory audit or legal hold, the Orchestrator must mark those logs for indefinite retention in GCP or Splunk before the normal retention window expires.

---

## Integration Points

### GCP Cloud Logging (GKE Workloads)

Agents running against GKE-deployed products write logs to Cloud Logging:

```python
# Python example — writing structured audit log to Cloud Logging
from google.cloud import logging as gcp_logging

client = gcp_logging.Client()
logger = client.logger("pdlc-agent-audit")

logger.log_struct(audit_log_entry, severity="INFO",
                  labels={"product_id": product_id, "agent_id": agent_id})
```

**Cloud Logging filter to view all HITL decisions:**
```
logName="projects/{project}/logs/pdlc-agent-audit"
jsonPayload.autonomy_tier="HITL"
```

**Cloud Logging filter to view all escalations:**
```
logName="projects/{project}/logs/pdlc-agent-audit"
jsonPayload.escalation_triggered=true
```

### On-Premise Syslog / Splunk (VCF Workloads)

Agents running against VCF-deployed products write to syslog, forwarded to Splunk:

```bash
# Syslog format for Splunk ingestion
logger -t pdlc-audit -p local0.info "$(echo $AUDIT_LOG_JSON | jq -c .)"
```

**Splunk search for HITL decisions:**
```spl
index=pdlc_audit autonomy_tier=HITL | table timestamp, agent_id, product_id, task_id, human_reviewer, decision
```

**Splunk search for unresolved escalations:**
```spl
index=pdlc_audit escalation_triggered=true decision=pending | table timestamp, agent_id, product_id, escalation_urgency
```

---

## Use in Governance Reviews

The quarterly pod health review uses audit log data directly:

| Review Item | Audit Log Query |
|---|---|
| Escalation frequency trend | Count `escalation_triggered=true` per week, per product |
| Human review response time | `decision_timestamp - timestamp` for all HITL entries |
| ADR compliance trend | Count `adr_compliance_check=failed` per week |
| Cost trend | Sum `cost_usd` grouped by `product_id`, `week` |
| Rework frequency | Count entries where `output_summary.lines_removed > 50` on HOTL code_review actions |
| Rubber stamp detection | HITL entries where `decision_timestamp - timestamp < 60 seconds` — flag for review |

---

## Compliance Notes

### GDPR
- Audit logs must not contain PII (user data, personal information from the product's domain)
- The `input_summary` and `output_summary` fields contain only file paths and line counts — not file contents
- If an agent's output contains PII (e.g., generated test fixtures with real-looking data), the Ops role must ensure it is not captured in the audit log `notes` field
- Data residency: audit logs for VCF-hosted products remain on-premise in Splunk; GKE logs stay in the GCP region of the cluster

### Regulatory Demonstrability
When regulators ask for evidence of human oversight, the audit log provides:
- Complete list of all production deployments and who approved them (`action_type=deployment_production`, `decision=approved`, `human_reviewer`)
- Complete list of all security-related findings and how they were resolved (`escalation_trigger` contains "security")
- ADR compliance check history for all merged code (`adr_compliance_check` field)
- Escalation and resolution timeline for all incidents

The audit log is **not** sufficient alone — regulators will also want to see the HITL process documentation, escalation paths, and evidence that human reviewers understood what they were approving. Refer to [hitl-model.md](hitl-model.md).

---

*Last updated: 2026-06 | Owner: Ops/Platform + Orchestrator | Review cadence: Annually for schema; immediately on compliance requirement changes*
