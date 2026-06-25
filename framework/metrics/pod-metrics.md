# Pod Performance Metrics

This document defines the performance measurement framework for a hybrid human+agent pod. The guiding principle is **system-level outcomes over individual output metrics** — the team succeeds or fails as a unit, not as individuals or agents in isolation.

---

## Philosophy

> "What gets measured gets managed — but measuring the wrong things destroys what matters."

Traditional engineering metrics (lines of code, PRs merged, story points completed) are **counterproductive in an agent-augmented team**. They reward volume over value and create perverse incentives — agents can trivially inflate any count-based metric.

This framework measures **outcomes that humans care about**: reliable delivery, architectural integrity, operational stability, and sustainable cost.

---

## Metric Definitions

### 1. Spec Coverage

| Field | Value |
|---|---|
| **Description** | Percentage of implemented features that have a corresponding, approved spec section |
| **Why it matters** | Without spec coverage, agent output is unanchored — review is subjective and drift is invisible |
| **How to measure** | Count tasks in `tasks.md` with `Status: done` that have a non-empty `Spec Reference` field ÷ total done tasks × 100 |
| **Target** | ≥ 90% |
| **Review cadence** | Per sprint (after each task batch completes) |
| **Owner** | Orchestrator |
| **Anti-pattern** | Retroactively adding spec references after implementation |

---

### 2. ADR Compliance Rate

| Field | Value |
|---|---|
| **Description** | Percentage of merged PRs that passed the Harness ADR compliance lint check without a human override |
| **Why it matters** | ADR violations indicate architectural drift — agents are deviating from agreed decisions |
| **How to measure** | Harness pipeline: count PRs where "ADR Compliance Check" stage = PASSED (no override) ÷ total merged PRs × 100 |
| **Target** | ≥ 95% |
| **Review cadence** | Weekly |
| **Owner** | Architect |
| **Alert threshold** | < 90% in any week → immediate Architect review |

---

### 3. Defect Escape Rate

| Field | Value |
|---|---|
| **Description** | Percentage of defects found in production versus total defects found (production + staging + test) |
| **Why it matters** | Measures test and review quality; agent-generated code should have lower escape rates than manual code |
| **How to measure** | Count production incidents caused by code defects ÷ (production incidents + staging defects caught + test failures caught) × 100 |
| **Target** | < 5% |
| **Review cadence** | Monthly |
| **Owner** | Engineer |
| **Alert threshold** | Any P1 production incident caused by agent-generated code → immediate retrospective |

---

### 4. Agent Output Rework Ratio

| Field | Value |
|---|---|
| **Description** | Percentage of agent-generated code that required significant human rework before merge (defined as: PR required >3 review rounds or human rewrote >30% of the diff) |
| **Why it matters** | High rework indicates poor context quality, insufficient spec detail, or ADR gaps — not poor agent capability |
| **How to measure** | Engineers tag PRs with `rework: significant` when threshold met; count tagged PRs ÷ total agent PRs × 100 |
| **Target** | < 15% |
| **Review cadence** | Per sprint |
| **Owner** | Engineer |
| **Action when breached** | Review CLAUDE.md freshness, ADR completeness, and spec detail level — not agent prompts first |

---

### 5. Delivery Stability

| Field | Value |
|---|---|
| **Description** | Two sub-metrics: (a) Change Failure Rate — % of deployments causing a production incident; (b) Mean Time to Recovery (MTTR) — average time from incident detection to service restoration |
| **Why it matters** | Stability is the foundation. AI-accelerated delivery is worthless if it destabilises production. |
| **How to measure** | (a) Harness deployment data: failed deploys ÷ total deploys × 100. (b) Average time between `P1 incident created` and `P1 incident resolved` timestamps |
| **Target** | (a) < 5% | (b) < 30 min for GKE services, < 60 min for VCF services |
| **Review cadence** | Monthly (trend analysis); immediate review on any P1 |
| **Owner** | Ops/Platform |

---

### 6. Time to Onboard a New Product

| Field | Value |
|---|---|
| **Description** | Calendar days from "decision to onboard product" to "first approved spec in repo" |
| **Why it matters** | Measures the framework's ability to rapidly bring new products under governance |
| **How to measure** | Track start date (product directory created) and end date (spec.md status = approved) |
| **Target** | ≤ 5 working days (discovery path); ≤ 2 working days (greenfield path) |
| **Review cadence** | Per onboarding event |
| **Owner** | Orchestrator |

---

### 7. Agent Cost per Feature

| Field | Value |
|---|---|
| **Description** | Total agent API cost (Claude API tokens) + Harness build minutes attributed to implementing one feature (one spec section → merged PR) |
| **Why it matters** | Ensures AI leverage is economically sustainable; catches cost runaway early |
| **How to measure** | Tag each agent run with `product_id` + `task_id`; aggregate token costs from API billing + Harness build cost per pipeline run |
| **Target** | Track trend (no absolute target initially; establish baseline in first quarter, then target 10% reduction per quarter) |
| **Review cadence** | Monthly |
| **Owner** | Ops/Platform |
| **Alert threshold** | Any single task exceeding 3× the rolling average cost |

---

### 8. Context Quality Score

| Field | Value |
|---|---|
| **Description** | Qualitative assessment of how current and accurate the product's `CLAUDE.md`, ADRs, and spec are relative to the actual codebase |
| **Why it matters** | Context quality is the single biggest driver of agent output quality — more than model choice or prompt quality |
| **How to measure** | Quarterly review by Architect: score each product 1–5 across three dimensions: (a) CLAUDE.md currency (b) ADR coverage (c) Spec-code alignment |
| **Target** | Average score ≥ 4/5 across all three dimensions |
| **Review cadence** | Quarterly |
| **Owner** | Architect |

---

### 9. Sprint Throughput Trend

| Field | Value |
|---|---|
| **Description** | Number of features (merged PRs with a spec reference) delivered per sprint. Track trend, not absolute value. |
| **Why it matters** | Measures whether agent leverage is actually improving delivery velocity over time |
| **How to measure** | Count merged spec-linked PRs per sprint; plot trend |
| **Target** | Positive trend over rolling 3-month window. Specific target set after H1 baseline is established (typically 2–4× H2 velocity within 6 months) |
| **Review cadence** | Per sprint |
| **Owner** | Orchestrator |

---

## McKinsey Horizon Self-Assessment Rubric

Use this rubric at quarterly pod reviews to track maturity progression.

### H1 → H2 Transition Criteria (achieved when ALL are true):
- [ ] Spec Coverage ≥ 70% for the last full quarter
- [ ] At least one product successfully onboarded via discovery path
- [ ] ADR compliance check running in Harness CI for all products
- [ ] CLAUDE.md files maintained and dated within 30 days for all products
- [ ] Team can articulate what each agent does and when to intervene

### H2 → H3 Transition Criteria (achieved when ALL are true):
- [ ] Spec Coverage ≥ 90% sustained for 2 consecutive quarters
- [ ] ADR Compliance Rate ≥ 95% sustained for 1 quarter
- [ ] Defect Escape Rate < 5% sustained for 1 quarter
- [ ] Agent Cost per Feature tracked and trending downward
- [ ] Greenfield product spec-to-deployed within 10 working days
- [ ] Team has completed at least 2 game day exercises with documented learnings

### H3 → H4 Transition Criteria (achieved when ALL are true):
- [ ] All H2→H3 criteria sustained for 2 consecutive quarters
- [ ] HOOTL autonomy granted to at least 3 routine operation categories
- [ ] Context Quality Score ≥ 4.5/5 for all products
- [ ] Human rework ratio < 10%
- [ ] Agent cost per feature < baseline × 0.6
- [ ] Team has defined their own H4 success criteria (framework becomes product-specific at this level)

---

## Quarterly Pod Health Review — Agenda

Run this review every quarter. Duration: 90 minutes.

### Agenda

| Time | Topic | Owner |
|---|---|---|
| 0–10 min | Metric dashboard review (all 9 metrics, trend charts) | Orchestrator |
| 10–25 min | Horizon self-assessment — score each criterion, discuss gaps | Full pod |
| 25–40 min | Context quality audit — CLAUDE.md, ADR, spec review per product | Architect |
| 40–55 min | Agent incident retrospective — rework events, escalations, failures | Engineer |
| 55–70 min | Cost review — per-product agent cost trend; identify outliers | Ops |
| 70–80 min | Game day debrief (if game day occurred this quarter) | Full pod |
| 80–90 min | Actions for next quarter — metric targets, process improvements | Orchestrator |

### Outputs
- Updated metric baseline targets for next quarter
- CLAUDE.md refresh tasks for any product with Context Quality < 4
- At least one process improvement per retrospective item
- Updated Horizon position (H1/H2/H3/H4)

---

## Anti-Metrics (What NOT to Measure)

These metrics are explicitly excluded from pod assessment. Tracking them creates perverse incentives in an agent-augmented team.

| Anti-metric | Why it's harmful |
|---|---|
| Lines of code written | Agents write verbose code; rewards bloat |
| Number of PRs merged | Agents can trivially open small PRs; rewards fragmentation |
| Story points / velocity | Meaningless when agents complete tasks in minutes; creates pressure to under-estimate |
| Token count per agent | Rewards shallow context; agents should be given rich context |
| Agent run count | Rewards over-automation of tasks that need human judgment |
| Individual PR review throughput | Rewards rubber-stamping; humans should review carefully, not quickly |
| Time-to-first-response | Rewards agents proceeding when they should escalate |

---

## Metric Dashboard Template

Maintain a lightweight dashboard (a Markdown table updated weekly) in the team's shared space:

```markdown
## Pod Metrics Dashboard — Week of {DATE}

| Metric | This Week | Last Week | Target | Status |
|---|---|---|---|---|
| Spec Coverage | 92% | 89% | ≥90% | ✅ |
| ADR Compliance Rate | 97% | 95% | ≥95% | ✅ |
| Defect Escape Rate | 2% | 3% | <5% | ✅ |
| Rework Ratio | 18% | 22% | <15% | ⚠️ |
| Delivery Stability (CFR) | 3% | 4% | <5% | ✅ |
| Agent Cost / Feature | £12 | £14 | Trend ↓ | ✅ |
| Time to Onboard | 4 days | 6 days | ≤5 days | ✅ |
| Horizon Position | H2 | H2 | Target H3 | 🔄 |

### This Week's Actions
- [ ] Rework ratio breached — Architect to review CLAUDE.md for product-x
- [ ] Horizon H2→H3 gap: spec coverage target for next quarter = 92%
```

---

*Last updated: 2026-06 | Owner: Orchestrator | Review cadence: Quarterly*
