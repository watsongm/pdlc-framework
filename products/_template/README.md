# Product Template

This directory is the template for all products managed by the AI-Native PDLC Framework. Copy this entire directory to `products/<your-product-name>/` when onboarding a new product.

---

## Quick Start

```bash
# Copy template
cp -r products/_template products/my-new-service

# Rename and fill in CLAUDE.md with your product's details
vi products/my-new-service/CLAUDE.md

# Choose your entry point:
# (a) Existing product → run discovery agent (see onboarding/runbook.md)
# (b) Greenfield → fill in spec/proposal.md directly (see onboarding/runbook.md)
```

---

## Directory Structure

```
products/my-service/
├── CLAUDE.md                 ← Agent context file — MUST be kept current
├── README.md                 ← This file (replace with product README)
├── discovery/                ← Onboarding phase outputs
│   ├── discovery-report.md   ← Discovery Agent output
│   ├── architecture-map.md   ← Dependency Agent output
│   ├── domain-glossary.md    ← Doc Agent output
│   └── interface-inventory.md← Dependency Agent output
├── spec/                     ← Spec phase documents
│   ├── proposal.md           ← Human-authored product intent
│   ├── spec.md               ← Architect Agent output (HITL approved)
│   └── design.md             ← Architect Agent output (HITL approved)
├── adrs/                     ← Architecture Decision Records
│   ├── compliance-rules-summary.md ← ADR Agent output (CI linting input)
│   └── ADR-NNNN-*.md         ← Individual ADRs (HITL approved)
├── tasks/                    ← Agent task backlog
│   └── tasks.md              ← Architect Agent output
└── ops/                      ← Operational outputs
    ├── reports/              ← Monitor Agent reports
    ├── incidents/            ← Incident Agent reports
    └── post-mortems/         ← Post-incident post-mortems
```

---

## Human Responsibilities

| Responsibility | Who | Frequency |
|---|---|---|
| Keep CLAUDE.md current | Engineer | After every significant code change |
| Review ADRs for currency | Architect | Quarterly |
| Review discovery outputs | Architect | After major refactors |
| Archive resolved tasks | Engineer | Per sprint |

---

*Template owner: Framework Architect | Version: 1.0*
