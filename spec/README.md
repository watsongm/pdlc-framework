# Spec Phase — Overview

> **Phase Position:** Discovery → **Spec** → Implementation → Verification → Release  
> **Primary Audience:** Human Architects, Product Engineers, AI Agents operating in this phase  
> **Related Files:** [pipeline.md](./pipeline.md) · [templates/](./templates/) · [prompts/](./prompts/)

---

## Purpose

The Spec phase translates the raw outputs of Discovery (or the intent captured in a Greenfield proposal) into a **living, machine-readable specification** that serves three distinct audiences simultaneously:

| Audience | What they need from the spec |
|---|---|
| **Human Engineers** | Clear, reviewable requirements they can debate, refine and sign off |
| **AI Agents** | Stable, structured context they can load and reason against without ambiguity |
| **CI/CD Pipeline** | Parseable constraints (ADRs) that can be linted against every pull request |

Without a spec phase, agents hallucinate requirements, engineers debate intent in code reviews, and architectural drift accumulates silently. The spec is the **single source of truth** from which all downstream work — design, code, tests, infrastructure — must be traceable.

---

## The Spec Pipeline

Every piece of work in this framework, whether it starts from discovery of an existing product or from a blank-canvas greenfield idea, flows through the same pipeline:

```
proposal.md → spec.md → design.md → ADRs → tasks.md
```

Each artifact is a versioned file committed to the product repository. Each transition between artifacts has a defined responsible party and, at critical gates, a **Human-in-the-Loop (HITL)** approval step that must be completed before the pipeline continues.

See [pipeline.md](./pipeline.md) for the full step-by-step breakdown with Mermaid diagram, responsible parties, and gate definitions.

---

## Why Spec-First Matters

### 1. Prevents Code Drift

When agents generate code without a stable specification anchor, they make implicit decisions about scope, edge cases, and data contracts that may contradict each other across multiple agent invocations. A spec forces these decisions to be made explicitly and reviewed by a human before code is written.

### 2. Gives Agents a Stable Context Anchor

Every agent in this framework loads the spec, the design document, and the relevant ADRs as mandatory context before generating any output. This means that if the spec is clear and complete, agents produce consistent, on-target outputs. If the spec is ambiguous, the agent is required to surface that ambiguity to a human rather than guess.

### 3. Serves as an Audit Trail

The spec is committed to version control alongside the code it describes. When requirements change — and they always do — the spec is updated first, the delta is reviewed, and the change propagates to design, ADRs, and tasks in a controlled way. This creates a reviewable history of *why* the system is built the way it is.

### 4. Enables Autonomous Agent Operation at Scale

As agents operate at higher autonomy tiers (HOTL → HOOTL), the spec and ADRs become the primary guardrail system that prevents autonomous agents from making unilateral architectural decisions. An agent that cannot find an ADR covering a decision escalates to a human rather than proceeding.

---

## Human vs. Agent Responsibilities

| Pipeline Step | Primary Responsible Party | Agent Role | HITL Gate |
|---|---|---|---|
| **Proposal** | Human (Product/Engineer) | May assist with drafting and gap analysis | ✅ Human writes & approves |
| **Spec Generation** | Architect Agent | Generates from proposal + discovery inputs | ❌ Agent generates autonomously |
| **Spec Review** | Human Architect | Agent flags issues found during generation | ✅ Human must approve spec.md |
| **ADR Generation** | ADR Agent | Identifies decisions; generates one ADR per decision | ❌ Agent generates autonomously |
| **ADR Review** | Human Architect | — | ✅ Human must approve each ADR |
| **Design Generation** | Architect Agent | Generates from approved spec + ADRs | ❌ Agent generates autonomously |
| **Design Review** | Human Architect | — | ✅ Human must approve design.md |
| **Task Decomposition** | Architect Agent | Decomposes spec+design into ordered task backlog | ❌ Agent generates autonomously |
| **Task Review** | Human Engineer | Advisory — may reject or reorder tasks | ⚠️ Advisory (not blocking) |

**Key principle:** Humans make decisions; agents generate artifacts that express those decisions in structured form. Agents never commit a spec, ADR, or design without human review.

---

## ADR as Guardrail

Architecture Decision Records (ADRs) are the enforcement mechanism that bridges the spec phase and the implementation phase.

### How ADRs Work in This Framework

1. **Generation:** The ADR Agent reads the approved spec and design documents and identifies every architectural decision that requires a record — database choice, auth mechanism, API style, async/sync patterns, deployment topology, etc.

2. **Storage:** ADRs are committed to the product repository at `<product-root>/adrs/ADR-NNNN-<title>.md`. They are version-controlled alongside the code they govern.

3. **Context Injection:** Every agent that operates on this codebase loads all ADRs from the `/adrs/` directory as mandatory context before producing any output. Agents must explicitly reference relevant ADR IDs in their outputs.

4. **CI Enforcement:** The Harness pipeline runs an **architectural linting** step on every pull request. This step parses the `compliance_rule` field from every accepted ADR and validates the PR diff against those rules. PRs that violate an ADR are blocked from merging.

5. **Lifecycle Management:** ADRs have a status lifecycle: `proposed → accepted → deprecated → superseded`. When an architectural decision changes, the old ADR is marked `superseded` and a new ADR is created. The CI linter flags use of patterns from deprecated/superseded ADRs.

### Why In-Repo ADRs Beat External Wikis

- They travel with the code in the same pull request
- They are visible to agents scanning the repository
- They cannot get out of sync with the codebase (CI catches violations)
- They serve as living documentation that engineers and agents both read

---

## Living Documentation Principle

Specs are not write-once artefacts. As the product evolves, the spec must evolve with it. This framework enforces a **delta spec pattern**:

1. When a requirement changes, the engineer or agent creates a **delta proposal** describing only what is changing and why.
2. The delta proposal triggers a targeted spec update — only the affected sections of `spec.md` are modified.
3. The spec change triggers a downstream check: does the design need updating? Do any ADRs need revision? Do any tasks need to be replanned?
4. All changes are committed atomically and reviewed as a coherent unit.

The spec version number increments on every approved change. The `changelog` section at the bottom of `spec.md` records what changed, when, and why.

This approach prevents the two failure modes that plague documentation-heavy processes:
- **Stale specs** (written once, never updated) — prevented by the delta pattern and CI spec-drift detection
- **Spec churn** (constant re-generation from scratch) — prevented by scoping updates to deltas only

---

## Directory Structure

```
spec/
├── README.md                    ← This file
├── pipeline.md                  ← Full pipeline detail with Mermaid diagram
├── templates/
│   ├── proposal.md              ← Template: starting point for all work
│   ├── spec.md                  ← Template: functional + non-functional spec
│   ├── design.md                ← Template: technical design document
│   ├── adr.md                   ← Template: Architecture Decision Record
│   └── tasks.md                 ← Template: agent task backlog
└── prompts/
    ├── spec-from-proposal.md    ← Prompt: Architect Agent generates spec from proposal
    ├── adr-from-spec.md         ← Prompt: ADR Agent generates ADRs from spec
    └── tasks-from-spec.md       ← Prompt: Architect Agent decomposes spec into tasks
```

---

## Quick Start

### For a new Greenfield product:
1. Copy `templates/proposal.md` → `<product-root>/spec/proposal.md`
2. Fill in all sections; mark `entry_point: greenfield`
3. Get human approval on the proposal (`status: approved`)
4. Trigger the Architect Agent using `prompts/spec-from-proposal.md`
5. Review the generated `spec.md` — approve or iterate
6. Trigger the ADR Agent using `prompts/adr-from-spec.md`
7. Review and approve each generated ADR
8. Architect Agent generates `design.md` and `tasks.md`
9. Review tasks; implementation phase begins

### For a Discovery (existing product):
1. Complete the Discovery phase — produces `discovery-report.md` and `architecture-map.md`
2. Copy `templates/proposal.md` → `<product-root>/spec/proposal.md`
3. Fill in proposal; mark `entry_point: discovery`; summarise discovery findings in `Discovery Summary`
4. Follow the same steps 3–9 as Greenfield above

---

*Last updated: 2026-06-24 | Framework version: 1.0*
