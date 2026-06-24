# Onboarding Runbook

> **Who runs this:** Onboarding Owner (engineer assigned to this phase)
>
> **Estimated duration:** ≤ 5 working days (Discovery) | ≤ 2 working days (Greenfield)
>
> **Overview:** [README.md](./README.md)

---

## Before You Begin

Confirm you have the following before starting any steps:

```
[ ] Product name agreed (kebab-case, ≤ 32 chars)        example: payments-gateway
[ ] Onboarding Owner assigned (one person, accountable)
[ ] Path confirmed: DISCOVERY or GREENFIELD
[ ] This runbook open alongside a terminal
```

---

## PATH A — Discovery (Existing Product)

Use this path when a codebase already exists.

---

### Step A1 — Create the Product Directory

```bash
# From the PDLC repo root
PRODUCT_NAME="your-product-name"   # replace with kebab-case name

cp -r products/_template products/${PRODUCT_NAME}
git add products/${PRODUCT_NAME}/
git commit -m "chore: scaffold onboarding directory for ${PRODUCT_NAME}"
```

Resulting structure:

```
products/
  your-product-name/
    CLAUDE.md               ← agent context file (edit next)
    discovery/              ← agent outputs land here
    spec/                   ← used in Spec Phase
    design/                 ← used in Design Phase
    adr/                    ← Architecture Decision Records
    tasks/                  ← used in Build Phase
```

---

### Step A2 — Fill in CLAUDE.md

Open `products/${PRODUCT_NAME}/CLAUDE.md` and populate all known fields. This file is **loaded by every agent** as the primary context anchor.

Minimum required fields for the Discovery Path:

```markdown
# Product Context: your-product-name

## Identity
- **Product Name:** your-product-name
- **Service Type:** api | microservice | monolith | data-pipeline   ← pick one
- **Team:** <team name>
- **Onboarding Owner:** <name, email>
- **Phase:** onboarding

## Repository
- **Repo URL:** https://github.com/org/repo
- **Primary Branch:** main
- **Language:** Java 21
- **Framework:** Spring Boot 3.x
- **Build Tool:** Maven | Gradle   ← pick one

## Known Context
- <Any known facts: what does this service do? Who calls it?>
- <Known downstream dependencies, if any>
- <Known databases, if any>

## Documentation Sources
- Confluence: <space URL or page URLs>
- README: <path or URL>
- Runbooks: <link>

## Infrastructure
- **Deployment Target:** GCP GKE | VMware VCF | hybrid
- **Kubernetes Namespace:** <namespace or "unknown">
- **Environment URLs:** dev=<url> staging=<url> prod=<url>

## Agent Instructions
- Treat all secrets, credentials, and PII references as redacted — do not output them.
- Do not modify any files in the repository.
- Flag anything you are uncertain about explicitly.
```

> [!IMPORTANT]
> If you cannot fill in a field, write `unknown` — never leave a field blank. Agents interpret blank fields as "not applicable" and may skip discovery for that area.

Commit the file:

```bash
git add products/${PRODUCT_NAME}/CLAUDE.md
git commit -m "chore: seed CLAUDE.md for ${PRODUCT_NAME} onboarding"
```

---

### Step A3 — Run the Discovery Agent

The Discovery Agent reads the codebase and produces `discovery/discovery-report.md`.

#### Option 1: Claude Code (recommended)

Open Claude Code in the PDLC repo root:

```bash
claude
```

Then run the slash command:

```
/project:run-discovery PRODUCT=your-product-name REPO_PATH=/path/to/cloned/repo
```

Or paste the prompt manually from `onboarding/prompts/discovery-prompt.md`, substituting:
- `{{PRODUCT_NAME}}` → `your-product-name`
- `{{REPO_PATH}}` → `/absolute/path/to/the/cloned/repo`
- `{{SERVICE_TYPE}}` → `api` | `microservice` | `monolith` | `data-pipeline`
- `{{TECH_STACK_HINT}}` → e.g., `Java 21, Spring Boot 3.2, Maven, PostgreSQL`

#### Option 2: Harness Pipeline

Trigger the `onboarding-discovery` Harness pipeline (see [Harness Integration](#harness-integration) below).

#### Expected Output

The agent writes to:

```
products/your-product-name/discovery/discovery-report.md
```

> [!NOTE]
> The Discovery Agent is **HOTL** — it runs autonomously and you review the output afterwards. It does **not** require approval for each step. It will **never modify** the target repository.

**Estimated runtime:** 15–60 minutes depending on codebase size.

**Watch for escalation notices in the output:**
- `[ESCALATE: ACCESS DENIED]` → stop, fix access, re-run
- `[ESCALATE: LARGE CODEBASE]` → codebase >500k LOC, manual review needed
- `[ESCALATE: NO TESTS]` → no test directory found, flag to team

---

### Step A4 — Run the Doc Agent

The Doc Agent ingests all available documentation and produces `discovery/domain-glossary.md`.

#### Run via Claude Code

```
/project:run-doc-ingest PRODUCT=your-product-name SOURCES="<comma-separated URLs or paths>"
```

Or paste the prompt from `onboarding/prompts/doc-ingest-prompt.md`, substituting:
- `{{PRODUCT_NAME}}` → `your-product-name`
- `{{DOC_SOURCES}}` → list of Confluence URLs, README paths, wiki links, runbook URLs

#### If Confluence Credentials Are Available

Ensure the Confluence MCP server is configured:

```bash
# In your Claude Code MCP config (~/.claude/mcp_settings.json)
{
  "mcpServers": {
    "confluence": {
      "command": "npx",
      "args": ["-y", "@anthropic-mcp/confluence-server"],
      "env": {
        "CONFLUENCE_BASE_URL": "https://your-org.atlassian.net",
        "CONFLUENCE_TOKEN": "<personal access token>"
      }
    }
  }
}
```

#### Expected Output

```
products/your-product-name/discovery/domain-glossary.md
```

**Estimated runtime:** 30–90 minutes depending on volume of documentation.

**Watch for escalation notices:**
- `[ESCALATE: STALE DOCS]` → documentation >3 years old
- `[ESCALATE: NO DOCS]` → no documentation found, must be filled manually
- `[ESCALATE: CONFLICT]` → code and docs contradict — code is ground truth

---

### Step A5 — Run the Dependency Agent

The Dependency Agent maps all service-to-service and service-to-data dependencies, producing `architecture-map.md` and `interface-inventory.md`.

> [!IMPORTANT]
> The Dependency Agent **requires** `discovery-report.md` to exist. Run Step A3 first.

#### Run via Claude Code

```
/project:run-dep-map PRODUCT=your-product-name K8S_NAMESPACE=<namespace or "none">
```

Or paste the prompt from `onboarding/prompts/dep-map-prompt.md`, substituting:
- `{{PRODUCT_NAME}}` → `your-product-name`
- `{{DISCOVERY_REPORT_PATH}}` → `products/your-product-name/discovery/discovery-report.md`
- `{{K8S_NAMESPACE}}` → Kubernetes namespace (or `none`)
- `{{SERVICE_REGISTRY}}` → e.g., `gcp-service-directory` | `consul` | `none`

#### Expected Outputs

```
products/your-product-name/discovery/architecture-map.md
products/your-product-name/discovery/interface-inventory.md
```

**Watch for escalation notices:**
- `[ESCALATE: CIRCULAR DEPENDENCY]` → circular service dependency found — architectural concern
- `[ESCALATE: SHARED DATABASE]` → multiple services share one DB schema — major concern
- `[ESCALATE: LARGE DEPENDENCY GRAPH]` → >20 external dependencies

---

### Step A6 — Human Review Gate (HITL)

> [!IMPORTANT]
> This is a mandatory **Human-in-the-Loop (HITL)** gate. No progression to the Spec Phase until the Onboarding Owner signs off.

#### Review Checklist

Open each agent output and work through:

**discovery-report.md**
```
[ ] Product overview is accurate
[ ] Tech stack versions are correct
[ ] All controllers/endpoints are listed
[ ] Data layer reflects reality (correct DB, ORM, schema tool)
[ ] External integrations are all captured
[ ] Testing state is accurate (coverage % is believable)
[ ] CI/CD state reflects current pipelines
[ ] Known technical debt is representative
[ ] Security posture section is complete
[ ] Discovery Confidence Score is reasonable
```

**domain-glossary.md**
```
[ ] Bounded contexts are correct (if applicable)
[ ] Key domain terms are defined correctly
[ ] Business rules are accurate
[ ] Naming inconsistencies are flagged where they exist
```

**architecture-map.md**
```
[ ] Service topology diagram is accurate
[ ] Component inventory is complete
[ ] Dependency matrix shows all known calls
[ ] Blast radius table is reasonable
[ ] Infrastructure deployment map (GKE vs VCF) is correct
```

**interface-inventory.md**
```
[ ] All REST endpoints are listed
[ ] Async interfaces (Pub/Sub, Kafka) are captured
[ ] Database interfaces are correct
[ ] External third-party integrations are all listed
```

#### Annotating Gaps

Add your annotations directly in each file under the **Human Review Notes** section, using this format:

```markdown
## Human Review Notes

- [CONFIRMED] Tech stack section is accurate as of 2024-Q4
- [GAP] Flyway migration history — agent could not access DB, verify V23 is latest
- [CORRECTION] FeignClient for `inventory-service` is missing — add manually
- [CONCERN] No integration tests found — raise with team lead
- [UNKNOWN] SLA for payment-gateway dependency — not documented anywhere
```

#### Sign-off

When satisfied, commit the annotated files and tag:

```bash
git add products/${PRODUCT_NAME}/discovery/
git commit -m "chore: human review gate complete for ${PRODUCT_NAME} onboarding"
git tag onboarding-complete/${PRODUCT_NAME}
git push origin --tags
```

---

### Step A7 — Produce the Onboarding Summary

After human review, produce a brief onboarding summary in `products/${PRODUCT_NAME}/discovery/onboarding-summary.md`. This is a free-form human-written document (1–2 pages) that captures:

- What this service does (1 paragraph, plain English)
- Top 3–5 things the Spec Phase team needs to know
- Key risks or concerns discovered
- Any gaps that remain unresolved and how to fill them
- Confirmation that the knowledge base is ready for Spec Phase

---

### Step A8 — Hand Off to Spec Phase

1. Post a message in the team channel: "Onboarding complete for `{PRODUCT_NAME}`. Discovery outputs committed and tagged `onboarding-complete/{PRODUCT_NAME}`."
2. Open `spec/README.md` and follow the Spec Phase entry checklist.
3. The Spec Agent will automatically pick up `CLAUDE.md` and all `discovery/` files.

---

## PATH B — Greenfield (New Product)

Use this path when there is no existing codebase to discover.

---

### Step B1 — Create the Product Directory

```bash
PRODUCT_NAME="your-new-product-name"

cp -r products/_template products/${PRODUCT_NAME}
git add products/${PRODUCT_NAME}/
git commit -m "chore: scaffold greenfield product directory for ${PRODUCT_NAME}"
```

---

### Step B2 — Fill in CLAUDE.md with Intent

Open `products/${PRODUCT_NAME}/CLAUDE.md`. For a greenfield product, focus on **intent** rather than discovered facts:

```markdown
# Product Context: your-new-product-name

## Identity
- **Product Name:** your-new-product-name
- **Service Type:** api | microservice | monolith | data-pipeline   ← pick one
- **Team:** <team name>
- **Onboarding Owner:** <name, email>
- **Phase:** onboarding (greenfield)
- **Status:** NEW BUILD — no existing codebase

## Intent
- <What problem does this product solve?>
- <Who are the primary users or consumers?>
- <What are the top 3 capabilities it must deliver?>

## Target Architecture
- **Primary Runtime:** Java 21 + Spring Boot 3.x
- **Build Tool:** Maven | Gradle
- **Deployment Target:** GCP GKE | VMware VCF | hybrid
- **Service Type Rationale:** <why this type was chosen>

## Constraints and Non-Negotiables
- <Any known technical constraints>
- <Compliance or regulatory requirements>
- <Integration requirements (must talk to system X)>
- <Performance / scalability requirements (rough)>

## Agent Instructions
- This is a greenfield product. Do not assume any existing code or infrastructure.
- Apply all framework defaults for Java 21 + Spring Boot 3.x.
- Flag any design decision that requires human input.
```

```bash
git add products/${PRODUCT_NAME}/CLAUDE.md
git commit -m "chore: seed CLAUDE.md intent for greenfield ${PRODUCT_NAME}"
```

---

### Step B3 — Confirm Service Type

Before writing the proposal, confirm the service type. Use this guide:

| Service Type | When to Choose |
|---|---|
| `api` | Primarily exposes REST/gRPC endpoints; relatively stateless; consumed by other services or a frontend |
| `microservice` | Part of a larger system; bounded context; has its own database; deploys independently |
| `monolith` | Single deployable unit; multiple capabilities in one service; typically runs on VCF |
| `data-pipeline` | Batch or streaming data processing; ingests, transforms, and outputs data; not typically request-driven |

Record the confirmed service type in `CLAUDE.md`.

---

### Step B4 — Write the Proposal

Create `products/${PRODUCT_NAME}/spec/proposal.md`. Use the template at `spec/templates/proposal.md` as your starting point.

The proposal must answer:
1. **Problem statement** — what pain or gap does this solve?
2. **Proposed solution** — what will be built at a high level?
3. **Success criteria** — how will we know it works?
4. **Scope** — what is in and out of scope for the first release?
5. **Non-functional requirements** — performance, scalability, security, compliance
6. **Stakeholders** — who owns this, who must approve it?
7. **Dependencies** — what systems must this integrate with?
8. **Target service type** — confirmed in Step B3
9. **Target infrastructure** — GKE, VCF, or hybrid

> [!TIP]
> The proposal does **not** need to be perfect. The Spec Agent will interrogate it and raise clarifying questions. Write what you know; mark unknowns explicitly.

```bash
git add products/${PRODUCT_NAME}/spec/proposal.md
git commit -m "feat: add product proposal for ${PRODUCT_NAME}"
```

---

### Step B5 — Proceed to Spec Phase

No human review gate is needed at this point for greenfield — the proposal itself is the output of this phase.

1. Post in the team channel: "Greenfield proposal complete for `{PRODUCT_NAME}`. Ready for Spec Phase."
2. Open `spec/README.md` and follow the Spec Phase entry checklist.

---

## Java / Spring Boot Discovery Specifics

When running the Discovery Agent against a Java/Spring Boot codebase, the agent looks for the following specific signals. This section documents what it finds and where — useful for human reviewers validating outputs.

### Build Configuration

| Signal | File(s) | What Agent Extracts |
|---|---|---|
| Maven build | `pom.xml` | `groupId`, `artifactId`, `version`, all `<dependency>` entries with versions, plugins, Spring Boot parent version |
| Gradle build | `build.gradle` / `build.gradle.kts` | `implementation`, `testImplementation` dependencies, Spring Boot plugin version |
| Wrapper version | `mvnw` / `gradlew` | Maven/Gradle version in use |

### Application Structure

| Signal | Annotation / Class | What Agent Extracts |
|---|---|---|
| Entry point | `@SpringBootApplication` | Main class FQCN, package structure |
| REST controllers | `@RestController`, `@Controller` | All `@GetMapping`, `@PostMapping`, `@PutMapping`, `@PatchMapping`, `@DeleteMapping` — path, method, return type |
| Service beans | `@Service` | Service class names and key methods |
| Repository beans | `@Repository`, `JpaRepository`, `CrudRepository` | Repository interfaces, entity types |
| Scheduled jobs | `@Scheduled` | Cron expressions, method names |
| Event listeners | `@EventListener`, `@KafkaListener`, `@RabbitListener`, `@SqsListener` | Event types, handler methods |
| Feature flags | `@ConditionalOnProperty` | Property keys that gate features |

### Integration Patterns

| Signal | Class / Annotation | What Agent Extracts |
|---|---|---|
| Feign clients | `@FeignClient` | Target service name, URL/URL placeholder, all methods mapped |
| RestTemplate calls | `RestTemplate.getForObject()`, `postForEntity()`, etc. | Target URLs or URI templates |
| WebClient calls | `WebClient.get()`, `.post()`, etc. | Target base URLs |
| Kafka producers | `KafkaTemplate.send()` | Topic names |
| Pub/Sub | `PubSubTemplate`, `@PubSubListener` | Topic and subscription names |
| RabbitMQ | `RabbitTemplate`, `@RabbitListener` | Exchange and queue names |

### Configuration

| Signal | File(s) | What Agent Extracts |
|---|---|---|
| App config | `application.yml`, `application.properties` | All non-secret keys; spring.datasource, spring.jpa, server.port, management.* |
| Profile configs | `application-{profile}.yml` | Profile names, profile-specific overrides |
| Secret references | `${SECRET_NAME}` or `${env.VAR}` patterns | Variable names only — values are never extracted |

### Data Layer

| Signal | File / Class | What Agent Extracts |
|---|---|---|
| Flyway | `src/main/resources/db/migration/V*.sql` | Migration file list, latest version number |
| Liquibase | `db/changelog/*.xml` or `*.yaml` | Changeset list, latest version |
| JPA entities | `@Entity` | Entity class names, table names if `@Table` present |
| Spring Data | `JpaRepository<Entity, ID>` | Entity type, ID type, custom query methods |

### Observability

| Signal | Dependency / Endpoint | What Agent Extracts |
|---|---|---|
| Actuator | `spring-boot-starter-actuator` in pom/gradle | Probes `/actuator/info`, `/actuator/health`, `/actuator/env` (safe endpoints only) |
| OpenAPI | `springdoc-openapi-starter-webmvc-ui` | Fetches `/v3/api-docs` if service is running |
| Micrometer | `micrometer-core`, `micrometer-registry-*` | Metrics backend (Prometheus, Datadog, etc.) |
| Logging | `logback-spring.xml`, `log4j2.xml` | Log format, appender types |

### Tests

| Signal | Directory / Annotation | What Agent Extracts |
|---|---|---|
| Unit tests | `src/test/java/**/*Test.java` | Count, framework (JUnit 5, TestNG) |
| Integration tests | `@SpringBootTest` | Count, profile used |
| Contract tests | `@PactTest`, `spring-cloud-contract` | Count, pact files |
| Coverage | `jacoco-maven-plugin`, `.jacoco.exec` | Target coverage %, actual if report exists |
| Test containers | `testcontainers` dependency | Which containers used (Postgres, Kafka, etc.) |

---

## Harness Integration

### Triggering the Discovery Pipeline from Harness

The PDLC framework includes a Harness pipeline template for running all three onboarding agents in sequence. Use this for automated or scheduled discovery runs.

#### Pipeline YAML Snippet

```yaml
pipeline:
  name: PDLC Onboarding Discovery
  identifier: pdlc_onboarding_discovery
  projectIdentifier: your_project
  orgIdentifier: your_org
  tags:
    framework: pdlc
    phase: onboarding

  stages:
    - stage:
        name: Discovery Agent
        identifier: discovery_agent
        type: CI
        spec:
          cloneCodebase: true
          platform:
            os: Linux
            arch: Amd64
          runtime:
            type: Cloud
            spec: {}
          execution:
            steps:
              - step:
                  type: Run
                  name: Run Discovery Agent
                  identifier: run_discovery_agent
                  spec:
                    shell: Bash
                    command: |
                      echo "Running PDLC Discovery Agent for ${PRODUCT_NAME}"
                      claude --no-interactive \
                        --system "$(cat onboarding/prompts/discovery-prompt.md)" \
                        --message "PRODUCT_NAME=${PRODUCT_NAME} REPO_PATH=${REPO_PATH} SERVICE_TYPE=${SERVICE_TYPE} TECH_STACK_HINT=${TECH_STACK_HINT}" \
                        --output products/${PRODUCT_NAME}/discovery/discovery-report.md
                    envVariables:
                      ANTHROPIC_API_KEY: <+secrets.getValue("anthropic_api_key")>
                      PRODUCT_NAME: <+pipeline.variables.PRODUCT_NAME>
                      REPO_PATH: <+pipeline.variables.REPO_PATH>
                      SERVICE_TYPE: <+pipeline.variables.SERVICE_TYPE>
                      TECH_STACK_HINT: <+pipeline.variables.TECH_STACK_HINT>

    - stage:
        name: Doc Agent
        identifier: doc_agent
        type: CI
        dependsOn:
          - discovery_agent
        spec:
          cloneCodebase: true
          platform:
            os: Linux
            arch: Amd64
          runtime:
            type: Cloud
            spec: {}
          execution:
            steps:
              - step:
                  type: Run
                  name: Run Doc Ingest Agent
                  identifier: run_doc_agent
                  spec:
                    shell: Bash
                    command: |
                      echo "Running PDLC Doc Agent for ${PRODUCT_NAME}"
                      claude --no-interactive \
                        --system "$(cat onboarding/prompts/doc-ingest-prompt.md)" \
                        --message "PRODUCT_NAME=${PRODUCT_NAME} DOC_SOURCES=${DOC_SOURCES}" \
                        --output products/${PRODUCT_NAME}/discovery/domain-glossary.md
                    envVariables:
                      ANTHROPIC_API_KEY: <+secrets.getValue("anthropic_api_key")>
                      CONFLUENCE_TOKEN: <+secrets.getValue("confluence_token")>
                      PRODUCT_NAME: <+pipeline.variables.PRODUCT_NAME>
                      DOC_SOURCES: <+pipeline.variables.DOC_SOURCES>

    - stage:
        name: Dependency Agent
        identifier: dependency_agent
        type: CI
        dependsOn:
          - discovery_agent
        spec:
          cloneCodebase: true
          platform:
            os: Linux
            arch: Amd64
          runtime:
            type: Cloud
            spec: {}
          execution:
            steps:
              - step:
                  type: Run
                  name: Run Dependency Map Agent
                  identifier: run_dep_agent
                  spec:
                    shell: Bash
                    command: |
                      echo "Running PDLC Dependency Agent for ${PRODUCT_NAME}"
                      claude --no-interactive \
                        --system "$(cat onboarding/prompts/dep-map-prompt.md)" \
                        --message "PRODUCT_NAME=${PRODUCT_NAME} DISCOVERY_REPORT=products/${PRODUCT_NAME}/discovery/discovery-report.md K8S_NAMESPACE=${K8S_NAMESPACE}" \
                        --output-dir products/${PRODUCT_NAME}/discovery/
                    envVariables:
                      ANTHROPIC_API_KEY: <+secrets.getValue("anthropic_api_key")>
                      PRODUCT_NAME: <+pipeline.variables.PRODUCT_NAME>
                      K8S_NAMESPACE: <+pipeline.variables.K8S_NAMESPACE>

  variables:
    - name: PRODUCT_NAME
      type: String
      description: "Product name (kebab-case)"
      required: true
    - name: REPO_PATH
      type: String
      description: "Absolute path to the cloned product repository on the delegate"
      required: true
    - name: SERVICE_TYPE
      type: String
      description: "api | microservice | monolith | data-pipeline"
      default: api
    - name: TECH_STACK_HINT
      type: String
      description: "Known tech stack hint, e.g. Java 21, Spring Boot 3.2, Maven"
      default: "Java 21, Spring Boot 3.x, Maven"
    - name: DOC_SOURCES
      type: String
      description: "Comma-separated list of documentation source URLs or paths"
      default: ""
    - name: K8S_NAMESPACE
      type: String
      description: "Kubernetes namespace of the target service (or 'none')"
      default: "none"
```

#### Environment Variables Required by the Pipeline

| Variable | Source | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` | Harness Secret | Claude Code API authentication |
| `CONFLUENCE_TOKEN` | Harness Secret | Confluence REST API access (Doc Agent) |
| `PRODUCT_NAME` | Pipeline Input | Target product name (kebab-case) |
| `REPO_PATH` | Pipeline Input | Path to cloned repo on Harness Delegate |
| `SERVICE_TYPE` | Pipeline Input | Service type classification |
| `TECH_STACK_HINT` | Pipeline Input | Known tech stack for context seeding |
| `DOC_SOURCES` | Pipeline Input | Doc URLs / paths for Doc Agent |
| `K8S_NAMESPACE` | Pipeline Input | Kubernetes namespace for Dependency Agent |

#### Harness Delegate Requirements

The Harness Delegate executing this pipeline must have:
- `git` installed and configured with access to the product repository
- `claude` CLI installed (Anthropic Claude Code)
- `kubectl` installed and configured if `K8S_NAMESPACE` is not `none`
- Network access to Confluence if Doc Agent is running
- Network access to the product service's running environment for Actuator probing (optional)

#### Delegate Tag

Tag your delegate with `pdlc-onboarding` and add the tag selector to the pipeline stage specs:

```yaml
  infrastructure:
    type: KubernetesDirect
    spec:
      connectorRef: your_k8s_connector
      namespace: harness-delegates
      nodeSelector:
        role: pdlc-agent
```
