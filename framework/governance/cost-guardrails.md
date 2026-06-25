# Cost Guardrails

This document defines the cost control framework for all agent-generated compute, API usage, and infrastructure resources within the AI-Native PDLC Framework. Uncontrolled costs are one of the most common failure modes in agentic systems.

---

## Cost Categories

| Category | Source | Billing Model | Owner |
|---|---|---|---|
| **Claude API tokens** | Anthropic API | Per input/output token | Ops/Platform |
| **Harness build minutes** | Harness SaaS | Per pipeline minute | Ops/Platform |
| **GCP GKE compute** | GCP Billing | Per vCPU-hour + memory-hour | Ops/Platform |
| **GCP supporting services** | GCP Billing | Per request / per GB | Ops/Platform |
| **VMware VCF resources** | Internal chargeback | Per vCPU + vRAM allocation | Ops/Platform |
| **MCP server hosting** | Variable | Depends on self-hosted vs managed | Ops/Platform |

---

## Per-Task Cost Attribution

Every agent run must be tagged so cost can be attributed to the product and task that generated it.

### Tagging Standard

**Claude API calls** — include in the system prompt or API metadata:
```json
{
  "metadata": {
    "product_id": "my-service",
    "task_id": "TASK-042",
    "agent_id": "dev-agent",
    "phase": "build",
    "environment": "development"
  }
}
```

**Harness pipeline runs** — use pipeline variables:
```yaml
variables:
  - name: productId
    type: String
    value: <+input>
  - name: taskId
    type: String
    value: <+input>
  - name: agentId
    type: String
    value: dev-agent
```

**GCP GKE resources** — use pod labels:
```yaml
metadata:
  labels:
    pdlc.io/product-id: my-service
    pdlc.io/agent-id: deploy-agent
    pdlc.io/environment: production
```

**VMware VCF VMs** — use vSphere tags:
- `pdlc:product-id={product_id}`
- `pdlc:agent-managed=true`

---

## Cost Limits

### Hard Limits (Automated enforcement — runs are terminated)

| Limit | Value | Enforcement |
|---|---|---|
| Single agent run — Claude API tokens | 500,000 tokens (input + output) | API middleware / proxy limit |
| Single Harness pipeline run | 60 minutes | Harness pipeline timeout |
| Daily Claude API spend per product | £50 | Anthropic usage limit or billing alert + auto-disable |
| Monthly Claude API spend (all products) | £500 | Billing alert → Ops alert → manual review |
| GKE pod CPU (per service) | 2 vCPU (limit in Helm values) | Kubernetes resource limits |
| GKE pod memory (per service) | 1Gi (limit in Helm values) | Kubernetes resource limits |

### Soft Limits (Alert triggered — human notified, run continues)

| Limit | Value | Notification |
|---|---|---|
| Single agent run — Claude API tokens | 200,000 tokens | Slack alert to Ops |
| Single agent run cost | £5 | Slack alert to Ops |
| Agent run count per product per day | 50 runs | Slack alert to Ops |
| Harness pipeline — stage duration | 20 minutes | Harness stage timeout warning |
| Weekly Claude API spend per product | £80 | Email to Orchestrator |

---

## Cost per Feature Tracking Formula

```
Cost per Feature = Σ(Agent API cost for all runs tagged to TASK-NNN)
                 + Σ(Harness build minutes for pipeline runs tagged to TASK-NNN × rate)
                 + Σ(GCP compute for staging deployment duration × rate)
```

**Establish baseline in first quarter.** Target 10% cost reduction per quarter thereafter through:
1. Improving context quality (reduces token waste from hallucinations and retries)
2. Reducing escalation frequency (each escalation burns tokens + human time)
3. Caching stable context (ADRs, runtime standards) where backend supports it
4. Right-sizing agent runs (decompose large tasks rather than single massive runs)

**Monthly cost report template** (produced by Monitor Agent, reviewed by Ops):
```markdown
## Agent Cost Report — {MONTH}

| Product | Tasks Completed | Claude API (tokens) | Claude API (£) | Harness (mins) | Harness (£) | Total (£) | Cost/Feature |
|---|---|---|---|---|---|---|---|
| my-service | 24 | 2.4M | £38 | 180 | £9 | £47 | £1.96 |

### Top 3 Most Expensive Tasks
1. TASK-031 — £8.20 (large refactor; 6 retries due to test failures)
2. TASK-019 — £6.10 (dependency mapping on 400k LOC monolith)
3. TASK-044 — £4.80 (contract test generation)

### Trend
Month-on-month: -8% (target: -10%)

### Recommended Actions
- TASK-031 high cost: add more granular ADR constraints to prevent retry loops
- Consider caching ADR context between same-day agent runs for my-service
```

---

## Alert Thresholds and Notifications

| Event | Threshold | Notified | Channel |
|---|---|---|---|
| Single run exceeds soft token limit | 200k tokens | Ops | Slack |
| Single run exceeds hard token limit | 500k tokens | Ops | Slack + email |
| Daily product spend exceeds £30 | Per product | Orchestrator | Slack |
| Daily product spend exceeds £50 (hard limit) | Per product | Ops + Orchestrator | Slack + pager |
| Monthly total spend exceeds £400 | Portfolio | Orchestrator + Ops | Email |
| Monthly total spend exceeds £500 (hard limit) | Portfolio | All pod members | Email + Slack |
| Any single task costs 3× rolling average | Per task | Ops | Slack |
| Agent run count >50/day for any product | Per product | Ops | Slack |

**Alert format (Slack):**
```
⚠️ PDLC Cost Alert
Product: {product_id}
Agent: {agent_id}
Task: {task_id}
Event: {event description}
Current Spend: £{amount}
Threshold: £{threshold}
Action Required: {none (soft) | review immediately (hard)}
```

---

## Anti-Patterns That Drive Cost

These patterns are the most common causes of unexpected agent cost escalation. Agents are instructed to avoid them; humans should look for them in code review.

| Anti-pattern | Description | Mitigation |
|---|---|---|
| **Unbounded context windows** | Loading entire codebases into context instead of targeted files | Use CLAUDE.md summaries; load only files referenced in the task |
| **Over-parallelism** | Spawning many subagents simultaneously for tasks that are actually sequential | Review task dependency graph before parallelising |
| **Redundant re-discovery** | Running the discovery agent every sprint instead of updating CLAUDE.md incrementally | Run full discovery quarterly; do targeted updates after significant changes |
| **Retry loops without escalation** | Agent retrying a failing test >3 times without escalating | Hard escalation trigger at 3 retries |
| **Context injection of full spec** | Loading all 50 pages of spec.md when only 2 sections are relevant | Use task's `Spec Reference` field to load only relevant sections |
| **Low-value automation** | Using agents for tasks faster done manually (e.g., renaming a variable) | Apply the 5-minute rule: if a human can do it in <5 minutes, don't use an agent |
| **Agent self-review without external validation** | Dev Agent reviewing its own code without a separate Review Agent run | Always run Review Agent as a separate step |

---

## Monthly Review Process

1. **Ops produces cost report** (Monitor Agent draft, Ops reviews) by the 3rd of each month for the previous month
2. **Orchestrator reviews** cost per feature trend and compares to targets
3. **Identify outlier tasks** — any task >3× rolling average is investigated
4. **Mitigation actions** are added to the next sprint's task backlog (e.g., "Improve CLAUDE.md for product-x to reduce retry rate")
5. **Horizon adjustment** — if costs are rising and throughput is not, this is a signal to review context quality before adding more agents

---

*Last updated: 2026-06 | Owner: Ops/Platform | Review cadence: Monthly cost review, quarterly limit review*
