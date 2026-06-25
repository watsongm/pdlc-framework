# Doc Ingest Agent — Prompt Template

> **Used by:** Doc Agent (Claude Code)
>
> **Phase:** Onboarding
>
> **Agent definition:** [`framework/agents/onboarding/doc-agent.md`](../../framework/agents/onboarding/doc-agent.md)
>
> **Output template:** [`templates/domain-glossary.md`](../templates/domain-glossary.md)

---

## Context Injection Block

Before running this prompt, load the following into the agent's context window. In Claude Code, use `@file` syntax or include file paths in the task invocation.

| Context Item | File / Source | Required? | Notes |
|---|---|---|---|
| Product CLAUDE.md | `products/{{PRODUCT_NAME}}/CLAUDE.md` | **Required** | Primary product context |
| Framework ADRs | `framework/adr/*.md` | Required | Governs terminology standards |
| Domain glossary template | `onboarding/templates/domain-glossary.md` | **Required** | Output format the agent must produce |
| Discovery report (if exists) | `products/{{PRODUCT_NAME}}/discovery/discovery-report.md` | Recommended | Provides tech context to cross-reference against docs |
| Confluence MCP server | Configured in `~/.claude/mcp_settings.json` | Optional | Required if Confluence URLs are in `{{DOC_SOURCES}}` |

---

## System Prompt

> _Copy this text verbatim as the system prompt when invoking the Doc Agent._

```
You are a documentation ingestion specialist for the AI-Native PDLC Framework.

Your role is to read, analyse, and synthesise all available documentation for a 
software product and produce a structured domain glossary. You extract business 
domain knowledge, operational context, and non-functional requirements from 
documentation sources — and you identify conflicts between what the documentation 
says and what the code actually does.

## Core Constraints

1. CODE IS GROUND TRUTH: When documentation and code conflict, the code is correct 
   and the documentation is stale or wrong. Document the conflict explicitly — never 
   silently defer to documentation over code evidence.

2. CITE EVERYTHING: Every term definition, business rule, and operational fact must 
   cite its source (Confluence page URL and section heading, README path, wiki URL, 
   etc.). Do not synthesise claims that have no documentary basis.

3. EXTRACT, DON'T INVENT: Your job is to extract what is written in the 
   documentation. If documentation is silent on a topic, say "Not documented — 
   see code analysis." Do not invent definitions based on your general knowledge.

4. FLAG STALENESS: If documentation is older than 3 years OR contains references to 
   technologies, versions, or processes clearly contradicted by the discovery report, 
   flag the document as stale with [ESCALATE: STALE DOCS].

5. FLAG CONFLICTS: When you find a conflict between documentation and the discovery 
   report, output a conflict flag: [CONFLICT: <topic> — docs say X, code shows Y].

6. HANDLE JARGON CAREFULLY: Products often have internal jargon that is not 
   self-explanatory. Extract every non-obvious term and define it. If a term is only 
   used in code but not documented, note it as "Used in code — not documented."

7. OUTPUT FORMAT: Populate the domain-glossary.md template exactly. Do not add 
   sections that are not in the template. Fill all {{PLACEHOLDER}} values.

8. PRIVACY: Do not include customer PII, employee PII, or system credentials in your 
   output, even if present in documentation. Redact any that you encounter.
```

---

## Task Prompt

> _Replace all `{{PLACEHOLDER}}` values before sending._

```
## Documentation Ingestion Task

Ingest all available documentation for the following product and produce a 
structured domain glossary.

### Parameters

- **Product Name:** {{PRODUCT_NAME}}
- **Service Type:** {{SERVICE_TYPE}}
- **Documentation Sources:** {{DOC_SOURCES}}
  (comma-separated list of URLs and/or file paths)
- **Discovery Report Path (for cross-referencing):** {{DISCOVERY_REPORT_PATH}}
  (may be "none" if discovery has not yet run)

### Documentation Sources Detail

For each source in `{{DOC_SOURCES}}`, perform the following:

{{DOC_SOURCE_DETAILS}}
<!--
  Replace {{DOC_SOURCE_DETAILS}} with a structured list like:
  
  1. Confluence space: https://org.atlassian.net/wiki/spaces/PAYMENTS
     - Ingest all pages in this space
     - Prioritise: README, architecture pages, runbooks, API docs, glossaries
  
  2. README: /path/to/repo/README.md
     - Ingest fully
  
  3. Runbook: https://wiki.internal/payments/runbook
     - Ingest fully — focus on operational dependencies and escalation procedures
  
  4. API documentation: https://api-docs.internal/payments
     - Extract endpoint descriptions and parameter definitions for the glossary
-->

### Ingestion Instructions

Perform the following steps. Record findings after each step before continuing.

#### Step 1: Source Inventory

1. Attempt to access each source in `{{DOC_SOURCES}}`.
2. For each source, record:
   - Source type (Confluence / README / wiki / runbook / API docs / ADR / other)
   - Last modified date (if available)
   - Approximate length / page count
   - Whether it is accessible (if not: note access failure)
3. If NO sources are accessible, immediately output `[ESCALATE: NO DOCS]` and 
   halt ingestion. The domain-glossary.md will need to be populated manually.
4. Record the source inventory in Section 8 (Documentation Quality Assessment) of 
   domain-glossary.md.

#### Step 2: Confluence Ingestion (if Confluence URLs present)

If Confluence URLs are in `{{DOC_SOURCES}}` and the Confluence MCP server is 
available:

1. Use the Confluence MCP server to fetch each page via the Confluence REST API:
   - `GET /wiki/rest/api/content/{pageId}?expand=body.storage,version,ancestors`
   - For spaces: `GET /wiki/rest/api/space/{spaceKey}/content` to enumerate pages
2. For each Confluence page:
   a. Extract the page title and last modified date.
   b. Convert the storage format body to plain text.
   c. Flag any page with last modified date > 3 years ago as potentially stale.
3. If Confluence MCP is NOT available but Confluence URLs are present:
   - Attempt to fetch via standard HTTP GET (may fail if authentication required).
   - If fetch fails, note: "Confluence access requires MCP server configuration. 
     See onboarding/runbook.md § Confluence MCP Setup."
   - Output `[ESCALATE: STALE DOCS — Confluence not accessible]` if critical 
     documentation cannot be retrieved.

#### Step 3: Domain Term Extraction

For each accessible document:

1. Extract all nouns and noun phrases that appear to be domain-specific terms — 
   things that would NOT be understood without context (not generic words like 
   "request", "response", "error", but domain-specific words like "settlement", 
   "authorization hold", "chargeback", "merchant", "acquirer").
2. For each term, record:
   - The term as used in documentation (exact capitalisation matters)
   - The definition as stated in documentation (direct quote preferred)
   - The source (URL/path + section heading)
   - The bounded context it belongs to (if determinable)
3. Cross-reference each term against the discovery report (if available):
   - Does the term appear in class names, method names, or variable names in code?
   - If yes, record the code location.
   - If the term name differs between docs and code, flag as a naming inconsistency.
4. Alphabetically sort terms and populate Section 2 (Domain Term Glossary) of 
   domain-glossary.md.

#### Step 4: Bounded Context Identification

1. Look for references to distinct business capabilities, departments, or service 
   areas in documentation (e.g., "the payments domain", "order management", 
   "customer identity").
2. Determine whether this product operates within a single bounded context or spans 
   multiple.
3. For each bounded context identified:
   - Name it (derive from documentation language, not invent)
   - Describe what it encompasses
   - List which teams own it
   - List the key entities within it
4. If the product is simple and operates in a single context, create one context 
   record named after the product domain.
5. Produce a context map Mermaid diagram showing relationships between contexts 
   (if more than one).
6. Populate Section 1 (Bounded Contexts) of domain-glossary.md.

#### Step 5: Business Rule Extraction

1. Scan all documents for explicit business rules — statements about what the system 
   MUST do, MUST NOT do, or conditions that MUST be met.
   Look for language like: "must", "shall", "cannot", "only if", "at least", 
   "no more than", "within X days", "requires", "is not permitted".
2. For each business rule:
   - Assign ID: BR-001, BR-002, etc.
   - State the rule clearly (paraphrase if documentation is ambiguous, note 
     the original wording).
   - Cite the source.
   - If discovery report is available, look for implementation evidence in the 
     code. Record the implementing class/method or "not found in code."
   - If a rule is in code but NOT in documentation, note it as 
     "Implemented but not documented."
3. Flag any rule that appears to contradict another rule.
4. Populate Section 3 (Key Business Rules) of domain-glossary.md.

#### Step 6: Domain Event Extraction

1. Look for references to events, notifications, triggers, or state changes in 
   documentation — things the system signals to other systems or internal 
   components.
2. Look for messaging documentation: Kafka topic lists, Pub/Sub topic definitions, 
   event catalogues, async API specs.
3. For each event:
   - Record the event name (canonical name from documentation)
   - Record what triggers it
   - Record the producer and known consumers
   - Record the broker if documented
   - Note whether this event is found in code (from discovery report)
4. Populate Section 4 (Domain Events) of domain-glossary.md.

#### Step 7: Abbreviation and Acronym Extraction

1. Scan all documents for abbreviations and acronyms. Include:
   - Business-domain acronyms (e.g., "PAN" = Primary Account Number)
   - Internal platform/product abbreviations (e.g., "OMS" = Order Management System)
   - Technical acronyms specific to this product (e.g., "MPAN" = Meter Point 
     Administration Number)
   - Do NOT include generic technical acronyms (REST, HTTP, JSON, JVM, etc.) 
     unless they have a product-specific meaning.
2. Cross-reference with code: do these abbreviations appear in class names, 
   variable names, column names?
3. Populate Section 5 (Abbreviations and Acronyms) of domain-glossary.md.

#### Step 8: Operational and Non-Functional Knowledge Extraction

1. Scan runbooks and operational documentation for:
   - Stated SLAs / SLOs (availability targets, latency targets)
   - Capacity requirements or throughput targets
   - Known failure modes and mitigations
   - Known operational dependencies (cron jobs, certificate renewal, manual steps)
   - Incident themes from post-mortems or known issues pages
2. Populate Section 7 (Operational and Non-Functional Knowledge) of 
   domain-glossary.md.

#### Step 9: Naming Inconsistency Analysis

1. Compare term names across:
   - Documentation (what docs call it)
   - Code (class names, method names, variable names from discovery report)
   - API paths and parameter names (from discovery report or OpenAPI spec)
   - Database column/table names (from discovery report)
2. For each inconsistency found:
   - Assign ID: NI-001, NI-002, etc.
   - Document all name variants and their locations
   - Recommend a canonical term (prefer the domain documentation term unless 
     clearly outdated)
3. Populate Section 6 (Naming Inconsistencies) of domain-glossary.md.

#### Step 10: Conflict Resolution

1. For each conflict between documentation and code (discovered in prior steps):
   - Output a conflict flag: [CONFLICT: <topic> — docs say "<doc statement>", 
     code shows "<code evidence from discovery report>"]
   - Record it in the Escalation Flags section.
   - Treat code as ground truth for the glossary content.
   - Note the documentation statement as historical context or outdated.
2. Populate Section 9 (Escalation Flags) of domain-glossary.md.

### Output

Produce the completed domain glossary by populating 
`onboarding/templates/domain-glossary.md` with all findings.

Save the output to: `products/{{PRODUCT_NAME}}/discovery/domain-glossary.md`

At the end, output a summary line:
```
DOC INGESTION COMPLETE — {{PRODUCT_NAME}} — [timestamp] — Sources ingested: [N] — 
Terms extracted: [N] — Business rules: [N] — Escalations: [N]
```
```

---

## Expected Output Format

The agent must produce a populated `domain-glossary.md` file at:

```
products/{{PRODUCT_NAME}}/discovery/domain-glossary.md
```

The file must:
- Follow the template structure exactly (all 10 sections present)
- Replace all `{{PLACEHOLDER}}` values with real findings
- Include source citations for all domain terms and business rules
- Include `[CONFLICT: ...]` and `[ESCALATE: ...]` flags as appropriate
- End with the DOC INGESTION COMPLETE summary line

---

## Validation Checklist

```
[ ] domain-glossary.md exists at products/{{PRODUCT_NAME}}/discovery/
[ ] Section 1 (Bounded Contexts) — at least one context defined
[ ] Section 2 (Glossary) — at least 10 domain terms extracted (0 terms = flag for human)
[ ] Section 3 (Business Rules) — at least 3 rules extracted (if docs exist)
[ ] Section 4 (Domain Events) — populated or explicitly "None documented"
[ ] Section 5 (Abbreviations) — common internal acronyms extracted
[ ] Section 6 (Naming Inconsistencies) — any inconsistencies flagged
[ ] Section 7 (Operational Knowledge) — SLAs and known incidents populated if available
[ ] Section 8 (Documentation Quality) — all sources assessed for currency
[ ] Section 9 (Escalation Flags) — all escalations documented
[ ] No {{PLACEHOLDER}} values remain unfilled
[ ] All terms cite their documentation source
[ ] Conflicts between docs and code are explicitly flagged
```

---

## Escalation Triggers

| Trigger | Flag | Required Action |
|---|---|---|
| No documentation sources are accessible | `[ESCALATE: NO DOCS]` | Halt ingestion. Human must manually populate glossary using domain expert interviews. |
| Any documentation source is >3 years old | `[ESCALATE: STALE DOCS]` | Flag the specific source. Human must verify currency. |
| Major conflict between docs and code (same concept, contradictory behaviour described) | `[ESCALATE: CONFLICT]` | Document the conflict. Code is ground truth. Human to confirm and update docs. |
| Fewer than 5 domain terms extractable (documentation is too sparse) | `[ESCALATE: SPARSE DOCS]` | Note in report. Recommend domain expert interviews. |

---

## Confluence MCP Configuration

If documentation is on Confluence and authentication is required, configure the Confluence MCP server before running this agent:

```json
// ~/.claude/mcp_settings.json
{
  "mcpServers": {
    "confluence": {
      "command": "npx",
      "args": ["-y", "@anthropic-mcp/confluence-server"],
      "env": {
        "CONFLUENCE_BASE_URL": "https://your-org.atlassian.net",
        "CONFLUENCE_TOKEN": "your-personal-access-token",
        "CONFLUENCE_EMAIL": "your-email@example.com"
      }
    }
  }
}
```

The Confluence MCP server exposes the following tools that the Doc Agent will use:
- `confluence_get_page` — fetch a single page by ID
- `confluence_search` — search pages by CQL query
- `confluence_get_space` — enumerate pages in a space
- `confluence_get_page_children` — traverse page hierarchy

---

## Alternative Backend Note

This prompt is designed for **Claude Code** with optional Confluence MCP.

### LangGraph

- Replace Confluence MCP with a `ConfluenceAPITool` (Python requests or `atlassian-python-api`)
- `WebFetchTool` for wiki / web-based documentation
- `ReadFileTool` for README and local documentation
- Structure as a `StateGraph` with a node per ingestion step
- Conflicts and escalations → `conditional_edges` to a human-review interrupt node

### Google ADK

- Implement as a sequential `DocumentIngestAgent` with `FunctionTool` wrappers:
  - `fetch_confluence_page(page_id)` → calls Confluence REST API
  - `fetch_web_doc(url)` → fetches public web documentation
  - `read_local_doc(path)` → reads local file
- Output → structured `DomainGlossary` dataclass → serialise to domain-glossary.md
- Use `AgentTool` to delegate Confluence pagination to a sub-agent if space is large
