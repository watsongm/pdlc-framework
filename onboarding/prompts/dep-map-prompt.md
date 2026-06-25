# Dependency Map Agent — Prompt Template

> **Used by:** Dependency Agent (Claude Code)
>
> **Phase:** Onboarding
>
> **Agent definition:** [`framework/agents/onboarding/dependency-agent.md`](../../framework/agents/onboarding/dependency-agent.md)
>
> **Output templates:**
> - [`templates/architecture-map.md`](../templates/architecture-map.md)
> - [`templates/interface-inventory.md`](../templates/interface-inventory.md)
>
> **Prerequisite:** `discovery-report.md` must exist (run Discovery Agent first)

---

## Context Injection Block

Load the following before running this prompt. All are **required** unless marked optional.

| Context Item | File / Source | Required? | Notes |
|---|---|---|---|
| Product CLAUDE.md | `products/{{PRODUCT_NAME}}/CLAUDE.md` | **Required** | Primary product context |
| Discovery report | `products/{{PRODUCT_NAME}}/discovery/discovery-report.md` | **Required** | Must exist before this agent runs |
| Architecture map template | `onboarding/templates/architecture-map.md` | **Required** | Primary output template |
| Interface inventory template | `onboarding/templates/interface-inventory.md` | **Required** | Secondary output template |
| Framework ADRs | `framework/adr/*.md` | Required | Architectural constraints |
| Kubernetes manifests | `{{K8S_MANIFEST_PATH}}` (if available) | Optional | e.g., `k8s/`, `helm/`, GitOps repo |
| Helm charts | `{{HELM_CHART_PATH}}` (if available) | Optional | |
| Terraform / IaC | `{{IAC_PATH}}` (if available) | Optional | GCP or VCF infrastructure definitions |
| Ansible inventory | `{{ANSIBLE_INVENTORY_PATH}}` (if available) | Optional | VCF VM topology |

---

## System Prompt

> _Copy this text verbatim as the system prompt when invoking the Dependency Agent._

```
You are a dependency mapping and architecture analysis specialist for the AI-Native 
PDLC Framework.

Your role is to build a complete, accurate map of how a software product connects to 
and depends on other systems — services, databases, message brokers, external APIs, 
and infrastructure. You work from a completed discovery-report.md as your primary 
input, supplemented by Kubernetes manifests, Helm charts, and infrastructure code 
where available.

You produce two documents:
1. architecture-map.md — service topology, dependency matrix, blast radius, 
   deployment map, data flows
2. interface-inventory.md — complete catalogue of all interfaces (inbound and 
   outbound, sync and async, file-based)

## Core Constraints

1. READ-ONLY: You must NEVER modify any file in the target repository or 
   infrastructure. Analysis only.

2. EVIDENCE-BASED: Every entry in the dependency matrix, topology diagram, and 
   interface inventory must cite its source (file path, line number, or Kubernetes 
   resource name). Do not infer connections that have no evidence.

3. FLAG ARCHITECTURAL CONCERNS: Proactively identify and flag:
   - Circular dependencies between services
   - Shared databases with write access from multiple services (major concern)
   - Synchronous dependency chains > 3 hops deep
   - External dependencies with no circuit breaker or fallback
   - Single points of failure with no redundancy
   - More than 20 distinct external dependencies (complexity concern)

4. DISTINGUISH SYNC vs ASYNC: Always specify whether a dependency is synchronous 
   (HTTP, JDBC, gRPC — blocking) or asynchronous (Kafka, Pub/Sub, RabbitMQ — 
   non-blocking). This matters for blast radius analysis.

5. GCP AWARENESS: Recognise GCP-specific infrastructure: Cloud Pub/Sub, Cloud SQL, 
   Firestore, GCS, Cloud Run, GKE, Cloud Spanner, Memorystore, API Gateway, 
   Cloud Endpoints, IAP. When detected, note the GCP service name and project.

6. VMware VCF AWARENESS: Recognise VM-based workloads from Ansible inventory, 
   vSphere tags, or infrastructure naming conventions. Note if a service is 
   VM-deployed rather than containerised — this affects deployment strategy and 
   blast radius differently.

7. MERMAID QUALITY: Your Mermaid topology diagram must be syntactically valid. 
   Test it mentally: no unclosed brackets, no invalid edge types, no duplicate 
   node IDs. Label every edge with protocol and direction.

8. BLAST RADIUS REALISM: Blast radius analysis must reflect REAL propagation — 
   consider retry storms, cache invalidation cascades, and database connection 
   pool exhaustion when a component fails, not just direct callers.
```

---

## Task Prompt

> _Replace all `{{PLACEHOLDER}}` values before sending._

```
## Dependency Mapping Task

Build a complete dependency and architecture map for the following product.

### Parameters

- **Product Name:** {{PRODUCT_NAME}}
- **Discovery Report Path:** {{DISCOVERY_REPORT_PATH}}
- **Repository Path:** {{REPO_PATH}}
- **Kubernetes Namespace:** {{K8S_NAMESPACE}}
  ("none" if not deployed on Kubernetes)
- **Service Registry:** {{SERVICE_REGISTRY}}
  (gcp-service-directory | consul | eureka | none | unknown)
- **Kubernetes Manifest Path:** {{K8S_MANIFEST_PATH}}
  (path to k8s/, helm/, or gitops repo — or "none")
- **Ansible Inventory Path:** {{ANSIBLE_INVENTORY_PATH}}
  (path to Ansible inventory for VCF VM topology — or "none")
- **GCP Project ID:** {{GCP_PROJECT_ID}}
  (if deploying on GCP — or "unknown")
- **VCF vSphere Cluster:** {{VCF_CLUSTER}}
  (if deploying on VMware VCF — or "unknown")

### Mapping Instructions

Perform the following steps in order.

#### Step 1: Ingest Discovery Report

1. Read the discovery report at `{{DISCOVERY_REPORT_PATH}}` in full.
2. Extract and note:
   - All FeignClient targets (service names and URLs)
   - All RestTemplate / WebClient base URLs
   - All database connections (host/database from datasource config)
   - All Kafka/Pub/Sub/RabbitMQ topic/queue references (producer and consumer)
   - All third-party API integrations
   - All `@KafkaListener`, `@RabbitListener`, `@PubSubListener` consumers
   - All `@RestController` endpoints (inbound)
   - All `@Scheduled` jobs and `@EventListener` handlers
3. This forms your raw dependency inventory — the foundation for all subsequent steps.

#### Step 2: FeignClient Mapping

1. For each `@FeignClient` found in the discovery report:
   a. Extract the `name` attribute — this is the logical service identifier.
   b. Extract the `url` attribute or `${config.key}` placeholder — resolve the config 
      key against `application.yml` to find the actual URL pattern.
   c. Extract all interface methods — record the HTTP method, path, and parameter 
      types. These are outbound REST dependencies.
   d. Check for `fallback` or `fallbackFactory` attribute — note if a fallback is 
      configured.
   e. Check if `Resilience4j` or `spring-cloud-circuitbreaker` is on the classpath 
      (from discovery report dependencies) — note if circuit breaker is present.
2. Map each FeignClient to an entry in the dependency matrix (Section 3.1 of 
   architecture-map.md) and the interface inventory (Section 5 for third-party, 
   Section 3.1 for inbound REST).

#### Step 3: RestTemplate / WebClient Mapping

1. For each RestTemplate usage found:
   a. Extract the URL pattern from `.getForObject(url, ...)`, `.postForEntity(url, ...)`, 
      `.exchange(url, ...)` etc.
   b. Determine the target service from the URL (hostname, config placeholder, 
      or service name pattern).
   c. Note whether the call is synchronous (RestTemplate is always synchronous).
2. For each WebClient usage:
   a. Extract the base URL from `WebClient.create(url)` or `builder.baseUrl(url)`.
   b. Note whether error handling (`.onStatus()`) is present.
   c. Note whether the call is subscribed to (reactive) or blocked (`.block()`).
3. Add all to dependency matrix (Section 3.1, architecture-map.md).

#### Step 4: Spring Data Repository Mapping

1. For each `JpaRepository` / `MongoRepository` / `CrudRepository` found in the 
   discovery report:
   a. Map it to its database (inferred from datasource config — Spring Data JPA uses 
      the configured datasource).
   b. Record the entity type and what tables it maps to.
   c. Note custom query methods (`@Query`) — these reveal data access patterns.
2. If multiple services share the same datasource URL (from their discovery reports, 
   if available for other services):
   - Immediately output: `[ESCALATE: SHARED DATABASE — <db-name> accessed by 
     <service-1> and <service-2>]`
   - Record in Section 10 (Architecture Concerns) of architecture-map.md.
3. Populate Section 3.2 (Service-to-Data Dependencies) and Section 3 (Database 
   Interfaces) of interface-inventory.md.

#### Step 5: Message Broker Mapping

1. For each Kafka producer/consumer:
   a. Map topic names to producer and consumer services.
   b. Note the partition key strategy if discoverable.
   c. Check for dead-letter topic configuration.
   d. Note consumer group ID.
2. For each GCP Pub/Sub topic/subscription:
   a. Note the topic and subscription names.
   b. Note if push or pull subscription.
   c. Note the service account used (from config — redact if secret).
3. For each RabbitMQ exchange/queue:
   a. Map exchange name, routing key, and queue name.
   b. Note binding configurations.
4. Check for circular event patterns: Service A publishes to Topic X, which Service B 
   consumes, which publishes to Topic Y, which Service A consumes. If found:
   `[ESCALATE: CIRCULAR EVENT PATTERN — <describe the cycle>]`
5. Populate Sections 2.1, 2.2 (Async Interfaces) of interface-inventory.md and 
   Section 3.3 (Service-to-Broker Dependencies) of architecture-map.md.

#### Step 6: Kubernetes / Helm Manifest Analysis

If `{{K8S_MANIFEST_PATH}}` is not "none":

1. Read all `Deployment` resources — extract:
   - Service names, container images, replica counts
   - Environment variable names (not values if they reference secrets)
   - Resource limits and requests
   - Liveness and readiness probe paths
2. Read all `Service` resources — extract:
   - Service type (ClusterIP / NodePort / LoadBalancer)
   - Port mappings
   - Selector labels (to match to Deployments)
3. Read all `Ingress` / `Gateway` resources — extract:
   - External hostnames
   - Path routing rules
4. Read `ConfigMap` resources — extract config keys (not secret values).
5. If Helm charts present: read `values.yaml` for default configuration.
6. If a GKE `BackendConfig` or `ManagedCertificate` is present — note GCP-managed 
   load balancing and TLS configuration.
7. Populate Section 7.1 (GCP GKE Components) of architecture-map.md.

#### Step 7: Ansible / VCF VM Topology Analysis

If `{{ANSIBLE_INVENTORY_PATH}}` is not "none":

1. Read the Ansible inventory file (INI or YAML format).
2. Extract host groups and host names — these represent VM names or patterns.
3. Look for host variables (`host_vars/`) that define service roles, ports, or 
   vSphere tags.
4. If vSphere tags are referenced (e.g., `vsphere_cluster`, `vsphere_datastore`), 
   extract them.
5. Map host groups to service components.
6. Populate Section 7.2 (VMware VCF Components) of architecture-map.md.

If Ansible inventory is not available but `{{VCF_CLUSTER}}` is known:
- Note the cluster name and that VM-level topology is not determinable from 
  static analysis. Flag for human annotation.

#### Step 8: External Service Registry Query (if available)

If `{{SERVICE_REGISTRY}}` is "gcp-service-directory":
1. Note the GCP Service Directory namespace and region for `{{GCP_PROJECT_ID}}`.
2. List any known services registered — this supplements the dependency map.

If `{{SERVICE_REGISTRY}}` is "consul" or "eureka":
1. Note the registry URL (from application.yml discovery config).
2. List registered services if accessible.

If "none" or "unknown": skip this step.

#### Step 9: Dependency Graph and Circular Dependency Detection

1. From all data gathered, build a directed graph:
   - Nodes: all services, databases, brokers, external services
   - Edges: dependency direction (from consumer to provider)
2. Perform cycle detection on the graph:
   - If any cycle exists in synchronous (HTTP/JDBC) dependencies: 
     `[ESCALATE: CIRCULAR DEPENDENCY — <cycle path>]`
   - Note: async event cycles are a concern but less critical — flag as 
     `[CONCERN: CIRCULAR EVENT PATTERN]`
3. Count total external dependencies:
   - If count > 20: `[ESCALATE: LARGE DEPENDENCY GRAPH — N external dependencies]`
4. Identify longest synchronous dependency chain (A → B → C → D):
   - If chain length > 3: note as architectural concern AC-XXX.

#### Step 10: Mermaid Topology Diagram

1. Produce a syntactically valid Mermaid `graph LR` diagram based on the dependency 
   graph from Step 9.
2. Rules for the diagram:
   - Use descriptive but concise node labels (service name + type)
   - Label every edge with: protocol (REST/gRPC/JDBC/Kafka/Pub/Sub/etc.) and 
     direction arrow
   - Use subgraphs to group: internal services, data stores, message brokers, 
     external/third-party services, upstream callers
   - Apply colour classes: services=blue, databases=green, brokers=amber, 
     external=purple, callers=grey
   - Do not include more than 25 nodes — if graph is larger, produce a summary 
     diagram of top-level components only and note that detail is in the matrix
3. Place the diagram in Section 1 of architecture-map.md.

#### Step 11: Data Flow Documentation

1. Identify the 3–5 most important data flows through the system:
   - The primary "happy path" (main user-facing operation)
   - The most complex flow (most hops through the system)
   - Any async / background processing flow
   - Any data ingestion flow (for data pipelines)
2. For each flow, trace step-by-step: trigger → service → data store / external call 
   → response / event.
3. Note which data stores are touched and whether operations are transactional.
4. Populate Section 5 (Data Flow Descriptions) of architecture-map.md.

#### Step 12: Blast Radius Analysis

For each component (service, database, broker, external service):

1. Identify all services that synchronously depend on it (direct callers).
2. Identify all services that asynchronously depend on it (event consumers).
3. Determine failure mode: what happens if this component becomes unavailable?
   - Synchronous dependency → calling service fails with timeout/500
   - Database unavailable → service cannot serve requests that touch that data
   - Broker unavailable → producers cannot publish (may queue), consumers halt
4. Is there a fallback?
   - Circuit breaker configured? (Resilience4j, Spring Cloud Circuit Breaker)
   - Read replica available?
   - Cache fallback?
   - Graceful degradation (returns partial data or cached response)?
5. Estimate user impact (% of user journeys affected based on criticality).
6. Assign blast radius level: Critical / High / Medium / Low.
7. Populate Section 6 (Blast Radius Analysis) of architecture-map.md.

#### Step 13: External Dependency SLA and Criticality

1. For each external dependency identified:
   a. Look for SLA information in documentation (from domain-glossary.md if it exists).
   b. If not documented, note "SLA not documented."
   c. Assign criticality based on blast radius analysis.
2. Populate Section 8 (External Dependencies and SLA) of architecture-map.md.

#### Step 14: Architecture Pattern Identification

1. Assess the codebase structure for the following patterns:
   - **Layered (n-tier):** controller → service → repository layer separation?
   - **Hexagonal (Ports & Adapters):** domain package separate from adapter/port 
     packages? Application services depend on interfaces, not implementations?
   - **Event-Driven:** significant use of async messaging for business operations?
   - **CQRS:** separate read and write models? Different repositories for queries 
     vs commands?
   - **Saga:** distributed transactions via events? Compensating transactions present?
   - **Circuit Breaker:** Resilience4j annotations or configuration present?
   - **API Gateway:** service sits behind a gateway? Gateway config in k8s ingress?
   - **Outbox Pattern:** transactional outbox table in DB migration files?
2. Provide evidence (file path / class name) for each pattern identified.
3. Populate Section 9 (Architecture Patterns Identified) of architecture-map.md.

#### Step 15: Architecture Concerns Documentation

1. Consolidate all concerns identified throughout the analysis:
   - Circular dependencies
   - Shared databases
   - Long sync chains
   - Missing circuit breakers
   - Missing dead-letter queues
   - Single points of failure
   - No API versioning
   - Hardcoded URLs
   - Missing distributed tracing
2. Assign severity (Critical / High / Medium / Low) and provide a recommendation.
3. Populate Section 10 (Architecture Concerns) of architecture-map.md.

#### Step 16: Interface Inventory Completion

Using all data gathered, complete interface-inventory.md:
- Section 1: REST endpoints (from discovery report controllers)
- Section 2: Async interfaces (from broker analysis)
- Section 3: Database interfaces (from repository mapping)
- Section 4: SDK/library interfaces (from discovery report dependencies — internal only)
- Section 5: External third-party (from FeignClient / WebClient / external API analysis)
- Section 6: Event streams (from broker analysis)
- Section 7: File/batch interfaces (from @Scheduled jobs and batch config)
- Section 8: Interface concerns (from all findings)

### Output

Produce two completed documents:

1. `products/{{PRODUCT_NAME}}/discovery/architecture-map.md`
2. `products/{{PRODUCT_NAME}}/discovery/interface-inventory.md`

At the end, output:
```
DEPENDENCY MAPPING COMPLETE — {{PRODUCT_NAME}} — [timestamp]
Components mapped: [N] | Dependencies: [N] | Escalations: [N] | Concerns: [N]
```
```

---

## Expected Output Format

Two files produced in `products/{{PRODUCT_NAME}}/discovery/`:

| File | Template | Sections |
|---|---|---|
| `architecture-map.md` | `onboarding/templates/architecture-map.md` | 11 sections + human review |
| `interface-inventory.md` | `onboarding/templates/interface-inventory.md` | 8 sections + human review |

Both files must:
- Have all `{{PLACEHOLDER}}` values replaced with real findings or `"None found."`
- Include file/line citations for all material findings
- Include a syntactically valid Mermaid diagram in architecture-map.md Section 1
- Include all escalation flags raised during analysis

---

## Validation Checklist

```
[ ] architecture-map.md exists at products/{{PRODUCT_NAME}}/discovery/
[ ] interface-inventory.md exists at products/{{PRODUCT_NAME}}/discovery/
[ ] Mermaid diagram in architecture-map.md is syntactically valid
[ ] Dependency matrix (3.1) covers all FeignClient / RestTemplate / WebClient calls
[ ] Data dependencies (3.2) cover all repository-to-database mappings
[ ] Broker dependencies (3.3) cover all Kafka/Pub/Sub/RabbitMQ usages
[ ] Blast radius table covers all components (service, DB, broker, external)
[ ] GKE components section populated (or explicitly "None — not GKE deployed")
[ ] VCF components section populated (or explicitly "None — not VCF deployed")
[ ] Architecture concerns documented
[ ] interface-inventory.md REST endpoints match discovery-report.md controllers
[ ] No {{PLACEHOLDER}} values remain
[ ] No target repo files were modified
[ ] Mapping ends with DEPENDENCY MAPPING COMPLETE line
```

---

## Escalation Triggers

| Trigger | Flag | Required Action |
|---|---|---|
| Circular synchronous dependency detected | `[ESCALATE: CIRCULAR DEPENDENCY]` | Must be resolved before Design Phase — architectural blocker |
| Shared database with write access from multiple services | `[ESCALATE: SHARED DATABASE]` | Major architectural concern — Design Phase must address data ownership |
| >20 distinct external dependencies | `[ESCALATE: LARGE DEPENDENCY GRAPH]` | Complexity risk — Design Phase must assess consolidation opportunities |
| Synchronous dependency chain >3 hops | `[CONCERN: LONG SYNC CHAIN]` | Architectural concern — flag for Design Phase |
| No circuit breaker on critical external sync call | `[CONCERN: NO CIRCUIT BREAKER]` | Resilience risk — flag for Design Phase |
| Circular event pattern detected (async) | `[CONCERN: CIRCULAR EVENT PATTERN]` | Review in Design Phase — potential infinite loop risk |

---

## GCP-Specific Notes

When the product uses GCP services, note the following in the relevant sections:

| GCP Service | Detection Signal | Note in Output |
|---|---|---|
| Cloud Pub/Sub | `com.google.cloud.spring.pubsub` dependency, `PubSubTemplate` usage | Note topic/subscription names and GCP project |
| Cloud SQL | `com.google.cloud.sql.postgres.SocketFactory` in datasource config | Note instance connection name |
| Firestore | `com.google.cloud.firestore` imports | Note collection names accessed |
| GCS | `com.google.cloud.storage` imports, `spring-cloud-gcp-starter-storage` | Note bucket names |
| Cloud Spanner | `com.google.cloud.spanner` imports | Note instance and database |
| Memorystore (Redis) | `spring.data.redis` config pointing to internal IP | Note as likely Memorystore |
| GKE Workload Identity | `iam.gke.io/gcp-service-account` annotation in k8s manifests | Note SA and permissions |

---

## VMware VCF-Specific Notes

When the product runs on VMware VCF (VM-based, not containerised):

| VCF Signal | Detection Method | Note in Output |
|---|---|---|
| VM hostname pattern | Ansible inventory host names, vSphere tag naming conventions | Record in Section 7.2 |
| NSX-T network segment | Ansible variables, Terraform VCF resources | Note network segment name |
| vSphere cluster | Ansible `vsphere_cluster` variable or Terraform `vsphere_compute_cluster` | Note cluster assignment |
| Resource pool | Ansible host variables, Terraform resource pool | Note resource pool |
| Non-containerised deployment | Absence of Dockerfile, presence of Ansible playbooks for JAR deployment | Flag that service is VM-deployed |

---

## Alternative Backend Note

This prompt is designed for **Claude Code** with filesystem MCP for repository access.

### LangGraph

- `tool_node` per step: `ReadFileTool`, `RegexSearchTool` (ripgrep), `KubeAPITool`
- Graph: `StateGraph` with dependency accumulator state shared across nodes
- Step 9 (cycle detection) → Python `networkx` as a tool function
- Step 10 (Mermaid) → string-templating tool with Mermaid syntax validation
- Escalations → `conditional_edges` routing to human-review interrupt before continuing

### Google ADK

- `DependencyMapAgent` with `FunctionTool` wrappers per discovery action
- Build the dependency graph as a structured `DependencyGraph` dataclass
- Cycle detection as a pure Python function (no external tool needed)
- Output serialisation: `DependencyGraph` → two markdown files via file-writing tools
- Sub-agents: `KubernetesAnalysisAgent` for k8s manifests, `VCFAnalysisAgent` for Ansible
