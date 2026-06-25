# HITL / HOTL / HOOTL Oversight Model

> **Framework:** AI-Native PDLC | **Version:** 1.0.0 | **Owner:** Orchestrator + Architect | **Review cadence:** Quarterly

---

## Overview

This document defines the three-tier oversight model used across all AI-native pod activities. Every action taken by an agent — and every action triggered by an agent — must be assigned to one of these tiers. Tier assignments are documented in agent definitions, enforced in Harness pipelines, and reviewed quarterly.

The model is based on a simple principle: **the more irreversible, high-impact, or context-sensitive an action is, the more human involvement is required.**

---

## Tier Definitions

### HITL — Human In The Loop

> **Definition:** A human must actively review and approve before the action executes. The system blocks until approval is received. No agent autonomy at this tier.

**When to use HITL:**
- The action is irreversible or difficult to reverse (production deployments, schema migrations, secret deletion)
- The action has security implications (IAM changes, credential rotation, network policy changes)
- The action affects business-critical data (data deletions, mass updates, external API calls with side effects)
- The action contradicts or overrides an existing ADR
- A human's accountability is legally or contractually required

**Implementation in Harness:**
HITL is implemented via Harness Approval steps in pipeline YAML:

```yaml
- step:
    type: HarnessApproval
    name: Production Deployment Gate
    spec:
      approvalMessage: |
        ## Production Deployment Review

        **Product:** <+pipeline.variables.product_name>
        **Version:** <+pipeline.variables.image_tag>
        **Environment:** production

        ### Pre-approval Checklist
        - [ ] Staging tests passed (see attached report)
        - [ ] No open P1/P2 incidents in production
        - [ ] Rollback plan documented and tested
        - [ ] On-call engineer notified

        **Approving this step authorises deployment to production.**
      approvers:
        userGroups:
          - ops-approvers
      minCount: 1
      includePipelineExecutorInApprovals: false
    timeout: 8h
    failureStrategies:
      - onFailure:
          errors:
            - ApprovalRejection
          action:
            type: Abort
```

**Human response required within:** Defined per gate type (see gate table below). If approval is not received within the timeout, the pipeline aborts and notifies the approver group.

---

### HOTL — Human On The Loop

> **Definition:** The action executes autonomously, but a human is notified and can intervene. The human monitors the execution and has a defined window to stop or roll back. If no intervention occurs, the action completes.

**When to use HOTL:**
- The action is reversible (staging deployments, configuration changes, test runs)
- Agent output will be human-reviewed before it reaches customers (code generation, PR creation)
- The action is well-understood with predictable failure modes
- The output quality matters but an imperfect result is recoverable

**Implementation in Harness:**
HOTL is implemented via notification steps with a monitoring window:

```yaml
- step:
    type: Run
    name: Notify HOTL Monitor
    spec:
      command: |
        curl -X POST ${SLACK_WEBHOOK} \
          -H 'Content-type: application/json' \
          --data '{
            "text": "HOTL Action Started: Staging deployment for *<+pipeline.variables.product_name>*\nMonitor window: 30 minutes\nTo abort: /harness abort <+pipeline.executionId>"
          }'
- step:
    type: ShellScript
    name: Deploy to Staging
    # ... deployment steps
    # Human can abort pipeline within monitoring window
```

**Monitoring window:** Defined per action type. Typically 30 minutes for staging deployments; longer for data migrations.

---

### HOOTL — Human Out Of The Loop

> **Definition:** The action executes fully autonomously. Humans are not notified in real time. All actions are logged in the audit log. Humans review the audit log retrospectively (daily or weekly, not per-action).

**When to use HOOTL:**
- The action is fully reversible with no customer impact (log analysis, metric collection)
- The action produces drafts that are always reviewed before use (documentation generation, PR descriptions)
- The action has well-established quality gates (automated tests, linters)
- The risk of human notification fatigue exceeds the risk of autonomous action

**Implementation in Harness:**
HOOTL actions run in pipeline stages with no approval gates. They write to the audit log:

```yaml
- step:
    type: Run
    name: Generate PR Description
    spec:
      command: |
        # Agent generates PR description
        ./agents/run.sh doc-agent --task pr-description --pr-id ${PR_ID}

        # Log to audit system
        ./ci/scripts/audit-log.sh \
          --action "pr-description-generated" \
          --tier "HOOTL" \
          --agent "doc-agent" \
          --product "${PRODUCT_ID}"
```

---

## Decision Matrix

The following table defines the canonical tier assignment for each action type. When in doubt, escalate one tier higher (more human involvement). Tier assignments may be changed by proposing an ADR and getting Architect + Orchestrator approval.

| Action Type | Default Tier | Rationale | Approver |
|---|---|---|---|
| **Production deployment** | HITL | Irreversible; customer impact | Ops |
| **Schema migration (additive)** | HITL | Irreversible data structure change | Ops + Architect |
| **Schema migration (destructive)** | HITL — elevated | Data loss risk | Ops + Architect + Orchestrator |
| **IAM / permission change** | HITL | Security surface; hard to audit post-facto | Ops |
| **Secret rotation** | HITL | Service continuity risk | Ops |
| **Architecture change (ADR override)** | HITL | Violates agreed contract | Architect + Orchestrator |
| **Data deletion (bulk)** | HITL | Irreversible | Ops + Orchestrator |
| **External API call with financial side effects** | HITL | Business liability | Orchestrator |
| **New third-party dependency introduction** | HITL | Security + architecture | Architect |
| **Network policy change (NSX-T / GKE)** | HITL | Security surface | Ops |
| **Staging deployment** | HOTL | Reversible; no customer impact | Ops monitors |
| **Non-breaking dependency update** | HOTL | Automated tests gate quality | Engineer monitors |
| **Configuration change (non-secret)** | HOTL | Reversible; audited | Ops monitors |
| **Feature flag change** | HOTL | Customer-facing but reversible in <5 min | Engineer monitors |
| **Integration test suite run** | HOTL | Engineer reviews results | Engineer monitors |
| **Agent-generated code PR** | HOTL | Engineer review gate before merge | Engineer reviews |
| **API contract update (non-breaking)** | HOTL | Contract tests gate quality | Architect monitors |
| **Log analysis** | HOOTL | Read-only; fully reversible | Audit log |
| **Metric collection and reporting** | HOOTL | Read-only | Audit log |
| **PR description generation** | HOOTL | Always human-reviewed before use | Audit log |
| **Test scaffolding generation** | HOOTL | Engineer reviews tests before merge | Audit log |
| **Documentation updates (non-ADR)** | HOOTL | Low risk; versioned | Audit log |
| **Commit message generation** | HOOTL | Human reviews before push | Audit log |
| **Discovery scan (read-only)** | HOOTL | Read-only; no writes to production | Audit log |
| **Audit log summary generation** | HOOTL | Humans review summaries | Audit log |

---

## Confidence-Based Tier Escalation

Agents are required to assess their own confidence before executing an action. If confidence falls below defined thresholds, the agent must escalate to the next tier — even if the default tier would normally permit autonomous action.

### Confidence Score Definition

Each agent produces a confidence score (0.0–1.0) based on:
- Context completeness: are all required inputs present?
- Ambiguity detection: does the task have a clear, unambiguous interpretation?
- Precedent: has the agent done this type of task successfully before?
- Scope: is the change within expected bounds (file count, line count, service count)?

### Escalation Thresholds

| Confidence Score | Action |
|---|---|
| 0.9 – 1.0 | Proceed at default tier |
| 0.7 – 0.89 | Proceed at default tier; add confidence note to PR description |
| 0.5 – 0.69 | Escalate one tier (HOOTL → HOTL, HOTL → HITL); notify monitor |
| 0.3 – 0.49 | Escalate to HITL regardless of default tier; include uncertainty summary |
| 0.0 – 0.29 | Do not proceed; generate escalation message and stop |

### Confidence Note Format in PR Description

```markdown
## Agent Confidence Assessment

**Overall Confidence:** 0.72 (HOTL — monitor recommended)

**Factors reducing confidence:**
- Ambiguous acceptance criterion: "Should support bulk operations" — interpreted as batch endpoint; verify this matches intent
- No existing precedent in this codebase for pagination pattern; used RFC 9457 approach per ADR-API-002

**Recommended human review focus:**
- Line 45: Pagination implementation — confirm page size default (used 20)
- Line 112: Bulk operation limit — hardcoded 100 items max; adjust if different limit required
```

---

## The Rubber Stamp Anti-Pattern

**Definition:** A HITL gate that is never rejected. If every approval is granted in <2 minutes without questions, the gate has become a rubber stamp — it provides the illusion of oversight without the reality.

### How to Detect Rubber Stamping

- Average approval time <2 minutes for a HITL gate that should require 15+ minutes of review
- Zero rejections or requested changes over a rolling 4-week period
- Approvers cannot articulate what they checked when asked retrospectively

### How to Prevent Rubber Stamping

**Structural controls:**
- Approval forms require the approver to confirm specific checklist items (e.g. "I have reviewed the staging test report and it passes acceptance criteria")
- Minimum approval time enforced: Harness approval steps can be configured with a minimum time before approval is accepted (use custom approval webhook to enforce this)
- Rotating approvers: approval authority rotates across team members quarterly so no single person becomes the default rubber-stamper

**Cultural controls:**
- Quarterly gate reviews: the team explicitly asks "is this HITL gate adding value or is it a rubber stamp? If rubber stamp, should we downgrade to HOTL or fix the checklist?"
- Rejection quota: each HITL gate should show at least one rejection or change request per month. If not, treat as a signal to investigate.
- Blameless post-mortems when rubber-stamped changes cause incidents — focus on process, not person

---

## Audit Log Requirements by Tier

| Tier | What Must Be Logged | When Logged | Retention |
|---|---|---|---|
| **HITL** | Full action details, approver identity, approval timestamp, approval time elapsed, rejection rationale (if rejected), decision outcome | At approval/rejection | 7 years |
| **HOTL** | Action details, monitoring window start/end, any interventions made, final outcome | At completion | 2 years |
| **HOOTL** | Action type, agent ID, input/output summary, cost tokens/USD, outcome | At completion | 1 year |

See [`framework/governance/audit-log-spec.md`](audit-log-spec.md) for the full log schema.

---

## Quarterly Tier Assignment Review

Every quarter, the pod reviews all tier assignments to determine if they are still appropriate.

### Review Agenda (60 minutes)

1. **Incident review (15 min):** Were there any incidents caused by HOOTL or HOTL actions that should have been HITL? Reclassify upward as needed.

2. **Rubber stamp review (10 min):** Review HITL gate approval times and rejection rates. Reclassify any rubber-stamped gates to HOTL with enhanced monitoring.

3. **Efficiency review (10 min):** Are any HITL gates creating bottlenecks without corresponding risk justification? Consider downgrading to HOTL with condition-based HITL escalation.

4. **New action types (10 min):** Assign tiers to any new action types introduced in the quarter (new agent capabilities, new infrastructure targets).

5. **Confidence threshold calibration (15 min):** Review agent escalation log. Are the confidence thresholds causing too many false escalations (too low) or missing real issues (too high)? Adjust.

### Output of Quarterly Review

- Updated decision matrix (this document, version bumped)
- ADR for any tier reclassifications
- Updated Harness pipeline configurations for any changed gates

---

*See also: [`framework/governance/escalation-paths.md`](escalation-paths.md) | [`framework/governance/audit-log-spec.md`](audit-log-spec.md) | [`framework/governance/cost-guardrails.md`](cost-guardrails.md)*
