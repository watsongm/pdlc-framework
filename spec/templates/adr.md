---
adr_id: "ADR-0000"           # Format: ADR-NNNN (four-digit, zero-padded, sequential)
title: ""                     # Short imperative title: "Use PostgreSQL as primary data store"
status: proposed              # proposed | accepted | deprecated | superseded
date: ""                      # YYYY-MM-DD (date proposed)
accepted_date: ""             # YYYY-MM-DD (date status changed to accepted)
authors:
  - ""                        # Name or agent identifier (e.g., "adr-agent@1.0", "Jane Smith")
product_name: ""              # Product this ADR applies to (or "global" for cross-cutting)
related_adrs:
  - ""                        # ADR IDs of related or conflicting ADRs
supersedes: ""                # ADR-NNNN if this ADR replaces an earlier decision
superseded_by: ""             # ADR-NNNN if this ADR has been replaced (set when deprecating)
review_date: ""               # YYYY-MM-DD — when this ADR must be reconsidered
tags:
  - ""                        # e.g., database, security, api-design, deployment, messaging
---

# {{ADR_ID}}: {{TITLE}}

> **Status:** `{{STATUS}}`  
> **Product:** {{PRODUCT_NAME}}  
> **Date:** {{DATE}}  
> **Authors:** {{AUTHORS}}  
> **Related ADRs:** {{RELATED_ADRS}}

---

## Context

<!--
Describe the situation, forces, and problem that made this decision necessary.
Write this section in the past tense as if describing the state of affairs at the time of the decision.

Good context sections answer:
- What is the system trying to do at this decision point?
- What constraints exist (technical, organisational, compliance)?
- Why can't we defer this decision?
- What was the trigger that surfaced this decision?

This section should be understandable by someone reading the codebase 2 years from now
with no knowledge of the original discussions.
-->

_Replace with context narrative._

---

## Decision Drivers

<!--
List the forces influencing this decision. Be honest — include forces that cut against the chosen option.
These are not requirements; they are the criteria by which the options were evaluated.

Examples:
- Operational expertise: team has 5 years of experience operating PostgreSQL
- Compliance: GDPR requires data residency in EU; chosen option must support this
- Performance: sub-200ms p95 latency at 500 RPS under write-heavy workload
- Operational cost: minimise managed service cost; prefer self-hosted where SLA allows
- Consistency: align with the platform team's standard technology list
-->

- **[Driver Name]:** _Description of the force_
- **[Driver Name]:** _Description of the force_
- **[Driver Name]:** _Description of the force_

---

## Considered Options

<!--
List every option seriously considered. Do NOT omit options just because they were obviously unsuitable
— document why they were ruled out. This is the most valuable part of the ADR for future readers.

Each option should have:
1. A brief description of what it would mean to choose this option
2. Why it was considered (what made it a plausible choice)
3. A summary of its pros and cons (use the full section below for the chosen option)
-->

### Option 1: {{Option Name}}

_Brief description of what choosing this option would mean._

**Why considered:** _What made this a plausible choice?_

**Pros:**
- 

**Cons:**
- 

---

### Option 2: {{Option Name}}

_Brief description._

**Why considered:**

**Pros:**
- 

**Cons:**
- 

---

### Option 3: {{Option Name}} _(add as many options as needed)_

_Brief description._

**Why considered:**

**Pros:**
- 

**Cons:**
- 

---

## Decision

<!--
State the chosen option clearly and unambiguously.
Explain WHY this option was chosen relative to the others — connect it explicitly to the decision drivers.
Do NOT just say "we chose Option 2 because it's better". Say which drivers it satisfies better
than the alternatives and why.

This section is written in present tense: "We use...", "We adopt...", "We will..."
-->

**We choose Option N: {{Chosen Option Name}}.**

_Explanation of why this option best satisfies the decision drivers listed above._

_Specific configuration or implementation constraints that accompany this decision (e.g., "We use PostgreSQL 15 with Flyway for migrations; Hibernate DDL auto is set to `validate`; ORM is Spring Data JPA")._

---

## Positive Consequences

<!--
What good things happen as a result of this decision?
Be concrete — not just "better performance" but "reduces p95 latency from ~400ms to ~80ms
under our target load based on benchmark X".
-->

- 
- 
- 

---

## Negative Consequences / Trade-offs

<!--
Be honest about what we're giving up or accepting by making this decision.
No decision is consequence-free. If you can't think of any trade-offs, you haven't
thought hard enough.

Future readers will trust this ADR more if it acknowledges its own downsides.
-->

- 
- 
- 

---

## Compliance Rule

<!--
MACHINE-READABLE constraint for CI architectural linting.
This field is parsed by the Harness arch-linter step and validated against every PR diff.

REQUIRED FORMAT:
  rule_id: ADR-NNNN-RNN
  description: "Plain English description of the rule"
  scope: file_pattern | package_pattern | annotation_presence | annotation_absence | class_hierarchy | import_pattern
  pattern: "the pattern to match or reject"
  action: DENY | REQUIRE | WARN
  applies_to: "glob pattern of files this rule applies to"
  message: "Message shown to developer when rule is violated"

Examples:
  - "All @RestController classes must not contain @Autowired field injection"
  - "Classes in .service. package must not import javax.persistence or jakarta.persistence directly"
  - "All Spring @Service classes must have at least one @Transactional method"
  - "No SQL strings in classes outside the .infrastructure.persistence. package"
-->

```yaml
compliance_rules:
  - rule_id: "{{ADR_ID}}-R01"
    description: "{{Plain English description of what must / must not happen}}"
    scope: annotation_absence          # annotation_presence | annotation_absence | class_hierarchy | import_pattern | file_pattern
    pattern: "{{pattern to detect violation}}"
    action: DENY                       # DENY (block PR) | REQUIRE (must be present) | WARN (advisory)
    applies_to: "**/*Controller.java"  # Glob pattern for files this rule applies to
    message: |
      Violation of {{ADR_ID}}: {{Title}}.
      {{Specific explanation of what the developer did wrong and what to do instead.}}
      See: adrs/{{ADR_ID}}-{{title-slug}}.md

  - rule_id: "{{ADR_ID}}-R02"
    description: "{{Second rule if needed}}"
    scope: import_pattern
    pattern: "{{import to deny or require}}"
    action: DENY
    applies_to: "**/*.java"
    message: |
      Violation of {{ADR_ID}}: {{Title}}.
      {{Explanation.}}
      See: adrs/{{ADR_ID}}-{{title-slug}}.md
```

---

## Agent Instructions

<!--
How should AI agents interpret and apply this ADR when generating or reviewing code?
This section is injected into agent context alongside the ADR.

Be specific and actionable. Agents follow explicit rules better than general principles.

Include:
- What to do when generating new code in this domain
- What to check when reviewing existing code
- What to refuse to do (and what to do instead)
- How to handle edge cases or exceptions to this ADR
- When to escalate to a human rather than proceeding
-->

### When generating code

- _Explicit instruction for code generation_
- _Explicit instruction for code generation_

### When reviewing code

- _Check for this pattern_
- _Flag this anti-pattern_

### When this ADR does not apply

- _Conditions under which this ADR is explicitly not applicable_

### Escalation triggers

- Escalate to a human if: _condition_
- Escalate to a human if: _condition_

---

## Review Date

**Next review:** {{REVIEW_DATE}}

**Review trigger conditions** (review regardless of date if any of these occur):
- _e.g., "If a new version of the technology is released with breaking changes"_
- _e.g., "If the team size doubles and the current approach creates operational overhead"_
- _e.g., "If a new compliance requirement is introduced that conflicts with this decision"_

---

## Changelog

| Date | Author | Change |
|---|---|---|
| {{DATE}} | {{AUTHOR}} | Initial proposal |

---

*Template version: 1.0 | Framework: PDLC 1.0*
