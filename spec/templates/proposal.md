---
product_name: ""
service_type: "" # api | microservice | monolith | data-pipeline
runtime: "" # java-springboot | other (specify below)
runtime_detail: "" # e.g., "Java 21 + Spring Boot 3.3.x" or "Python 3.12 + FastAPI"
entry_point: "" # discovery | greenfield
status: draft # draft | approved
author: ""
date: "" # YYYY-MM-DD
reviewed_by: ""
approved_date: ""
---

# Proposal: {{PRODUCT_NAME}}

> **Instructions:** Complete all sections. Sections marked `[REQUIRED]` are mandatory before the pipeline will proceed.  
> Sections marked `[RECOMMENDED]` are strongly encouraged but the Architect Agent will attempt to infer them.  
> Sections marked `[IF APPLICABLE]` only apply to certain contexts.  
> Delete instructional callouts before marking `status: approved`.

---

## Product Vision `[REQUIRED]`

<!--
1–3 sentences only. Answer: What does this product do? Who uses it? What outcome does it create?
Do NOT describe implementation. Write this for a non-technical stakeholder.

Good example:  
  "The Order Fulfilment API provides a unified interface for warehouse management systems to query
   and update order status across all fulfilment centres. It replaces four bespoke integrations
   with a single contract-tested service, reducing integration failures and enabling same-day
   status reporting for customer-facing applications."

Bad example:
  "We will build a Spring Boot microservice with a Postgres database that uses REST endpoints
   to handle order data using JPA repositories..."
-->

_Replace this line with your 1–3 sentence product vision._

---

## Problem Statement `[REQUIRED]`

<!--
What pain point, capability gap, or risk does this product address?
Be specific: quantify the problem if possible (e.g., "3–4 hours of manual reconciliation daily",
"4 integration failures per week causing order delays", "no audit trail for regulatory requirement X").

For discovery path: describe the current system's problem, not the proposed solution.
For greenfield: describe the market, operational or technical gap being filled.
-->

### Current State

_Describe the current situation or pain point._

### Impact

_Describe the business or technical impact of not solving this problem._

### Why Now

_What is driving the timing of this initiative?_

---

## Success Criteria `[REQUIRED]`

<!--
Measurable outcomes, not implementation details. Each criterion must be objectively verifiable.
Bad: "The system should be fast"
Good: "p95 response time ≤ 200ms at 500 RPS sustained load"

Bad: "Users should find it easy to use"
Good: "Integration partners can complete first successful API call within 30 minutes of receiving credentials"
-->

| # | Success Criterion | Measurement Method | Target Value |
|---|---|---|---|
| SC-001 | | | |
| SC-002 | | | |
| SC-003 | | | |

---

## Service Type Rationale `[REQUIRED]`

<!--
Why was the service type (api / microservice / monolith / data-pipeline) chosen?
This decision will be used to select the appropriate agent configuration and deployment template.

Consider:
- Team size and deployment independence requirements
- Data ownership and domain boundaries
- Operational complexity tolerance
- Existing system architecture
-->

**Chosen service type:** `{{SERVICE_TYPE}}`

**Rationale:**

_Explain why this service type is appropriate for this product and its context._

**Deployment target:** `{{GKE_NAMESPACE}} on GCP GKE` | `{{VCF_ENVIRONMENT}} on VMware VCF`

**Deployment rationale:**

_Explain why this deployment target is appropriate (e.g., data residency, latency to consumers, existing platform investment)._

---

## Scope `[REQUIRED]`

### In Scope

<!--
List capabilities, features or functions that this product WILL deliver in this iteration.
Be explicit — anything not listed here is assumed to be out of scope.
-->

- 

### Out of Scope `[REQUIRED — be explicit]`

<!--
Explicitly list what this product will NOT do. This prevents scope creep and gives agents
clear boundaries for what NOT to generate.
This is as important as the in-scope list.
-->

- 

### Future Considerations (Not in Current Scope)

<!--
Things you expect to add in a later iteration. Listing them here helps agents design
extension points without building the features now.
-->

- 

---

## Key Constraints `[REQUIRED]`

<!--
Non-functional constraints that MUST be met. These feed directly into the NFR section of the spec.
Use numbers, not adjectives.
-->

### Performance

| Metric | Target | Notes |
|---|---|---|
| Latency p50 | | |
| Latency p95 | | |
| Latency p99 | | |
| Throughput (RPS) | | Peak sustained |
| Batch size (if pipeline) | | Records per run |

### Availability

| Metric | Target | Notes |
|---|---|---|
| SLA (uptime) | | e.g., 99.9% |
| RTO | | Recovery Time Objective |
| RPO | | Recovery Point Objective |
| Maintenance window | | |

### Security and Compliance `[REQUIRED]`

| Constraint | Detail |
|---|---|
| Authentication mechanism | e.g., OAuth 2.0 / API Key / mTLS |
| Authorisation model | e.g., RBAC / ABAC / Scopes |
| Data classification | e.g., Internal / Confidential / PII |
| Encryption at rest | Required? Specify standard |
| Encryption in transit | TLS version |
| Data residency | e.g., EU-only, UK-only, no restriction |
| Compliance frameworks | e.g., GDPR, PCI-DSS, ISO 27001, none |
| Audit log requirement | Yes/No — if yes, what events |
| PII handling | Describe how PII is stored, masked, or excluded |

### Scalability

| Concern | Detail |
|---|---|
| Expected peak load | |
| Growth projection (12 months) | |
| Scale trigger | e.g., CPU 70%, queue depth 1000 |
| Scale target (GKE HPA / VCF) | e.g., min 2 / max 10 pods |

---

## Integration Points `[REQUIRED]`

<!--
List all known upstream and downstream systems. Incomplete integration lists are the #1 cause
of spec re-work. Include systems you depend on AND systems that depend on you.
-->

### Upstream Dependencies (this product consumes)

| System | Protocol | Data Exchanged | SLA / Reliability | Owner |
|---|---|---|---|---|
| | | | | |

### Downstream Consumers (this product serves)

| System | Protocol | Data Provided | Expected Volume | Owner |
|---|---|---|---|---|
| | | | | |

### Event Streams (if applicable)

| Topic / Queue | Direction (Produce/Consume) | Broker | Schema |
|---|---|---|---|
| | | | |

---

## Discovery Summary `[IF APPLICABLE: discovery entry point only]`

<!--
Complete this section if entry_point is 'discovery'.
Summarise the key findings from the discovery phase that are relevant to this proposal.
Link to the full discovery-report.md and architecture-map.md documents.

If entry_point is 'greenfield', delete this section or replace with 'Market / Technical Context'.
-->

**Discovery Report:** [Link to discovery-report.md]  
**Architecture Map:** [Link to architecture-map.md]

### Key Findings

_What did discovery reveal about the current system that directly influences this proposal?_

### Technical Debt Identified

_List significant technical debt discovered that this product must address or work around._

### Risks from Current State

_What risks exist in the current implementation that the new product must mitigate?_

---

## Market / Technical Context `[IF APPLICABLE: greenfield entry point only]`

<!--
Complete this section if entry_point is 'greenfield'.
Provide context that justifies building this product from scratch.
Delete if entry_point is 'discovery'.
-->

_What market trend, technology shift, or organisational capability gap is this product responding to?_

---

## Target Architecture Principles `[RECOMMENDED]`

<!--
List the high-level architecture principles this product will follow.
These will be used by the Architect Agent when making design trade-off decisions.
If none are specified, the agent will apply the default principles from runtime-standards.md.

Examples:
- "Prefer synchronous REST over async messaging for initial release; introduce async when load demands it"
- "All data written to this service is the system of record — no read-through caching"
- "Deploy stateless; all state in Postgres"
- "Design for zero-downtime deployment from day one"
-->

1. 
2. 
3. 

---

## Open Questions `[REQUIRED — list what is blocking]`

<!--
List unresolved questions that could block or significantly alter the spec.
The Architect Agent will halt spec generation if it encounters any of these and will prompt
a human for resolution. Better to surface them here proactively.

If there are no open questions, write "None — proposal is complete" explicitly.
-->

| # | Question | Impact if Unresolved | Owner | Due Date |
|---|---|---|---|---|
| OQ-001 | | | | |
| OQ-002 | | | | |

---

## Human Approval Sign-off

<!--
The pipeline will not proceed past this point until all sign-offs are completed.
The author and approver must be different people for HITL compliance.
-->

| Role | Name | Date | Signature / Approval Ref |
|---|---|---|---|
| **Author** | | | _(write: "Self-authored")_ |
| **Product Owner** | | | |
| **Technical Architect** | | | |
| **Security Review** (if PII/Compliance) | | | _(or "N/A — no PII")_ |

**Final status change:** Update YAML frontmatter `status` from `draft` to `approved` only after all required sign-offs above are complete.

---

*Template version: 1.0 | Framework: PDLC 1.0*
