# AI-Native PDLC Framework

> **A scaffold for teams of 3–5 to design, build, and continuously operate software products using managed AI agent teams — across the full Product Development Lifecycle.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Status: Active Development](https://img.shields.io/badge/Status-Active%20Development-orange.svg)]()
[![Runtime: Java 21 + Spring Boot 3.x](https://img.shields.io/badge/Runtime-Java%2021%20%2B%20Spring%20Boot%203.x-green.svg)]()
[![Agents: Claude Code](https://img.shields.io/badge/Agents-Claude%20Code-purple.svg)]()
[![CI/CD: Harness](https://img.shields.io/badge/CI%2FCD-Harness-blueviolet.svg)]()
[![Cloud: GCP GKE + VMware VCF](https://img.shields.io/badge/Cloud-GCP%20GKE%20%2B%20VMware%20VCF-4285F4.svg)]()

---

## The Problem

Most organisations adopting AI for software development sit at **McKinsey Horizon 2** — layering AI co-pilots onto existing processes. They see modest 20–30% productivity gains but don't fundamentally change how products are built or how teams are structured.

The step-change in value comes at **Horizon 3 and 4**: redesigning the entire Product Development Lifecycle (PDLC) around AI agents, where a small team of humans acts as orchestrators, architects, and reviewers — not code writers.

This framework is the **scaffold** that makes that transition practical, repeatable, and safe.

```
H1  ──────  No AI             1x
H2  ──────  AI co-pilots      ~1.2x
H3  ──────  PDLC redesigned   ~2x        ← This framework targets H3→H4
H4  ──────  Agents + humans   5–20x
```

---

## Core Design Ethos

### 1. Spec-First, Always
Specifications are the **primary, durable source of truth**. Code is a verified secondary artifact.

This prevents "code drift" — where AI-generated output gradually diverges from original intent over time. Every piece of work starts with a proposal, flows through a structured spec pipeline, and anchors all agent work to an approved design.

```
proposal.md → spec.md → design.md → ADRs → tasks.md → code
```

### 2. Agents as Team Members, Not Tools
Each agent in this framework has a **defined role, bounded scope, explicit inputs/outputs, and clear escalation paths**. They are not monolithic "do everything" prompts. They are structured team members with responsibilities — just like humans — and they know when to stop and ask.

### 3. Human-in-the-Loop by Architecture
The framework embeds a **three-tier oversight model** at every workflow stage:

| Tier | Mode | When |
|---|---|---|
| **HITL** | Agent pauses; human must approve | Production deployments, schema changes, ADR decisions, security changes |
| **HOTL** | Agent runs; human monitors and can intervene | Code generation, test runs, staging deployments, dependency updates |
| **HOOTL** | Fully autonomous; async audit log | Log analysis, metric collection, PR descriptions, test generation |

This is designed to **prevent both "AI theatre"** (rubber-stamping without understanding) **and over-caution** (human approval for every minor agent action).

### 4. ADRs as Living Guardrails
Architecture Decision Records are stored **in-repo, version-controlled, and injected into every agent's context**. They act as persistent memory for the codebase and as machine-readable constraints enforced by CI — preventing agents from re-litigating resolved decisions or drifting from agreed patterns.

### 5. Loose Coupling on Agent Infrastructure
The framework is **optimised for Claude Code** (Anthropic's terminal-native agentic coding system) but is designed so the underlying agent infrastructure can be swapped. All agent definitions conform to an [Agent Interface Contract](framework/standards/agent-interface.md) that maps cleanly to LangGraph, Google ADK, CrewAI, or Microsoft Agent Framework.

### 6. Observability First
Every agent action is traceable. If you cannot trace why an agent failed, you cannot fix it. The framework mandates structured audit logs at every tier, integrating with GCP Cloud Monitoring (for GKE workloads) and standard syslog/Splunk (for VMware VCF workloads).

---

## Framework Overview

```
PDLC/
├── README.md                          ← You are here
├── framework/
│   ├── roles/                         ← Human team role definitions (3–5 person pod)
│   ├── agents/                        ← Agent team definitions (role, prompts, escalations)
│   │   ├── onboarding/                ← Discovery Agent · Doc Agent · Dependency Agent
│   │   ├── spec/                      ← Architect Agent · ADR Agent
│   │   ├── build/                     ← Dev Agent · Test Agent · Review Agent
│   │   └── ops/                       ← Deploy Agent · Monitor Agent · Incident Agent
│   ├── governance/                    ← HITL model, escalation paths, cost controls, audit log spec
│   ├── standards/                     ← Runtime standards (Java/Spring Boot) + Agent Interface Contract
│   └── metrics/                       ← Hybrid pod performance metrics
├── onboarding/                        ← Phase 1: Understand the product
│   ├── runbook.md                     ← Step-by-step (discovery path + greenfield path)
│   ├── templates/                     ← Discovery report, architecture map, glossary, interface inventory
│   └── prompts/                       ← Agent prompt templates
├── spec/                              ← Phase 2: Define the target architecture
│   ├── pipeline.md                    ← Full spec pipeline + Mermaid diagram
│   ├── templates/                     ← proposal · spec · design · adr · tasks templates
│   └── prompts/                       ← Agent prompt templates
├── build/                             ← Phase 3: Build toward the spec
│   ├── workflow.md                    ← Build loop + Harness CI pipeline definition
│   └── prompts/                       ← Agent prompt templates
├── ops/                               ← Phase 4: Operate and evolve
│   ├── change-management.md           ← Change types, flows, approval gates
│   ├── runbooks/                      ← Incident response · Dependency updates · Game days
│   └── prompts/                       ← Agent prompt templates
└── products/
    └── _template/                     ← Copy this for each new product
        ├── CLAUDE.md                  ← Primary agent context file (human-maintained)
        ├── discovery/                 ← Phase 1 outputs (agent-generated)
        ├── spec/                      ← Phase 2 outputs (living spec)
        ├── adrs/                      ← Architecture Decision Records
        ├── tasks/                     ← Current task backlog
        └── ops/                       ← Operational runbooks per product
```

---

## The Four Phases

### Phase 1 — Onboarding: Understand the Product

An agent team autonomously analyses an existing product — reading its codebase, documentation, and infrastructure — and produces a structured **product knowledge base** used as context for all future agents.

**Two entry points:**
- **Discovery Path** — Existing product → agent team produces architecture map, dependency graph, interface inventory, domain glossary
- **Greenfield Path** — No existing product → human writes `proposal.md` directly → skip to Phase 2

**Agent Team:** Discovery Agent · Doc Agent · Dependency Agent  
**Time target:** < 5 working days

---

### Phase 2 — Spec: Define the Target Architecture

Human intent and discovery outputs flow through a structured pipeline to produce a living, machine-readable specification that all future agents treat as their contract.

```
proposal.md
    ↓ (Architect Agent generates)
spec.md ─────────── [HITL: Architect reviews]
    ↓ (ADR Agent generates)
design.md + ADRs ── [HITL: Architect approves]
    ↓ (Architect Agent decomposes)
tasks.md ─────────── [Advisory: Engineer validates]
```

**Agent Team:** Architect Agent · ADR Agent

---

### Phase 3 — Build: Implement the Spec

The build agent team works through the task backlog, with humans providing oversight at defined quality gates. Every PR is checked against spec and ADRs before reaching a human reviewer.

```
Pick task from tasks.md
    → Dev Agent implements (HOTL)
    → Test Agent generates & runs tests (HOTL)
    → Review Agent checks spec/ADR compliance (HOOTL → escalates violations)
    → PR created with compliance report
    → Harness CI: build · ADR lint · Docker · staging deploy
    → Human Engineer reviews PR (HOTL gate)
    → HITL approval → production deploy
```

**Agent Team:** Dev Agent · Test Agent · Review Agent

---

### Phase 4 — Operate: Continuous Evolution

Agents handle routine operations; humans make judgment calls on complex or high-stakes decisions. All changes flow through the same spec pipeline, ensuring the spec remains the source of truth throughout the product's life.

**Agent Team:** Deploy Agent · Monitor Agent · Incident Agent

---

## Human Team Model

| Role | Count | Focus |
|---|---|---|
| **Orchestrator** (Tech/Product Lead) | 1 | Product intent → spec translation; governance; stakeholder alignment |
| **Architect** | 1 | ADR ownership; spec validation; architectural integrity |
| **Engineer** | 1–2 | PR review; agent oversight; context engineering; prompt refinement |
| **Ops/Platform** | 0–1 | Harness CI/CD; GKE/VCF deployments; observability; cost monitoring |

> The key skill shift: from **writing code** to **specifying intent, engineering context, and verifying agent output**.

---

## Technology Stack

| Concern | Choice | Notes |
|---|---|---|
| **Primary Runtime** | Java 21 + Spring Boot 3.x | Extensible — see [runtime-standards.md](framework/standards/runtime-standards.md) |
| **Build Tool** | Maven (primary) / Gradle | Spring Boot BOM managed |
| **Agent Infrastructure** | Claude Code | Loosely coupled — see [agent-interface.md](framework/standards/agent-interface.md) |
| **CI/CD** | Harness | Pipeline YAML, Delegates, Template Library, Approval Gates |
| **Container Platform (cloud)** | GCP GKE (Autopilot) | Workload Identity, Cloud Monitoring, HPA |
| **On-Premise** | VMware VCF | vSphere + NSX-T; VM and container workloads |
| **API Contracts** | OpenAPI 3.1 | springdoc-openapi; machine-readable for agent context |
| **Testing** | JUnit 5 · Mockito · Testcontainers · Spring Boot Test | Contract tests: Spring Cloud Contract |
| **Observability** | Micrometer + GCP Cloud Monitoring | Structured JSON logs, Cloud Trace |
| **ADR Management** | Markdown in-repo | `/products/<name>/adrs/` — injected into all agent contexts |

---

## Supported Service Types

| Type | Deployment Target | Entry Points |
|---|---|---|
| **REST API** | GCP GKE | Discovery + Greenfield |
| **Microservice** | GCP GKE | Discovery + Greenfield |
| **Monolith** | VMware VCF | Discovery-focused |
| **Data Pipeline** | VMware VCF or hybrid | Discovery + Greenfield |

---

## Pod Performance Metrics

Moving away from individual output metrics toward system-level outcomes:

| Metric | Target |
|---|---|
| Spec Coverage (% features with linked spec) | > 90% |
| ADR Compliance Rate (PRs passing CI lint) | > 95% |
| Defect Escape Rate | < 5% |
| Agent Output Rework Ratio | < 15% |
| Time to Onboard New Product | < 5 working days |
| Delivery Stability (change failure rate) | Trend ↓ |
| Agent Cost per Feature | Trend ↓ |

---

## Getting Started

### Prerequisites
- Claude Code (or alternative — see [agent-interface.md](framework/standards/agent-interface.md))
- Harness account with delegate configured for your target environments
- GCP project with GKE cluster, and/or VMware VCF environment
- Git repository for the candidate product
- Team of 3–5 with roles assigned per [framework/roles/](framework/roles/)

### Onboarding an Existing Product

```bash
# 1. Clone the framework
git clone https://github.com/watsongm/pdlc-framework
cd pdlc-framework

# 2. Create a product directory from the template
cp -r products/_template products/<your-product-name>

# 3. Fill in the agent context file
vi products/<your-product-name>/CLAUDE.md

# 4. Run the onboarding agent team
# Follow: onboarding/runbook.md → Discovery Path

# 5. Human review gate (HITL) — review discovery outputs
# See: products/<your-product-name>/discovery/

# 6. Proceed to Phase 2: Spec
# Follow: spec/pipeline.md
```

### Greenfield New Product

```bash
# 1. Create a product directory
cp -r products/_template products/<your-product-name>

# 2. Fill in context + proposal
vi products/<your-product-name>/CLAUDE.md
cp spec/templates/proposal.md products/<your-product-name>/spec/proposal.md
vi products/<your-product-name>/spec/proposal.md

# 3. Proceed to Phase 2: Spec
# Follow: spec/pipeline.md
```

---

## To Do

> This framework is actively being built. Contributions welcome.

### ✅ Foundation — Complete
- [x] Framework design philosophy and core ethos
- [x] McKinsey Horizon maturity model integration
- [x] Three-tier oversight model (HITL / HOTL / HOOTL)
- [x] Spec pipeline design (`proposal → spec → design → ADRs → tasks`)
- [x] Human team role definitions (Orchestrator, Architect, Engineer, Ops)
- [x] Governance model (HITL decision matrix, escalation paths, cost guardrails, audit log spec)
- [x] Agent Interface Contract (loose coupling for backend swap)
- [x] Runtime standards (Java 21 + Spring Boot 3.x primary; extensibility hooks)
- [x] Pod performance metrics framework
- [x] Product `_template` directory scaffold + `CLAUDE.md`

### 🔄 In Progress — Phase Scaffold
- [ ] **Onboarding phase** — agent definitions (Discovery, Doc, Dependency) + prompt templates + runbook
- [ ] **Spec phase** — pipeline templates (proposal, spec, design, ADR, tasks) + agent definitions + prompts
- [ ] **Build phase** — workflow + Harness CI pipeline YAML + prompt templates (Dev, Test, Review agents)
- [ ] **Ops phase** — runbooks (incident, dependency, game day) + agent definitions + prompts

### 📋 Planned — Extensions & Tooling
- [ ] End-to-end example walkthrough (proposal → running Spring Boot service on GKE)
- [ ] Harness pipeline template library (reusable stage templates for Java/Spring Boot)
- [ ] GKE base Helm chart for Spring Boot services (liveness/readiness, HPA, Workload Identity)
- [ ] VMware VCF deployment runbook
- [ ] ADR compliance CI linting script ("architectural linting" for Harness)
- [ ] CLAUDE.md staleness detector (agent that flags when context is out of date)
- [ ] Per-product agent cost dashboard template
- [ ] Python/FastAPI runtime standard extension
- [ ] Node/TypeScript runtime standard extension
- [ ] Go runtime standard extension
- [ ] LangGraph adapter for Agent Interface Contract
- [ ] Google ADK adapter for Agent Interface Contract
- [ ] CrewAI adapter for Agent Interface Contract
- [ ] Game day scenario library (expanded beyond core 5 scenarios)
- [ ] Interactive onboarding CLI (`pdlc init` — guides `cp _template` and fills `CLAUDE.md`)
- [ ] Spec drift detection agent (flags when code diverges from spec)
- [ ] Multi-product portfolio dashboard (aggregate metrics across product pods)

---

## Extending the Framework

### Adding a New Runtime Standard
See [runtime-standards.md](framework/standards/runtime-standards.md) — add a new section for your runtime (Python/FastAPI, Node/TypeScript, Go, etc.) and update the discovery agent's heuristics for that language.

### Swapping the Agent Backend
See [agent-interface.md](framework/standards/agent-interface.md) — the Agent Interface Contract defines how to map any agent definition to LangGraph, Google ADK, CrewAI, or OpenAI Agents SDK without changing agent role definitions.

### Adding a New Agent Role
1. Create `framework/agents/<phase>/<agent-name>.md` following the standard agent definition structure
2. Add corresponding prompt template in the phase's `prompts/` directory
3. Update the phase README and relevant runbooks

---

## Background Reading

- McKinsey: [The AI Revolution in Software Development](https://www.mckinsey.com/capabilities/tech-and-ai/our-insights/the-ai-revolution-in-software-development)
- Anthropic: [Claude Code — Agentic Coding Best Practices](https://docs.anthropic.com/en/docs/claude-code)
- GitHub: [Spec Kit — Structured AI Development Pipeline](https://githubnext.com/projects/github-spark)
- Martin Fowler: [Architecture Decision Records](https://martinfowler.com/bliki/ArchitecturalDecisionRecord.html)
- Bain & Company: AI-Native Engineering Teams (2025)

---

## Contributing

1. Fork the repo
2. Create a feature branch: `git checkout -b feat/your-feature`
3. Follow the agent definition structure in `framework/agents/` for new agents
4. Follow the runtime standards structure in `framework/standards/` for new runtime support
5. Open a PR with a clear description of what you're adding and why

---

## License

MIT — see [LICENSE](LICENSE)

---

*Built on research from McKinsey, Bain, Anthropic, Google, Thoughtworks, EY, and KPMG on AI-native engineering team design.*
