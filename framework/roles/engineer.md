# Role: Engineer

> **Headcount:** 1–2 people | **Typical title:** Senior Engineer, Software Engineer | **Autonomy tier scope:** HOTL primary (monitors agent implementation), HITL for production-affecting code review decisions

---

## Role Purpose

The Engineer's role in an AI-native pod is fundamentally different from a traditional engineering role. The primary output is not code — it is **oversight, context quality, and prompt craftsmanship** that determines the quality of agent-generated code.

Engineers in this framework:
1. **Review agent-generated PRs** — not just for correctness, but for maintainability, safety, and spec alignment
2. **Engineer the context** that agents use — maintaining `CLAUDE.md`, curating reference files, writing effective prompts
3. **Operate as the first line of quality assurance** for everything agents produce before it reaches the Architect or Ops gates
4. **Intervene deliberately** when agent output requires human correction — and then improve the context so the same correction is not needed twice

Engineers who try to compete with agents on code output volume will fail and burn out. Engineers who treat context engineering as their primary craft will multiply team throughput by an order of magnitude.

---

## Day-to-Day Responsibilities

### Weekly Rhythm

**Monday — Task preparation:**
- Review `tasks.md` and verify each task has sufficient context linked (spec section, relevant ADRs, acceptance criteria)
- Identify any tasks where the agent is likely to need clarification and pre-populate context proactively
- Review any prompt templates that were flagged as producing poor output in the previous sprint

**Daily — PR review queue:**
- Review agent-generated PRs (HOTL for most; HITL for security-adjacent or data-model-affecting changes)
- Apply the PR review checklist (see below) — not just "does it work" but "will this be maintainable?"
- Merge or request revision; if revision, update the relevant prompt or context file so the same issue does not recur

**Daily — Context maintenance:**
- Update `CLAUDE.md` when PRs reveal context gaps (agents that asked the wrong question usually had missing context)
- Keep the domain `glossary.md` current with terminology introduced in that day's work
- Flag any ADR that agents are consistently getting wrong — this usually means the ADR needs clearer language or better injection placement

**Wednesday — Test quality review:**
- Review agent-generated test suites for quality (see Java/Spring Boot testing section)
- Confirm coverage targets are being met (`jacoco` report in CI)
- Check Testcontainers usage — integration tests must use real infrastructure, not mocks of infrastructure

**Friday — Prompt refinement:**
- Review which prompts produced the lowest-quality outputs this sprint
- Iterate on prompt templates using the structured improvement process (see prompt engineering section)
- Document prompt improvements in the agent definition files in `agents/`

---

## Java/Spring Boot Specific Responsibilities

### Reviewing Spring Boot Test Patterns

Agent-generated test code in Java/Spring Boot must be reviewed against the following standards:

#### Unit Tests (JUnit 5 + Mockito)

**Required patterns:**
```java
// Use @ExtendWith(MockitoExtension.class) — not @RunWith
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderService orderService;

    @Test
    @DisplayName("should return empty optional when order not found")
    void shouldReturnEmptyOptionalWhenOrderNotFound() {
        // Arrange
        when(orderRepository.findById(99L)).thenReturn(Optional.empty());

        // Act
        Optional<Order> result = orderService.findById(99L);

        // Assert
        assertThat(result).isEmpty();
        verify(orderRepository, times(1)).findById(99L);
    }
}
```

**Red flags to reject in agent-generated tests:**
- `@MockBean` in unit tests (belongs only in `@SpringBootTest` integration tests)
- Assertions using `assertTrue(result.equals(expected))` — use AssertJ `assertThat(result).isEqualTo(expected)`
- Missing `@DisplayName` on test methods
- Tests that only assert the mock was called, not the actual return value
- `Thread.sleep()` for async testing — use `Awaitility`
- Static state in test classes (shared mutable fields across test methods)

#### Integration Tests (Spring Boot Test + Testcontainers)

**Required patterns:**
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderControllerIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldCreateOrderSuccessfully() {
        // Real HTTP call to a real Spring context with a real PostgreSQL container
    }
}
```

**Red flags to reject in agent-generated integration tests:**
- `H2` in-memory database used for integration tests (Testcontainers with the real DB image is required)
- Mocking `@Repository` beans in `@SpringBootTest` — this defeats the purpose of integration tests
- Integration test class not annotated with `@Testcontainers`
- Missing `@DynamicPropertySource` — hardcoded ports will cause test flakiness
- Shared test state not cleaned up between tests (use `@BeforeEach` or `@Sql` scripts)

#### Slice Tests

The following slice test annotations are approved for use:
- `@WebMvcTest` — Controller layer only; mock service layer
- `@DataJpaTest` — Repository layer only; uses Testcontainers postgres, not H2
- `@JsonTest` — JSON serialisation/deserialisation testing

#### Test Coverage Requirements
- Unit test coverage: >80% line coverage per service class (enforced by JaCoCo in CI)
- Integration test coverage: all public REST endpoints must have at least one passing integration test
- Mutation testing: PIT (`pitest`) run on every PR; mutation score target >60% for service classes

### Reviewing Generated Code Quality

Beyond testing, Engineers review agent-generated Java code for:

**Readability:**
- Method length: flag any method >30 lines; request decomposition
- Class length: flag any class >300 lines; request decomposition or extraction
- Naming: Java naming conventions enforced; acronyms treated as words (`HttpUrl`, not `HTTPUrl`)
- Comments: no redundant comments ("// set the name" above `setName(name)`); only comments that explain *why*

**Error Handling:**
- No swallowed exceptions (`catch (Exception e) {}` without logging or re-throwing)
- Custom exception hierarchy (not `RuntimeException` thrown directly)
- ProblemDetail response for all REST errors (Spring 6 standard)
- Logging at correct levels: DEBUG for trace, INFO for business events, WARN for recoverable issues, ERROR for unrecoverable issues

**Performance:**
- No blocking calls in reactive pipelines (if WebFlux is in use)
- No N+1 queries (check entity relationships for missing `fetch = FetchType.LAZY`)
- No unbounded collection loading (always page; flag `findAll()` on large tables)

**Security:**
- SQL injection: verify no string concatenation in queries
- Path traversal: flag any file path constructed from user input
- Sensitive data: no PII or credentials in log statements or response bodies

---

## How to Review Agent-Generated PRs Effectively

The Engineer is the primary reviewer for all agent-generated PRs. Unlike human-authored PRs, agent PRs have specific failure modes that require a different review approach.

### The Agent PR Review Checklist

**1. Spec Alignment (5 minutes)**
- Does the PR description reference the spec section it implements?
- Does the implementation match the acceptance criteria in `spec.md`?
- Are there features in the PR that are NOT in the spec? (agents sometimes helpfully add unrequested functionality — reject these)

**2. Context Completeness (3 minutes)**
- Did the agent have the right ADRs in context? (Check if any ADR is contradicted)
- Did the agent acknowledge constraints from `CLAUDE.md`?
- Are there signs of context blindspots (e.g. agent reinvented a pattern that already exists in the codebase)?

**3. Code Quality (10 minutes)**
- Apply the Java/Spring Boot review checklist above
- Check imports: no unused imports, no wildcard imports
- Check for TODO comments left by the agent — these must be resolved or converted to GitHub issues before merge

**4. Test Quality (10 minutes)**
- Apply the test quality checklist above
- Run tests locally if any test failure suspicion; trust CI otherwise
- Confirm test names describe behaviour, not implementation

**5. Security Scan (5 minutes)**
- Review any new dependency for known CVEs (check the Harness CI OWASP dependency check step)
- Review any change to authentication/authorisation logic — flag for Architect

**6. Operational Readiness (3 minutes)**
- New endpoints have metrics registered (Micrometer)
- New background jobs have health indicators
- Logging is structured JSON (Logback config check)

### When to Approve vs. Request Changes

| Situation | Action |
|---|---|
| Minor style issues fixable in <5 min | Fix inline and approve |
| Spec misalignment — feature missing or wrong | Request changes; update prompt/context and re-run agent |
| Test quality insufficient | Request changes; add test patterns to CLAUDE.md |
| Security concern | Block; escalate to Architect immediately |
| ADR contradiction | Block; escalate to Architect; do not merge until resolved |
| Good implementation, missing edge cases | Approve with issue created for follow-up |
| Agent added unrequested scope | Request changes; strictly enforce scope boundaries |

---

## Context Engineering Responsibilities

Context engineering is the Engineer's primary craft. It determines the quality ceiling of all agent output.

### CLAUDE.md Ownership

The Engineer is the day-to-day maintainer of `products/<product>/CLAUDE.md`. The Orchestrator owns its structure and strategic content; the Engineer keeps it accurate and actionable.

**CLAUDE.md sections the Engineer maintains:**
- `## Codebase Conventions` — Java/Spring patterns, naming, package structure
- `## Test Standards` — test types, libraries, coverage targets
- `## Known Footguns` — common mistakes agents make in this codebase, with corrective guidance
- `## Active Sprint Context` — updated each sprint with current task context

**CLAUDE.md maintenance rules:**
- Maximum 8,000 tokens total (enforced by CI lint step)
- Every section dated; sections >30 days without update flagged in weekly review
- No duplicate information between CLAUDE.md and ADRs (CLAUDE.md points to ADRs, does not reproduce them)

### Curating Context Files

Beyond CLAUDE.md, the Engineer curates:
- `products/<product>/glossary.md` — domain terms with precise definitions
- `products/<product>/adr/manifest.json` — which ADRs to inject per phase (in coordination with Architect)
- Reference code snippets in `products/<product>/context/` — example implementations of approved patterns that agents can reference

### Prompt Engineering as a First-Class Discipline

Prompts are production artefacts. They are versioned, reviewed, and iterated with the same discipline as code.

**Structured improvement process:**
1. **Observe:** Note which prompt produced poor output and what the failure mode was
2. **Hypothesise:** Identify the likely cause (missing context, ambiguous instruction, wrong output format specified)
3. **Iterate:** Make one change to the prompt; re-run the agent
4. **Measure:** Compare output quality against the previous run
5. **Commit:** If improved, update the prompt template in `agents/<agent-name>.md` and commit with a descriptive message

**Prompt improvement log:** Engineers maintain a brief log in `products/<product>/context/prompt-log.md` noting what changed and why. This builds institutional knowledge and accelerates future prompt work.

---

## When to Intervene vs. Let Agents Proceed

### Let Agents Proceed (HOOTL)
- Generating boilerplate (DTOs, mappers, test scaffolding)
- Writing PR descriptions and commit messages
- Generating documentation from code
- Analysing logs and metrics
- Running repetitive refactors within established patterns

### Monitor and Review (HOTL)
- Implementing features from spec
- Writing integration tests
- Generating Flyway migrations (additive only; monitor for unintended changes)
- Updating configuration files

### Intervene (HITL)
- Any change to authentication or authorisation logic
- Any change affecting data persistence schemas
- Any introduction of a new external service dependency
- Any PR that touches more than 500 lines of production code
- Any test that would disable or reduce coverage for existing passing tests
- Any agent-generated code that the Engineer cannot explain to the Architect on demand

---

## Metrics the Engineer Tracks

| Metric | Target | Measurement Method | Review Cadence |
|---|---|---|---|
| Defect Escape Rate | <5% of merged PRs produce production bugs | Production incident correlation | Monthly |
| Test Coverage | >80% unit, 100% endpoint integration coverage | JaCoCo + CI report | Per sprint |
| PR Review Turnaround | Median <4h for agent PRs | Git timestamp analysis | Weekly |
| Context Quality Score | Qualitative, 1–5 scale | Quarterly peer review of CLAUDE.md | Quarterly |
| Agent Re-run Rate | <20% of agent tasks require a second run | Agent run log | Per sprint |
| Prompt Improvement Commits | At least 2 per sprint | Git log on `agents/` directory | Per sprint |
| Rework Ratio | <15% of agent-generated code requires major post-merge rework | Git blame + issue tracking | Monthly |

---

## Anti-Patterns to Avoid

### Reviewing Agent PRs Like Human PRs
Agent PRs are not written by a person who intended every line. Review with higher scepticism than you would apply to a trusted colleague's PR. An agent that lacked context will produce confident-looking wrong code.

### Fixing Agent Mistakes Without Updating Context
If an Engineer fixes a mistake in agent output without updating `CLAUDE.md` or the relevant prompt, the same mistake will recur in the next sprint. Every fix must be accompanied by a context update.

### Parallelising Too Many Agent Tasks
Running ten agent tasks simultaneously creates ten review queues simultaneously. Engineers should throttle agent parallelism to match their review capacity. Throughput is not the number of agent tasks started — it is the number of quality PRs merged.

### Treating Test Generation as Low-Value Work
Test generation is where agents add enormous leverage *and* where they can silently reduce quality (tests that always pass, tests that only test the mock, tests that do not cover the acceptance criteria). Test review is high-value, not administrative.

---

*See also: [`framework/governance/hitl-model.md`](../governance/hitl-model.md) | [`framework/standards/runtime-standards.md`](../standards/runtime-standards.md) | [`framework/standards/agent-interface.md`](../standards/agent-interface.md)*
