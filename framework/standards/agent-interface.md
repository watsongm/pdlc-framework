# Agent Interface Standard

This document defines the **Agent Interface Contract** — the standard that decouples agent role definitions from the underlying orchestration backend. Every agent in this framework must conform to this contract, regardless of whether it runs on Claude Code, LangGraph, Google ADK, CrewAI, or another system.

---

## Why Loose Coupling Matters

The AI agent tooling landscape is evolving rapidly. Committing exclusively to one backend creates lock-in that is expensive to reverse. This framework treats the agent orchestration backend as an **infrastructure concern** — swappable beneath a stable contract layer — similar to how a database abstraction layer decouples business logic from a specific database engine.

**Default backend:** Claude Code (Anthropic) — optimised for terminal-native agentic coding, CLAUDE.md context injection, and MCP server integration.

**Supported alternative backends:** LangGraph, Google ADK (Java/Go), CrewAI, Microsoft Agent Framework, OpenAI Agents SDK.

---

## The Agent Contract

Every agent definition in `framework/agents/**/*.md` must declare all of the following fields. These fields are **backend-agnostic** and serve as the stable interface.

```yaml
# Agent Contract Schema
agent_id: string               # Unique identifier (e.g., "discovery-agent")
role: string                   # Human-readable role name
phase: string                  # onboarding | spec | build | ops
autonomy_tier: HITL | HOTL | HOOTL

inputs:
  - name: string
    type: file | directory | url | structured-data
    required: boolean
    description: string

outputs:
  - name: string
    type: file | structured-data
    description: string

escalation_triggers:
  - condition: string          # Plain-English description of what triggers escalation
    action: halt | notify | request-approval

constraints:
  - string                     # Hard constraints the agent must never violate

system_prompt: |               # The actual system prompt text (backend-agnostic)
  ...

context_package:               # Files/data always injected into agent context
  - path: string
    description: string
```

---

## Context Injection Standard

All agents — regardless of backend — receive the same **core context package** at the start of every run. Additional context is declared in the agent contract's `context_package` field.

### Core Context Package (Always Injected)

| File | Purpose |
|---|---|
| `products/<name>/CLAUDE.md` | Product overview, architecture summary, active ADR list, agent instructions |
| `products/<name>/adrs/*.md` | All accepted ADRs for this product |
| `framework/standards/runtime-standards.md` (relevant section) | Runtime-specific coding standards |
| `products/<name>/spec/spec.md` (if exists) | Current approved specification |
| `products/<name>/discovery/domain-glossary.md` (if exists) | Domain terminology |

### Context Injection Rules
1. **Always load ADRs before generating code** — no exceptions
2. **Prefer CLAUDE.md summaries over raw file dumps** — curate context, don't flood the window
3. **Load only the spec sections relevant to the current task** — use the task's `Spec Reference` field
4. **Never load secrets or credentials** — use environment variable references only

---

## Backend Adapters

### Claude Code Adapter (Default)

Claude Code is the default backend. It uses:
- **CLAUDE.md** as the primary context file (loaded automatically by Claude Code when present in the project root or product directory)
- **Slash commands** (`/onboard`, `/spec`, `/build`, `/review`) as task triggers
- **MCP servers** for structured tool access (filesystem, Confluence, GitHub, etc.)
- **Subagent spawning** for parallel tasks within a phase

**Mapping Agent Contract → Claude Code:**

| Contract Field | Claude Code Implementation |
|---|---|
| `system_prompt` | Prepended to CLAUDE.md under `## Agent Instructions` section |
| `context_package` | Listed under `## Context Files` in CLAUDE.md; Claude Code loads them |
| `inputs` | Provided as file paths in the task invocation prompt |
| `outputs` | Agent writes files to specified output paths |
| `escalation_triggers` | Agent outputs a structured `ESCALATION` block and halts |
| `autonomy_tier: HITL` | Agent always outputs proposed action + rationale; awaits explicit approval |
| `autonomy_tier: HOTL` | Agent proceeds; logs actions for human review |
| `autonomy_tier: HOOTL` | Agent proceeds; writes audit log entry |

**Claude Code invocation pattern:**
```bash
# In the product directory, with CLAUDE.md present:
claude "Run the discovery agent for this Java/Spring Boot service.
Repo: /path/to/service
Service type: microservice
Output: products/my-service/discovery/"
```

**MCP servers used across agents:**

| Agent | MCP Servers |
|---|---|
| Discovery Agent | `filesystem`, `git` (optional), `confluence` (optional) |
| Doc Agent | `confluence`, `browser`, `filesystem` |
| Dependency Agent | `filesystem`, `kubernetes` (optional), `gcp` (optional) |
| Dev Agent | `filesystem`, `git`, `github` |
| Deploy Agent | `harness`, `kubernetes`, `gcp` |
| Monitor Agent | `gcp-monitoring`, `gcp-logging`, `harness` |

**`agent-config.yaml` for Claude Code:**
```yaml
backend: claude-code
version: "1.0"
products:
  my-service:
    context_root: products/my-service/
    claude_md: products/my-service/CLAUDE.md
    mcp_servers:
      - filesystem
      - git
      - confluence
    autonomy_overrides: {}  # Per-product HITL overrides if needed
```

---

### LangGraph Adapter

LangGraph (Python, from LangChain) maps the Agent Contract to a **StateGraph** with typed state, tool nodes, and conditional edges.

**Mapping:**

| Contract Field | LangGraph Implementation |
|---|---|
| `system_prompt` | `SystemMessage` in the agent node's prompt |
| `inputs` | Typed fields in the `AgentState` TypedDict |
| `outputs` | Written to `AgentState` output fields |
| `escalation_triggers` | Conditional edge: if trigger condition met → route to `human_review` node |
| `autonomy_tier: HITL` | `interrupt_before=["action_node"]` in graph compilation |
| `autonomy_tier: HOTL` | No interrupt; `human_review` node with async notification |
| `autonomy_tier: HOOTL` | No interrupt; audit log tool call in node |
| `context_package` | Loaded into `AgentState.context` at graph entry |

**Minimal LangGraph skeleton:**
```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from typing import TypedDict, Annotated

class AgentState(TypedDict):
    product_id: str
    context: dict           # Core context package
    task: dict              # Current task from tasks.md
    outputs: list[str]      # File paths written
    escalation: str | None  # Escalation message if triggered

def agent_node(state: AgentState) -> AgentState:
    # Load system prompt from agent contract
    # Execute agent logic
    # Write outputs
    # Check escalation triggers
    return state

builder = StateGraph(AgentState)
builder.add_node("agent", agent_node)
builder.add_node("human_review", human_review_node)
builder.add_conditional_edges("agent", route_on_escalation)
graph = builder.compile(checkpointer=MemorySaver(),
                        interrupt_before=["action_node"])  # For HITL
```

**Context injection for LangGraph:**
```python
def load_context_package(product_id: str) -> dict:
    """Load the standard context package for a product."""
    return {
        "claude_md": read_file(f"products/{product_id}/CLAUDE.md"),
        "adrs": load_all_adrs(f"products/{product_id}/adrs/"),
        "spec": read_file(f"products/{product_id}/spec/spec.md"),
        "runtime_standards": get_runtime_section("java-springboot"),
        "domain_glossary": read_file(f"products/{product_id}/discovery/domain-glossary.md"),
    }
```

---

### Google ADK Adapter

Google Agent Development Kit (ADK) targets GCP-native deployments. It is opinionated and batteries-included, best suited when the full stack is GCP.

**Mapping:**

| Contract Field | Google ADK Implementation |
|---|---|
| `system_prompt` | `Agent(instruction=...)` parameter |
| `inputs` | Passed as session state or tool arguments |
| `outputs` | Written via `FunctionTool` file write operations |
| `escalation_triggers` | Custom `EscalationTool` that transfers control to a `HumanReviewAgent` |
| `autonomy_tier` | Modelled as sub-agent routing in a multi-agent `SequentialAgent` or `ParallelAgent` |
| `context_package` | Loaded into session state at agent initialisation |

**Minimal ADK skeleton (Python):**
```python
from google.adk.agents import Agent, SequentialAgent
from google.adk.tools import FunctionTool

discovery_agent = Agent(
    name="discovery_agent",
    model="gemini-2.0-flash",
    instruction="""[System prompt from agent contract]""",
    tools=[filesystem_tool, git_tool],
)

# HITL via human review sub-agent
human_review_agent = Agent(
    name="human_review",
    instruction="Present findings to human operator and await approval.",
)

pipeline = SequentialAgent(
    name="onboarding_pipeline",
    sub_agents=[discovery_agent, human_review_agent],
)
```

---

### CrewAI Adapter

CrewAI uses role-based crews with task delegation. Maps naturally to this framework's agent-per-role design.

**Mapping:**

| Contract Field | CrewAI Implementation |
|---|---|
| `system_prompt` | `Agent(role=..., goal=..., backstory=...)` |
| `inputs` | `Task(description=..., context=[...])` |
| `outputs` | `Task(expected_output=...)` + file write tools |
| `escalation_triggers` | Custom tool that raises `HumanInputRequired` |
| `autonomy_tier: HITL` | `human_input=True` on the Task |
| `autonomy_tier: HOTL/HOOTL` | `human_input=False`; async notification tool |

**Minimal CrewAI skeleton:**
```python
from crewai import Agent, Task, Crew

discovery_agent = Agent(
    role="Codebase Discovery Specialist",
    goal="Produce a complete discovery report for a Java/Spring Boot service",
    backstory="[From agent contract system_prompt]",
    tools=[filesystem_tool, git_tool],
    verbose=True,
)

discovery_task = Task(
    description="Analyse the repository at {repo_path} and produce discovery-report.md",
    expected_output="Completed discovery-report.md written to products/{product_id}/discovery/",
    agent=discovery_agent,
    human_input=False,  # HOTL — human reviews output, not each step
)

crew = Crew(agents=[discovery_agent], tasks=[discovery_task])
```

---

## Escalation Message Format

When any agent triggers an escalation (regardless of backend), it must produce a structured escalation block:

```
## ESCALATION REQUIRED

**Agent:** {agent_id}
**Product:** {product_id}
**Phase:** {phase}
**Trigger:** {escalation_trigger_condition}
**Urgency:** P1 | P2 | P3

**Context:**
{What the agent was doing when the escalation was triggered}

**Finding:**
{What the agent found that requires human judgment}

**Options:**
1. {Option A — with consequences}
2. {Option B — with consequences}

**Recommendation:**
{Agent's recommended option, if any}

**Agent State:**
{Summary of work completed so far; safe to resume from this point}

**Action Required:**
{Exactly what the human needs to do to unblock the agent}
```

---

## Switching Backends

To switch a product from Claude Code to LangGraph:

1. Update `agent-config.yaml` → set `backend: langgraph`
2. Implement the LangGraph adapter mappings for each agent used by that product
3. Verify context injection produces equivalent context to the CLAUDE.md pattern
4. Run all agents against the staging environment and compare outputs
5. Write an ADR documenting the switch and its rationale
6. Update the product's `CLAUDE.md` with a note that LangGraph is the active backend

To switch at the framework level (all products), update `framework/standards/agent-config.yaml` default backend field and migrate all products via the above process.

---

*Last updated: 2026-06 | Owner: Framework Architect | Review cadence: Quarterly*
