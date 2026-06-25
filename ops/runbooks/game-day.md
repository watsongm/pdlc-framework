# Game Day Runbook

## Purpose
Game days are quarterly exercises in which the team practises intervening in agent workflows, catching agent errors, and exercising human judgment — skills that atrophy when agents run well for extended periods. The goal is to ensure the team never rubber-stamps agent output and never loses the ability to override and correct.

> "If you never have to intervene, you won't know how to when it matters."

---

## Schedule

| Quarter | Focus Theme | Date (week of) |
|---|---|---|
| Q1 | Spec and ADR violations — catching architectural drift | First week of Q2 (end of Q1 lookback) |
| Q2 | Incident response — human vs agent triage accuracy | First week of Q3 |
| Q3 | Cost guardrails — cost runaway and context quality | First week of Q4 |
| Q4 | Full-system resilience — combined scenario | First week of Q1 next year |

**Duration:** Half-day (4 hours)  
**Attendees:** Full pod (all 3–5 members)  
**Facilitator:** Orchestrator (or rotate)  
**Observer:** Optional — invite a stakeholder or another pod to observe

---

## Preparation (1 week before)

- [ ] Select 2–3 scenarios from the library below (match to focus theme)
- [ ] Set up a **sandbox environment** — use staging, not production
- [ ] Confirm all team members have their escalation credentials ready (Harness access, GCP Console, etc.)
- [ ] Brief the team on the focus theme — no scenario spoilers
- [ ] Disable any automated alerts that would interfere with the exercise (routing to sandbox, not production Slack)
- [ ] Prepare the debrief template

---

## Scenario Library

### Scenario 1: ADR Violation — Catch and Reject

**Setup:** The facilitator manually modifies a Dev Agent PR to introduce an ADR violation (e.g., adds `@Autowired` field injection, violating JAVA-001). The team must catch it before merge.

**What the team practises:**
- Reading agent-generated code critically, not just approving it
- Using the ADR compliance report in the PR
- Rejecting a PR with clear, specific feedback

**Success criteria:**
- ADR violation identified within 15 minutes of PR being opened
- Violation correctly referenced by ADR ID and rule
- PR rejected with specific feedback that the agent could act on

**Facilitator notes:** Start with a subtle violation (not obviously wrong). Add more obvious ones if the team doesn't catch the subtle one.

---

### Scenario 2: Conflicting Requirements — Human Resolution

**Setup:** The facilitator edits `spec.md` to introduce a contradiction (e.g., one section says p99 latency < 100ms, another requires 3 synchronous downstream calls to services with 50ms average latency). The Architect Agent escalates with an escalation message. The team must resolve it.

**What the team practises:**
- Reading escalation messages and understanding the exact conflict
- Making a real architectural decision under time pressure
- Communicating the decision clearly so the agent can resume

**Success criteria:**
- Team reads the escalation message thoroughly (not skimmed)
- A concrete decision is made (not "let's discuss later")
- Decision is communicated in the correct format for the agent to resume
- Agent successfully resumes with the updated requirement

---

### Scenario 3: Wrong Incident Triage — Human Override

**Setup:** The facilitator injects a false signal into the sandbox environment — either a fake alert or a misattributed log entry — that causes the Incident Agent to diagnose the wrong root cause (e.g., blames the database when the actual issue is a recently deployed code change). The team must identify the incorrect diagnosis and override.

**What the team practises:**
- Critically reviewing agent incident reports rather than accepting them
- Using the deployment timeline to identify deployment-correlated issues
- Overriding the agent's recommendation with the correct diagnosis

**Success criteria:**
- Team identifies the incorrect root cause within 20 minutes
- The correct root cause is documented with supporting evidence
- Remediation is selected based on the correct diagnosis (not the agent's recommendation)

---

### Scenario 4: Agent Cost Runaway — Apply Guardrails

**Setup:** The facilitator simulates a cost spike by showing the team a "cost report" indicating one product's agent spend has tripled week-on-week (prepared mock data — no real cost is incurred). The team must identify the cause using the audit log and apply the correct guardrail.

**What the team practises:**
- Reading the cost per feature metrics
- Querying the audit log to find the high-cost agent runs
- Identifying the anti-pattern causing cost (e.g., unbounded context window, retry loop)
- Applying the correct guardrail (context size limit, max retries, CLAUDE.md update)

**Success criteria:**
- Cost spike root cause identified from audit log within 20 minutes
- Specific anti-pattern named (from the cost guardrails anti-pattern list)
- Concrete mitigation implemented (CLAUDE.md updated, prompt refined, limit set)

---

### Scenario 5: Stale Context — Refresh CLAUDE.md

**Setup:** The facilitator manually outdates the CLAUDE.md for a product (changes it to reflect an old architecture — e.g., references a database that was replaced 3 months ago). The team then runs an agent against this product and observes the degraded output quality. They must identify the staleness and refresh the context.

**What the team practises:**
- Recognising when agent output quality is degraded due to context staleness
- Updating CLAUDE.md systematically (not just editing random lines)
- Verifying that refreshed context produces better agent output

**Success criteria:**
- Agent output degradation attributed to stale context (not "the agent is bad")
- CLAUDE.md updated correctly and completely
- Agent re-run produces noticeably higher quality output
- Team documents a trigger for this product's next context review

---

### Scenario 6 (Advanced): Cascading Agent Failure

**Setup:** The facilitator introduces a corrupted ADR (invalid YAML frontmatter), causing the ADR Agent to fail when generating tasks. This blocks the Dev Agent, which escalates. Meanwhile, the Monitor Agent is also producing false positives from a metric threshold that was set too low. The team must manage multiple simultaneous agent issues.

**What the team practises:**
- Triaging multiple agent escalations simultaneously
- Correctly prioritising (ADR fix unblocks build; monitor false positive is lower urgency)
- Maintaining clear documentation of what they did and why

**Success criteria:**
- Both issues identified and correctly prioritised within 15 minutes
- ADR YAML fixed correctly; build pipeline unblocked
- Monitor threshold adjusted with reasoning documented
- Audit log entry created for each human override

---

## Debrief (30 minutes after scenarios)

Run immediately after the scenarios. No post-processing — the team's immediate reaction is the most valuable data.

### Debrief Template

```markdown
# Game Day Debrief — {QUARTER} {YEAR}

**Date:** {date}
**Attendees:** {names}
**Facilitator:** {name}
**Scenarios Run:** {scenario numbers and names}

---

## Scenario Results

### Scenario {N}: {Name}
- **Success criteria met:** Yes / Partial / No
- **Time to resolution:** {minutes}
- **What the team did well:** {notes}
- **What could be improved:** {notes}

---

## Overall Observations

### Intervention Speed
- Average time to identify agent error: {minutes}
- Target: < 20 minutes for any scenario

### Confidence Levels (team self-assessment, 1-5)
- Reading and acting on escalation messages: {score}
- Making architectural decisions under pressure: {score}
- Overriding agent recommendations: {score}
- Understanding audit log queries: {score}

### Rubber Stamp Risk
- Were any approvals given without full understanding? Yes / No
- If yes, what caused it? {notes}

---

## Action Items from This Game Day

| Action | Owner | Due Date |
|---|---|---|
| {action} | {name} | {date} |

---

## Framework Improvements Identified

{Any gaps in agent definitions, escalation paths, runbooks, or CLAUDE.md templates revealed by the scenarios}

---

## Next Game Day Focus

**Theme for next quarter:** {theme}
**Scenarios to consider:** {list}
```

---

## Metrics to Track per Game Day

| Metric | How to Measure | Target |
|---|---|---|
| Intervention speed | Time from scenario start to correct identification | < 20 min per scenario |
| Correct override rate | % of team overrides that were correct (not the agent) | ≥ 80% |
| Team confidence score | Average self-assessed score (1-5) across all dimensions | ≥ 3.5 / 5.0 |
| Action items per game day | Count of framework improvements identified | ≥ 2 per game day |
| Action item completion rate | % of prior game day actions completed by current game day | ≥ 90% |

---

*Owner: Orchestrator (scheduling + facilitation) | Frequency: Quarterly | Duration: Half-day*
