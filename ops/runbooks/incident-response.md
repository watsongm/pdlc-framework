# Incident Response Runbook

## Purpose
This runbook guides the full incident lifecycle for services managed under the AI-Native PDLC Framework — from alert detection through triage, escalation, remediation, and post-mortem. The Incident Agent automates triage; humans make all remediation decisions.

---

## Severity Levels

| Severity | Definition | Response SLA (GKE) | Response SLA (VCF) | Escalation |
|---|---|---|---|---|
| **P1 — Critical** | Service completely down; all users affected; data loss risk | 15 min to human | 30 min to human | Immediate page |
| **P2 — High** | Major feature broken; majority of users affected; significant degradation | 1 hour | 2 hours | Slack + email |
| **P3 — Medium** | Minor feature broken; subset of users affected; degraded performance | 4 hours | 8 hours (next business day if overnight) | Slack channel |
| **P4 — Low** | Non-functional issue; no user impact; operational concern | Next business day | Next business day | Ticket |

**MTTR Targets:**
- P1: < 30 min (GKE) / < 60 min (VCF)
- P2: < 2 hours
- P3: < 1 business day
- P4: < 1 sprint

---

## Agent-Assisted Response Flow

```
Alert fires (Cloud Monitoring / vROps / User report)
    ↓
Monitor Agent creates incident-trigger.md
    ↓
Incident Agent activated automatically
    ↓
Incident Agent executes triage protocol (< 5 minutes)
    ↓
Incident report produced:
  - Severity assessment
  - Probable cause
  - Blast radius
  - Remediation options
    ↓
Is severity P1?
  YES → Immediate human page; agent halts and waits
  NO  → Human notified via Slack/email; agent continues monitoring
    ↓
Human reviews incident report (HITL for all remediation)
    ↓
Human selects remediation option
    ↓
Agent assists implementation (PR for code fix, Harness rollback trigger)
  All actions require HITL approval
    ↓
Service restored
    ↓
Incident Agent drafts post-mortem
    ↓
Architect reviews and publishes within 48 hours (P1/P2) or 5 days (P3)
```

---

## GCP-Specific Alert Chain

```
Cloud Monitoring Alert Policy
    → Pub/Sub topic: pdlc-alerts
    → Cloud Functions (or Harness webhook)
    → Harness pipeline trigger: incident-triage-pipeline
    → Incident Agent activated via Claude Code
```

**Cloud Monitoring alert policy (example):**
```json
{
  "displayName": "Order Service — High Error Rate",
  "conditions": [{
    "displayName": "HTTP 5xx error rate > 5%",
    "conditionThreshold": {
      "filter": "metric.type=\"custom.googleapis.com/spring_boot/http_requests_total\" AND metric.labels.status=\"5xx\" AND resource.labels.namespace_name=\"order-service-prod\"",
      "comparison": "COMPARISON_GT",
      "thresholdValue": 0.05,
      "duration": "60s",
      "aggregations": [{"alignmentPeriod": "60s", "perSeriesAligner": "ALIGN_RATE"}]
    }
  }],
  "notificationChannels": ["projects/{project}/notificationChannels/pdlc-pubsub"]
}
```

---

## VMware VCF-Specific Alert Chain

```
vRealize Operations (vROps) alert
    → Email notification to Ops distribution list
    → Ops manually creates incident-trigger.md
    → Incident Agent activated
```

**vROps alert configuration:**
- Alert criticality ≥ "Immediate" → page Ops
- Alert criticality "Warning" → email Ops
- All PDLC-managed VMs tagged `pdlc-managed=true` are in scope

---

## Rollback Procedure

### GKE — Harness Rollback
```bash
# Via Harness API — trigger rollback stage
POST https://app.harness.io/pipeline/api/pipeline/execute/{rollback_pipeline_id}
# Harness rollback pipeline reverts to the previous successful deployment artifact
# Rollback pipeline: deploy-previous-artifact → health-check → notify
```

```bash
# Direct Helm rollback (emergency only — bypasses Harness audit trail)
# Only use if Harness is unavailable
helm rollback {release-name} -n {namespace}
kubectl rollout status deployment/{service-name} -n {namespace}
```

### VMware VCF — VM Snapshot Restore
```bash
# Via Ansible playbook
ansible-playbook rollback.yml \
  -e "product_id={product_id}" \
  -e "target_artifact={previous_version}" \
  -i inventory/{environment}.ini
```

---

## Communication Templates

### P1 — Initial Stakeholder Notification (send within 15 minutes)
```
Subject: [P1 INCIDENT] {Product Name} — {Brief Description}

Status: Investigating
Affected Service: {product_name} ({environment})
Impact: {user-facing description of impact}
Started: {timestamp}

We are actively investigating. Next update in 15 minutes.

Incident Commander: {name}
```

### P1 — Update (every 15 minutes)
```
Subject: [P1 UPDATE] {Product Name} — {timestamp}

Status: {Investigating | Mitigating | Monitoring}
Current Understanding: {1-2 sentences on root cause hypothesis}
Actions Taken: {list}
Next Steps: {list}
Estimated Resolution: {time or "under investigation"}

Next update at: {time}
```

### All Severities — Resolution
```
Subject: [RESOLVED] {Product Name} — {Brief Description}

Status: RESOLVED
Resolved At: {timestamp}
Duration: {duration}

Root Cause: {1-2 sentences}
Fix Applied: {brief description}
Post-mortem: {link} — due {date}
```

---

## Post-Mortem Template

Write to `products/<name>/ops/post-mortems/PM-{INCIDENT-ID}.md`:

```markdown
# Post-Mortem — {INCIDENT-ID}

**Date:** {incident date}
**Product:** {product_name}
**Severity:** P{1|2|3}
**Duration:** {start} to {end} ({total minutes})
**Authors:** {names}
**Status:** Draft | Final

---

## Summary
{2-3 sentences: what happened, impact, how it was resolved}

## Timeline
| Time | Event | Actor |
|---|---|---|
| {time} | {event} | Human/Agent |

## Root Cause
{Detailed explanation of the root cause}

## Contributing Factors
{Other factors that made the incident worse or harder to detect}

## 5 Whys
1. Why did the service fail? → {answer}
2. Why did {answer}? → {answer}
3. Why did {answer}? → {answer}
4. Why did {answer}? → {answer}
5. Why did {answer}? → {root cause}

## Impact
- Users affected: {number or estimate}
- SLA breach: Yes/No
- Data loss: Yes/No
- Revenue impact: {estimate or N/A}

## What Went Well
{Honest assessment of what the team/agents did right}

## What Could Be Improved
{Honest assessment of gaps in process, tooling, or documentation}

## Action Items
| Action | Owner | Due Date | Status |
|---|---|---|---|
| {action} | {name} | {date} | Open |

## Framework Gaps Identified
{Were there gaps in the spec, ADRs, CLAUDE.md, or monitoring config that contributed to this incident?}
{These become tasks in the next sprint}
```

---

*Owner: Ops/Platform | Applies to: All PDLC-managed products on GKE and VMware VCF*
