# Source Synthesis — Agentic Application Assessment Framework

**Purpose:** Extract reusable audit concepts from this repository's TDS, module specs, and completed audit to inform the portable assessment framework. This document separates universal behavioral invariants from repo-specific implementation preferences.

---

## Referenced Source Documents

| Document | Version | Lines | Role |
|----------|---------|-------|------|
| `docs/technical-design-specification.md` | 5.0.0 | ~600 | System architecture, module boundaries, technology stack |
| `docs/agent.core.md` | 2.0.0 | ~870 | Runtime engine, contracts, execution semantics |
| `docs/agent.api.md` | 1.0.0 | ~500 | HTTP interface, DTOs, SSE streaming |
| `docs/agent.workflows.md` | 1.0.0 | ~300 | Workflow/skill definitions, registry |
| `docs/agent.adapters.md` | 1.0.0 | ~400 | MCP servers, storage/cache adapters, normalization |
| `docs/agent.web.md` | 1.0.0 | ~300 | Frontend workbench UI |
| `.claude/skills/audit/SKILL.md` | — | 465 | Existing monolithic audit skill (44 principles, 9 domains) |
| `docs/aaron/assessments/agent-platform-audit-2026-03-26.md` | — | 147 | Completed audit against this project |

---

## Reusable Behavioral Invariants

These are architectural properties that apply to agentic applications generally, regardless of framework, language, or implementation approach. Each is derived from a specific TDS/spec concern but generalized.

### Execution Authority

| Source Pattern | Portable Invariant |
|---------------|-------------------|
| "AgentRuntime is the primary orchestrator. No other module initiates runs." (agent.core §6) | Execution authority must be centralized or consistently enforced — no module outside the runtime may independently initiate, drive, or complete agent executions. |
| "API route handlers delegate to runtime; no business logic in handlers." (TDS §5.2) | Transport layers must delegate execution decisions to the runtime — they accept, validate, and respond but do not own execution semantics. |
| "agent.web consumes agent.api exclusively — no direct core access." (TDS §5.2) | UI layers must consume the API boundary only — no direct access to runtime, orchestration, or integration layers. |

### Configuration Discipline

| Source Pattern | Portable Invariant |
|---------------|-------------------|
| "All runtime configuration lives in agent.yaml, validated at startup against AgentConfig." (TDS §7) | Runtime configuration must be externalized and validated at startup — hardcoded behavioral parameters in source code violate portability. |
| "If validation fails, the runtime does not start." (agent.core §4.1) | Startup must act as a safety gate — the system must refuse to start with invalid or incomplete configuration. |
| "Required servers block startup; optional servers degrade gracefully." (agent.core §6.2) | Required dependencies must be distinguished from optional ones — required failures are fatal, optional failures degrade gracefully with clear logging. |

### Observability

| Source Pattern | Portable Invariant |
|---------------|-------------------|
| "Every run carries a correlation ID propagated across all modules." (TDS §16.1) | Every execution must carry a unique identifier propagated across all components and external calls for the duration of that execution. |
| "All logging must use structlog with bound context." (agent.core §18.3) | All logging must be structured (key-value or JSON) with execution context — not free-form strings. |
| "SSE events buffered in Valkey with sequence numbers; reconnection supported." (audit P3.4) | If real-time event streaming is offered, it must support reconnection without event loss via buffering and sequence numbers. |
| "Events must allow reconstruction of full run state." (TDS §12) | Significant lifecycle transitions must emit structured events — clients must be able to reconstruct run state from the event stream alone. |

### Integration and Tool Model

| Source Pattern | Portable Invariant |
|---------------|-------------------|
| "MCPClient is the sole mechanism for all external tool interactions." (agent.core §11) | All external system interactions must go through a single, identifiable abstraction layer — no direct HTTP calls or subprocess invocations from the runtime. |
| "Tool catalog built at startup from MCP server discovery." (agent.core §11.1) | Tools must be registered and discoverable from a catalog or registry — not scattered or hardcoded. |
| "Normalization happens exactly once, at the adapter boundary." (TDS §14.4) | External data must be normalized to internal canonical schemas at the integration boundary — raw vendor payloads must not flow into the runtime. |

### Schema and Contract Discipline

| Source Pattern | Portable Invariant |
|---------------|-------------------|
| "All canonical contracts live in agent/core/contracts/. No other module may redefine." (agent.core §17) | System-wide data entities must be defined once in a single authoritative location — no duplicate definitions across modules. |
| "WorkflowDefinition (template) vs Run (record) clearly separated." (audit P5.2) | Definition schemas (what can be executed) must be separated from runtime records (what was executed). |
| "DTOs in agent/api/dtos/; contracts in agent/core/contracts/; no DTO imports in core." (audit P5.3) | Transport-layer data shapes must be separate from internal domain models — DTOs and canonical contracts are distinct. |
| "validate_output_against_format() exists but is never invoked." (audit GAP-2) | LLM outputs that feed downstream systems must be validated against a schema before use — unvalidated LLM text flowing into structured systems is a gap. |

### Error Handling

| Source Pattern | Portable Invariant |
|---------------|-------------------|
| "Error model with ErrorClass enum, message, detail, retryable flag, context IDs." (agent.core §15) | Errors must be first-class entities with consistent structure: classification, message, diagnostic detail, and execution context. |
| "Step failures propagate to run via state.status = FAILED with state.error set." (audit P6.3) | Component failures must propagate with context to the run level — silent error swallowing is a gap. |
| "Valkey writes are fire-and-forget; required MCP servers fail startup." (audit P6.4) | Non-critical infrastructure failures must not crash the system — graceful degradation for optional services, fail-fast for required ones. |

### Orchestration Model

| Source Pattern | Portable Invariant |
|---------------|-------------------|
| "VALID_TRANSITIONS dict enforces state machine; invalid transitions raise ValueError." (audit P7.2) | All run and step state transitions must be explicit, recorded, and non-skippable. |
| "Autonomous execution bounded by max_iterations, max_tool_calls, max_tokens." (agent.core §5.3) | Autonomous agent execution must have explicit guardrails — unbounded execution is a gap. |
| "Steps are known before execution begins; autonomous mode is explicitly separate." (TDS §9) | Structured workflows must be deterministic — LLM-constructed step sequences must be flagged as autonomous mode. |
| "Run records persist workflow_version, all step outputs, tool invocation I/O." (audit P7.5) | The system must record enough state to audit any run after the fact. |

### Scalability and Extensibility

| Source Pattern | Portable Invariant |
|---------------|-------------------|
| "Workflows loaded from YAML directory at startup; adding a workflow = adding a file." (audit P8.1) | New workflows must be addable without modifying the runtime or execution engine. |
| "Tools discovered from MCP servers at startup; adding tools = configuring servers." (audit P8.2) | New tools must be addable without modifying the agent loop. |
| "Runtime references no specific workflow/skill names; executes any conforming definition." (audit P8.5) | The execution engine must not reference specific definition names — it must execute any conforming definition generically. |

### Autonomous Agent Safety

| Source Pattern | Portable Invariant |
|---------------|-------------------|
| "Skills declare tool_restrictions (allowlist/denylist); enforced in agent_node." (audit P9.1) | The agent must not have unrestricted access to all tools in all contexts — tool access control must exist. |
| "Tool filtering iterates skills sequentially, each narrowing; takes restrictive intersection." (audit P9.2) | When multiple constraints are active with conflicting tool restrictions, the most restrictive intersection applies — not the permissive union. |
| "HUMAN_GATE step type pauses execution; approval API resolves." (audit P9.3) | For consequential actions, there must be a mechanism to require human approval before execution. |
| "If any limit is reached, the run transitions to COMPLETED with truncation, not FAILED." (agent.core §5.3) | Guardrail hits must produce completion with truncation — not failure. Truncation is expected boundary behavior, not an error. |

---

## Repo-Specific Patterns to Avoid Encoding as Universal Rules

These are implementation preferences from this repository that must NOT become mandatory assessment criteria.

| Repo-Specific Pattern | Why It Must Not Be Universal |
|-----------------------|------------------------------|
| Pydantic AI as orchestration framework | Many valid orchestration approaches exist (LangGraph, custom state machines, simple loops). The invariant is "deliberate orchestration choice," not "use Pydantic AI." |
| MCP as the tool protocol | LangChain tools, OpenAI function calling, plain registries, plugin systems are equivalent. The invariant is "single integration surface + catalog," not "use MCP." |
| agent.yaml as config format | TOML, JSON, env vars, Zod schemas are equivalent. The invariant is "externalized, validated config," not "YAML config." |
| Pydantic for schema validation | Zod, JSON Schema, TypeScript interfaces, Go structs are equivalent. The invariant is "typed, validated contracts," not "Pydantic models." |
| MongoDB for storage | Any persistent store satisfies auditability. The invariant is "run state persisted for audit," not "MongoDB." |
| Valkey for caching/SSE buffering | Any event buffer with sequence support satisfies reconnection. The invariant is "buffered event delivery with reconnection," not "Valkey." |
| FastAPI for HTTP | Any HTTP framework satisfies transport separation. The invariant is "thin transport layer," not "FastAPI." |
| structlog for logging | Any structured logging library satisfies P3.2. The invariant is "structured logging," not "structlog." |
| Logfire for tracing | Any distributed tracing system satisfies observability. The invariant is "traced execution," not "Logfire." |
| Five-module architecture (core/api/adapters/workflows/web) | Some systems have 2 modules; some have 20. The invariant is "clear boundaries with enforced dependency direction," not "exactly 5 modules." |
| YAML workflow definitions | Workflows can be defined in code, JSON, TOML, or DSLs. The invariant is "declarative, registerable definitions," not "YAML files." |
| Specific folder layout (`agent/core/contracts/`, etc.) | Folder structure varies by language and convention. The invariant is "canonical contracts in one location," not "in a `contracts/` directory." |
| `correlation_id` naming | `trace_id`, `request_id`, `run_id` are equivalent. The invariant is "propagated unique identifier," not "field named correlation_id." |

---

## Candidate Control-Plane Concerns

These are the behavioral surfaces the framework must assess. Derived from the 9 existing audit domains, confirmed against the TDS architecture.

| Concern | What It Governs | Failure Mode |
|---------|----------------|--------------|
| Execution boundaries | Who can initiate, drive, and complete runs | Execution logic scattered across modules; transport layer owns business logic |
| Configuration discipline | How runtime behavior is parameterized | Hardcoded values, unvalidated config, silent startup with missing deps |
| Observability | How execution is traced, logged, and monitored | Blind spots in traces, unstructured logs, missing event types, no run correlation |
| Integration and tool model | How external systems are accessed | Direct HTTP calls in runtime, no tool catalog, raw vendor payloads in business logic |
| Schema and contract discipline | How data shapes are defined and enforced | Duplicate entity definitions, DTOs used as domain models, unvalidated LLM output |
| Error handling | How failures are classified, propagated, and surfaced | Silent error swallowing, generic exceptions, infrastructure failures crashing the system |
| Orchestration model | How execution is sequenced and bounded | Implicit state transitions, unbounded autonomous loops, no run auditability |
| Scalability and extensibility | How new capabilities are added | Hardcoded workflows/tools/skills in runtime, behavior variation requires code changes |
| Autonomous agent safety | How agent authority is constrained | Unrestricted tool access, no approval gates, guardrail hits treated as failures |

---

## Candidate Domain/Principle Mappings

The existing audit skill defines 44 principles across 9 domains. After review, the mapping is complete and no new domains are needed. However, the following cross-domain interactions should be highlighted in the framework:

| Interaction | Domains | Effect |
|------------|---------|--------|
| P5.5 (LLM output validation) amplifies P5.4 (step output enforcement) | Schema × Schema | Together they mean zero output validation in the pipeline |
| P3.6 (uneven observability) amplifies P6.3 (failure propagation) | Observability × Error | Failures in unmonitored modules are harder to diagnose |
| P7.3 (bounded execution) depends on P9.5 (truncation not failure) | Orchestration × Safety | Guardrails must exist AND produce the right terminal state |
| P4.1 (single integration surface) enables P4.4 (tool observability) | Integration × Integration | Centralized tool access makes instrumentation possible |
| P2.3 (startup safety gate) depends on P2.1 (externalized config) | Config × Config | Can't validate at startup if config is hardcoded |

---

## Candidate Trace Patterns

These are execution traces the framework should guide assessors to follow. Derived from the TDS execution model but generalized.

| Trace Type | What to Follow | Principles Served |
|-----------|---------------|-------------------|
| Tool invocation trace | Tool selection → authorization/filtering → invocation → result capture → result propagation | P4.1, P4.2, P4.4, P9.1, P9.2 |
| Failure propagation trace | Error origin → step state update → run state update → event emission → API response | P6.1, P6.2, P6.3, P3.5 |
| Run lifecycle trace | Request arrival → dispatch → run creation → step execution → completion/failure → persistence | P1.1, P1.2, P7.2, P7.5 |
| Configuration trace | Config source → validation → startup gate → runtime consumption | P2.1, P2.2, P2.3, P2.4 |
| Schema boundary trace | External payload → normalization → validation → internal consumption → DTO projection | P4.3, P5.1, P5.3 |
| Correlation propagation trace | ID generation → logging context → event emission → tool call headers → error context | P3.1, P3.2, P3.5 |
| Guardrail enforcement trace | Limit definition → runtime check → truncation behavior → terminal state → output indicator | P7.3, P9.5 |
| Definition loading trace | File on disk → parse/validate → register in catalog → runtime resolution → execution | P8.1, P8.2, P8.3, P8.5 |

---

## Candidate Report Artifacts

The existing audit report structure is sound. The framework should formalize these elements:

| Artifact | Purpose | Source |
|----------|---------|--------|
| System hypothesis | Classify system type before assessment begins | Existing skill Phase 1 |
| Conformance overview table | Domain-level summary with counts | Existing audit report |
| Gap register | Per-finding detail with severity, evidence, risk, verification | Existing audit report |
| Passed principles list | Brief confirmation per principle | Existing audit report |
| N/A justifications | Per-principle rationale for inapplicability | Existing skill N/A conditions |
| Recommended next steps | Prioritized remediation actions | Existing audit report |

Additional artifacts the framework should add:

| Artifact | Purpose |
|----------|---------|
| System classification record | Formal classification against taxonomy (drives applicability) |
| Evidence inventory | What was read, what it evidenced, what was not found |
| Behavioral verification register | Principles requiring runtime verification, with suggested test cases |

---

## Terminology to Preserve

These terms from the TDS and existing audit carry precise meaning that the framework should adopt.

| Term | Meaning | Keep As-Is |
|------|---------|-----------|
| run | A single top-level execution instance | Yes |
| step | A unit of execution within a workflow | Yes |
| workflow | A predefined, ordered sequence of steps | Yes |
| skill | A reusable behavioral unit (system prompt + constraints) | Yes |
| tool | An external capability invocable by the agent | Yes |
| contract | A defined data shape with validation rules | Yes |
| enforcement point | Where a rule is checked and violations are rejected | Yes |
| propagation | How a value (ID, error, state) flows across boundaries | Yes |

## Terminology to Normalize

These terms are used inconsistently or carry repo-specific meaning that must be generalized.

| Repo Term | Normalized Term | Reason |
|-----------|----------------|--------|
| MCP server | external integration / tool provider | MCP is one protocol among many |
| MCP client | tool client / integration client | Same — protocol-specific |
| canonical contract | domain contract / canonical entity | "Canonical" is fine but context must clarify it means "authoritative domain model" |
| DTO | transport shape / API model | "DTO" is Java-ism but widely understood; keep as acceptable synonym |
| agent.core | runtime module / execution engine | Module names are repo-specific |
| agent.adapters | adapter layer / integration layer | Same |
| GraphState | execution state / run state | Framework-specific (Pydantic AI graph) |
| AgentConfig | configuration model | Class name is repo-specific |
| Logfire span | trace span / instrumentation span | Vendor-specific |
| Valkey | cache backend / event buffer | Product-specific |
| correlation_id | trace identifier / execution identifier | Naming convention varies |

---

## Open Design Decisions for the Framework

| Decision | Options | Recommendation |
|----------|---------|----------------|
| Should the framework support adding custom domains? | (a) Fixed 9 domains only (b) Extensible domain model | (b) — include extension guidance but ship with 9 |
| Should principle IDs be renumbered? | (a) Keep P1.1-P9.5 (b) Renumber by domain | (a) — preserves compatibility with existing audits |
| How should the SKILL.md reference individual principle files? | (a) Read all at startup (b) Read on demand per domain | (b) — keeps context window manageable for large frameworks |
| Should the framework define a machine-readable report format? | (a) Markdown only (b) Markdown + JSON schema | (a) for v1, with JSON schema as a documented extension point |
| How should behavioral verification flags work? | (a) Per-principle flag (b) Separate verification register | Both — flag in principle, aggregate in report |
| Should compact mode be a framework feature or a SKILL.md feature? | (a) Framework defines it (b) SKILL.md implements it | (b) — compact mode is an execution optimization, not a principle concern |
