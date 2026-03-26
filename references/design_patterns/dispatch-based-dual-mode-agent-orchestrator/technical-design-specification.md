# Agent Platform — Technical Design Specification

**Version:** 5.0.0  
**Status:** Authoritative — Build Specification  
**Scope:** System Architecture + Runtime Definition + Observability + Technology Stack  
**Audience:** Claude Code (implementation), developers (consumers)

---

## 1. Purpose

This document defines a portable, reusable agent platform that can be embedded into any application. It is the single authoritative reference for Claude Code during implementation. All architectural decisions, module boundaries, technology choices, and execution semantics are specified here or in the module documents referenced herein.

The platform must:

- Provide a reusable agent runtime capable of executing structured workflows and autonomous agent tasks
- Define skills as reusable behavioral units the agent draws on when reasoning independently
- Enforce canonical schemas as the single source of truth across all modules
- Integrate external systems exclusively through MCP servers, accessed via a first-class MCP client in the runtime
- Expose all functionality through a stable, versioned HTTP API
- Include a first-party workbench UI for running workflows and inspecting results
- Deliver strong, structured observability from the first line of code
- Be fully documented and reusable across projects with minimal integration effort

---

## 2. Technology Stack

All technology choices are locked. Claude Code must not substitute alternatives without explicit instruction.

### 2.1 Approved Stack

| Concern | Technology | Notes |
|---------|-----------|-------|
| Agent runtime / orchestration | Pydantic AI | Graph module for multi-step flows; native MCP client |
| LLM provider | Anthropic Claude (claude-sonnet-4-20250514) | Sole LLM provider |
| Tool protocol | MCP (Model Context Protocol) | All external tools exposed as MCP servers |
| MCP client | Pydantic AI native MCP client | Lives in agent.core; connects to all registered servers |
| API framework | FastAPI | Async, OpenAPI auto-generation |
| Streaming transport | SSE (Server-Sent Events) | Typed JSON events; see Section 12 |
| Observability | Logfire | Native Pydantic AI + OpenTelemetry integration |
| Structured logging | structlog | All modules |
| Cache backend | Valkey | Run state, session data, SSE event buffer, deduplication |
| Storage backend | MongoDB | Runs, artifacts, workflow registry, skill registry |
| Configuration | agent.yaml | Validated at startup against AgentConfig Pydantic model; see Section 7 |
| Frontend framework | React + TypeScript | |
| Frontend styling | Tailwind CSS | Core utility classes only |
| Frontend motion | framer-motion | |
| Frontend icons | lucide-react | |
| Frontend fonts | Rajdhani, IBM Plex Sans, JetBrains Mono | Dark ops aesthetic |

### 2.2 Prohibited Dependencies

The following are explicitly prohibited. Claude Code must not introduce these under any circumstances, regardless of framing or transitive dependency:

- LangChain
- LangGraph
- LangSmith
- Any langchain-* package
- Any langgraph-* package

If a capability appears to require one of these, the solution must be implemented using approved stack components.

---

## 3. Primary Goal

The agent runtime, workflows, skills, and integration interfaces must be reusable across applications with minimal modification. A new application adopts the platform by providing its own `agent.yaml`, workflow definitions, skill definitions, and MCP server configuration — with no modification to core modules required.

---

## 4. Core Architectural Principle

If a concept must be interpreted identically across applications, it belongs in agent.core. If a concept may legitimately vary by application, tenant, or deployment, it must remain outside the core and integrate through defined extension points.

This principle governs every boundary and dependency decision in the system.

---

## 5. System Modules

### 5.1 Module Overview

| Module | Role |
|--------|------|
| agent.core | Runtime engine, MCP client, canonical contracts, event/error model, observability primitives, configuration validation |
| agent.workflows | Workflow definitions, skill definitions, shared registry |
| agent.adapters | MCP servers (external integrations), storage adapter, cache adapter, payload normalization |
| agent.api | HTTP interface, DTOs, SSE streaming, authentication |
| agent.web | Workbench UI, run inspection, step trace viewer |

### 5.2 Module Responsibilities

**agent.core**

- Configuration model and startup validation (agent.yaml → AgentConfig)
- Runtime execution engine (Pydantic AI graph module)
- MCP client — connects to all registered MCP servers at startup, manages lifecycle, discovers tools
- Dispatch model — routes requests to workflow or autonomous execution path
- Run lifecycle management (created → running → completed / failed / cancelled)
- Step lifecycle management per step type
- Approval system
- Canonical contracts (single source of truth)
- Event model and structured event emission
- Error model and diagnostic context
- Observability primitives (Logfire instrumentation, trace propagation)
- Schema validation on MCP responses (does not normalize — see agent.adapters)

**agent.workflows**

- Workflow definitions (declarative, registry-driven)
- Skill definitions (reusable behavioral units)
- Workflow and skill registry (registration, discovery, versioning)
- Run context model (shared state passed across steps)

**agent.adapters**

- MCP servers for all external integrations (vendor-specific implementations)
- MongoDB storage adapter
- Valkey cache adapter
- External schema definitions
- Normalization of external payloads to canonical contracts (exactly once, at this boundary — before responses leave the adapter)

**agent.api**

- FastAPI HTTP interface
- Request/response DTOs (transport layer only — never used internally)
- Authentication and request context propagation
- Run execution endpoints
- SSE streaming endpoints
- Approval endpoints
- Workflow and skill discovery endpoints

**agent.web**

- Workbench UI: select workflow or invoke autonomous agent, provide input, submit
- Run inspection: real-time SSE event stream, step-by-step execution trace
- Artifact viewer
- Consumes agent.api exclusively — no direct core or adapter access

---

## 6. Dependency Direction

```
agent.web → agent.api → agent.core → agent.workflows
                                   → agent.adapters
```

Enforced rules:

- agent.web must not call agent.core or agent.adapters directly
- agent.adapters normalizes external payloads to canonical contracts before returning; agent.core validates but does not normalize
- agent.workflows must not implement runtime logic
- agent.api must not own execution semantics
- agent.core is the only module that may drive execution
- The MCP client lives in agent.core — no other module connects to MCP servers directly

---

## 7. agent.yaml — Configuration

All runtime configuration lives in `agent.yaml` at the project root. It is validated at startup against the `AgentConfig` Pydantic model defined in `agent/core/config/model.py`. If validation fails, the runtime does not start.

### 7.1 Required Sections

| Section | Contents |
|---------|----------|
| identity | Agent name, description, deployment version |
| mcp_servers | List of MCP server connections with `required` flag, timeouts, health check intervals |
| registries | Paths to workflow and skill definition directories |
| storage | MongoDB connection (URI, database name) |
| cache | Valkey connection and TTL configuration (SSE buffer, run context, paused graph state) |
| observability | Logfire project, environment, log level |
| execution | Default limits for workflow steps and autonomous mode guardrails |

### 7.2 MCP Server Required Flag

Each MCP server entry carries a `required` boolean (default: `true`):

- **required: true** — startup fails if this server is unreachable
- **required: false** — server is marked UNAVAILABLE in the tool catalog; startup continues

At least one server must be marked `required: true`.

### 7.3 Execution Defaults

The `execution` section defines default limits that apply when not overridden per-workflow or per-step:

| Setting | Default | Description |
|---------|---------|-------------|
| max_tool_calls_per_step | 25 | Maximum MCP tool invocations in a single AGENT step |
| max_tokens_per_step | 16384 | Maximum LLM tokens per AGENT step |
| step_timeout_seconds | 300 | Per-step execution timeout |
| autonomous_max_iterations | 50 | Hard cap on reasoning iterations in autonomous mode |
| autonomous_max_tool_calls | 100 | Hard cap on total tool invocations in autonomous mode |
| autonomous_max_tokens | 131072 | Hard cap on total LLM token spend in autonomous mode |

### 7.4 Loading Behavior

- `agent.yaml` is loaded once at startup; changes require a restart
- Config path can be overridden via the `AGENT_CONFIG_PATH` environment variable
- Workflow and skill definition files referenced in `registries` are loaded from disk and registered into the MongoDB-backed runtime registry at startup (files are the authoring-time source of truth; MongoDB is the runtime source of truth)

---

## 8. Capability Model

The agent has three capability surfaces it draws on at runtime:

```
Agent
├── Skills        — how to reason, respond, and behave
├── Workflows     — structured task sequences to execute
└── MCP Tools     — external systems to interact with
```

In autonomous mode the agent selects freely across all three surfaces based on the goal it receives.

In workflow mode the workflow drives execution, but individual agent steps within a workflow may invoke skills and MCP tools as directed by the step definition.

### 8.1 Skills

A skill is a reusable behavioral unit. It defines:

- A focused system prompt or reasoning approach
- Output format constraints
- Any tool restrictions applicable when the skill is active

Skills are defined in agent.workflows and registered in the shared registry. They are composable — a single agent invocation may activate multiple skills. Skill system prompts are concatenated in the order specified by the `skill_ids` list on the step definition (or `active_skills` on the run for run-level skills).

### 8.2 Workflows

A workflow is a predefined, ordered sequence of steps. It is deterministic in structure — the steps are known before execution begins. Individual steps may be automated (no LLM) or agent-driven (LLM + tools), but the sequence itself is fixed.

Workflows are defined in agent.workflows and registered in the shared registry.

### 8.3 MCP Tools

All external system interactions are exposed as MCP server tools. The MCP client in agent.core connects to registered servers at startup and makes their tools available to both workflow steps and autonomous agent execution uniformly.

---

## 9. Execution Paths

There are exactly two execution paths. All requests resolve to one of them.

### 9.1 Workflow Execution

Triggered when the request specifies a `workflow_id`.

- The workflow is resolved from the registry; the resolved version is recorded on the Run entity
- Steps execute sequentially in defined order
- Each step receives the run context object, which accumulates outputs from all prior steps
- The agent does not decide what to do next — the workflow definition drives sequencing
- Agent steps within the workflow are guided by a skill reference and bounded context
- Each step is bounded by `max_tool_calls_per_step`, `max_tokens_per_step`, and `step_timeout_seconds` (defaults from agent.yaml, overridable per step)

### 9.2 Autonomous Execution

Triggered when no `workflow_id` is specified.

- The agent receives a goal and an optional list of skills to activate
- The agent selects tools, determines its own sequence of actions, and reasons toward the goal
- No predefined steps — the Pydantic AI graph drives execution dynamically
- Behavioral consistency is enforced through active skills, not hardcoded prompts
- Output format is governed by the active skill's output constraints
- Execution is bounded by `autonomous_max_iterations`, `autonomous_max_tool_calls`, and `autonomous_max_tokens` from agent.yaml
- If any limit is reached, the run completes with a truncation indicator — it does not fail

### 9.3 Dispatch

Dispatch logic lives exclusively in agent.core. The dispatch model inspects the incoming request and routes to the correct execution path based on a single signal: the presence or absence of `workflow_id`. No classifier, no heuristics, no confidence thresholds. No other module makes routing decisions.

---

## 10. Step Model

Steps are the atomic unit of workflow execution. Sub-workflows are out of scope.

### 10.1 Step Types

| Type | Description | LLM Involved |
|------|-------------|--------------|
| AUTOMATED | Deterministic MCP tool call or code execution; no LLM | No |
| AGENT | LLM-driven reasoning; may invoke MCP tools; guided by a skill | Yes |
| HUMAN_GATE | Pauses execution pending human approval | No |

### 10.2 Run Context

Each workflow maintains a run context object — a shared state store scoped to the run. Every step may read outputs from any prior step by step ID and write its own output. The runtime passes the run context forward automatically; steps do not manage it directly.

The run context is persisted in MongoDB and cached in Valkey for active runs.

### 10.3 Step Execution Guarantees

- All state transitions must be explicit and recorded
- Every step must emit at minimum a `step.started` and `step.completed` or `step.failed` event
- All side effects must occur via MCP tool calls — never inline in runtime logic
- Every step output must be written to the run context before the next step begins

---

## 11. Approval System

### 11.1 Lifecycle

```
Step reaches HUMAN_GATE
    → Approval entity created (status: PENDING)
    → Run paused (status: AWAITING_APPROVAL)
    → approval.requested event emitted
    → Human resolves via API (APPROVED / REJECTED / MODIFIED)
    → approval.resolved event emitted
    → Run resumes from gate step
```

Paused graph state is preserved in Valkey with a configurable TTL (default: 7 days, set in agent.yaml).

### 11.2 Approval Canonical Entity

| Field | Description |
|-------|-------------|
| approval_id | Unique identifier |
| run_id | Parent run |
| step_id | Gate step that triggered the approval |
| requested_at | Timestamp |
| resolved_at | Timestamp (present on resolution) |
| status | PENDING \| APPROVED \| REJECTED \| MODIFIED |
| context | Data presented to the reviewer |
| resolution | Reviewer's decision payload |
| resolved_by | Resolver identity |

### 11.3 API Surface

```
POST /runs/{run_id}/approvals/{approval_id}/resolve
GET /runs/{run_id}/approvals
```

---

## 12. Streaming Contract

All streaming is delivered via SSE. The event stream is typed JSON. Clients must be able to reconstruct run state from the event stream alone.

### 12.1 SSE Endpoint

```
GET /runs/{run_id}/stream
Content-Type: text/event-stream
```

### 12.2 Event Envelope

```json
{
  "event_type": "string",
  "run_id": "string",
  "step_id": "string | null",
  "timestamp": "ISO-8601",
  "sequence": "integer",
  "payload": {}
}
```

### 12.3 Required Event Types

| Event Type | Emitted When |
|------------|-------------|
| run.created | Run entity created |
| run.started | Execution begins |
| run.completed | Run reaches terminal success |
| run.failed | Run reaches terminal failure |
| run.cancelled | Run is cancelled |
| run.awaiting_approval | Run pauses at HUMAN_GATE |
| step.started | Step execution begins |
| step.completed | Step execution succeeds |
| step.failed | Step execution fails |
| tool.invoked | MCP tool call dispatched |
| tool.result | MCP tool call returns |
| llm.token | Streaming token from LLM |
| approval.requested | Approval entity created |
| approval.resolved | Approval decision received |

### 12.4 Reconnection

- SSE connections are managed per-run
- Clients may reconnect with `?since_sequence=N` to resume from a sequence number
- Events are buffered in Valkey for a configurable window (default: 10 minutes, set in agent.yaml under `cache.sse_buffer_ttl_seconds`)

---

## 13. Workflow and Skill Registry

### 13.1 Design

Workflows and skills share a single registry backed by MongoDB, with type discrimination. Both are defined declaratively as YAML files and loaded at startup from the directories specified in `agent.yaml` under `registries`. Files are the authoring-time source of truth; MongoDB is the runtime source of truth.

### 13.2 Workflow Registration Contract

| Field | Description |
|-------|-------------|
| workflow_id | Unique identifier |
| name | Human-readable name |
| version | Semantic version |
| description | Purpose and behavior summary |
| steps | Ordered list of step definitions |
| input_schema | Pydantic model for validated input |
| output_schema | Pydantic model for validated output |
| tags | Classification tags |

### 13.3 Skill Registration Contract

| Field | Description |
|-------|-------------|
| skill_id | Unique identifier |
| name | Human-readable name |
| version | Semantic version |
| description | When and why to activate this skill |
| system_prompt | The behavioral instruction set |
| output_format | Constraints on response structure |
| tool_restrictions | Optional allowlist/denylist of MCP tools |

### 13.4 Versioning

- Multiple versions of a workflow or skill may coexist
- Unversioned requests resolve to the highest registered version
- The resolved version is recorded on the Run entity (`workflow_version`) for auditability
- Registry state is queryable via `GET /workflows` and `GET /skills`

---

## 14. Canonical Contract Strategy

### 14.1 Authority

All canonical contracts live in:

```
agent/core/contracts/
```

These are the single source of truth. No other module may redefine or duplicate them.

### 14.2 Contract Types by Module

| Type | Location | Purpose |
|------|----------|---------|
| Canonical | agent.core | System-wide meaning |
| Workflow / Skill | agent.workflows | Definition-layer schemas |
| External | agent.adapters | Vendor payloads, normalized before leaving adapter |
| DTO | agent.api | Transport shapes, never used internally |

### 14.3 Required Canonical Entities

| Entity | Description |
|--------|-------------|
| Asset | External resource referenced in a run |
| Run | A single top-level execution instance |
| Step | A unit of execution within a workflow |
| ToolInvocation | A discrete MCP tool call with inputs and outputs |
| ToolCatalogEntry | A tool discovered from an MCP server |
| Artifact | A produced output persisted from a run |
| Approval | A human-gate decision request and its resolution |
| Event | A structured emission from any point in the runtime |
| Error | A structured failure with diagnostic context |
| RunContext | Shared state store scoped to a run |

### 14.4 Canonical Data Flow

```
External Payload
    → MCP Server (agent.adapters) — normalize to canonical
    → MCP Client (agent.core) — validate against canonical schema
    → Runtime
    → DTO at API boundary
    → UI
```

Normalization happens exactly once, at the adapter boundary. agent.core validates that responses conform to canonical schemas but does not perform normalization.

---

## 15. MCP Client

The MCP client is a first-class component of agent.core. It is the sole mechanism by which the runtime interacts with external systems.

### 15.1 Responsibilities

- Connect to all MCP servers declared in agent.yaml at startup
- Enforce the `required` flag: required servers block startup on failure; optional servers degrade gracefully
- Manage server connection lifecycle (connect, health check, reconnect)
- Discover and cache available tools and their schemas at startup
- Validate tool inputs against discovered schemas before invocation
- Validate tool outputs against canonical schemas after invocation
- Invoke tools and return results
- Emit `tool.invoked` and `tool.result` events for every call
- Classify errors: connection failure / tool failure / validation failure / timeout
- Expose the full tool catalog to the Pydantic AI agent

### 15.2 Tool Catalog

At startup the MCP client builds a tool catalog — a unified index of all tools available across all connected servers. The catalog is queryable by the runtime and exposed via `GET /tools`.

Each catalog entry includes:

| Field | Description |
|-------|-------------|
| tool_id | Stable identifier ({server_id}.{tool_name}) |
| server_id | Which MCP server owns this tool |
| name | Tool name |
| description | What the tool does |
| input_schema | Expected input shape |
| output_schema | Expected output shape |
| available | Whether the server is currently reachable |

ToolCatalogEntry is defined in `agent/core/contracts/tool_catalog_entry.py` alongside all other canonical contracts.

---

## 16. Observability Model

Observability is mandatory from the first line of implementation.

### 16.1 Requirements

- Every run carries a correlation ID propagated across all modules and all external calls
- Every step emits structured start and completion/failure events
- All MCP tool calls are traced with latency and outcome
- All API requests propagate trace context
- All failures include structured diagnostic context

### 16.2 Coverage by Module

| Module | Observability Requirement |
|--------|--------------------------|
| agent.core | Run/step lifecycle events, execution telemetry, MCP client call traces |
| agent.adapters | MCP server instrumentation, latency, error classification |
| agent.api | Request tracing, SSE session telemetry, auth events |
| agent.web | Run timeline visualization, step trace viewer |

### 16.3 Logfire Configuration

- Pydantic AI native Logfire instrumentation enabled at startup
- Logfire project and environment read from agent.yaml
- structlog output forwarded to Logfire
- Trace IDs map 1:1 to run_id where possible
- Span names follow `agent.core.{component}.{operation}` (e.g., `agent.core.runtime.execute`, `agent.core.mcp.invoke`)

---# Agent Platform — Technical Design Specification

**Version:** 5.0.0  
**Status:** Authoritative — Build Specification  
**Scope:** System Architecture + Runtime Definition + Observability + Technology Stack  
**Audience:** Claude Code (implementation), developers (consumers)

---

## 1. Purpose

This document defines a portable, reusable agent platform that can be embedded into any application. It is the single authoritative reference for Claude Code during implementation. All architectural decisions, module boundaries, technology choices, and execution semantics are specified here or in the module documents referenced herein.

The platform must:

- Provide a reusable agent runtime capable of executing structured workflows and autonomous agent tasks
- Define skills as reusable behavioral units the agent draws on when reasoning independently
- Enforce canonical schemas as the single source of truth across all modules
- Integrate external systems exclusively through MCP servers, accessed via a first-class MCP client in the runtime
- Expose all functionality through a stable, versioned HTTP API
- Include a first-party workbench UI for running workflows and inspecting results
- Deliver strong, structured observability from the first line of code
- Be fully documented and reusable across projects with minimal integration effort

---

## 2. Technology Stack

All technology choices are locked. Claude Code must not substitute alternatives without explicit instruction.

### 2.1 Approved Stack

| Concern | Technology | Notes |
|---------|-----------|-------|
| Agent runtime / orchestration | Pydantic AI | Graph module for multi-step flows; native MCP client |
| LLM provider | Anthropic Claude (claude-sonnet-4-20250514) | Sole LLM provider |
| Tool protocol | MCP (Model Context Protocol) | All external tools exposed as MCP servers |
| MCP client | Pydantic AI native MCP client | Lives in agent.core; connects to all registered servers |
| API framework | FastAPI | Async, OpenAPI auto-generation |
| Streaming transport | SSE (Server-Sent Events) | Typed JSON events; see Section 12 |
| Observability | Logfire | Native Pydantic AI + OpenTelemetry integration |
| Structured logging | structlog | All modules |
| Cache backend | Valkey | Run state, session data, SSE event buffer, deduplication |
| Storage backend | MongoDB | Runs, artifacts, workflow registry, skill registry |
| Configuration | agent.yaml | Validated at startup against AgentConfig Pydantic model; see Section 7 |
| Frontend framework | React + TypeScript | |
| Frontend styling | Tailwind CSS | Core utility classes only |
| Frontend motion | framer-motion | |
| Frontend icons | lucide-react | |
| Frontend fonts | Rajdhani, IBM Plex Sans, JetBrains Mono | Dark ops aesthetic |

### 2.2 Prohibited Dependencies

The following are explicitly prohibited. Claude Code must not introduce these under any circumstances, regardless of framing or transitive dependency:

- LangChain
- LangGraph
- LangSmith
- Any langchain-* package
- Any langgraph-* package

If a capability appears to require one of these, the solution must be implemented using approved stack components.

---

## 3. Primary Goal

The agent runtime, workflows, skills, and integration interfaces must be reusable across applications with minimal modification. A new application adopts the platform by providing its own `agent.yaml`, workflow definitions, skill definitions, and MCP server configuration — with no modification to core modules required.

---

## 4. Core Architectural Principle

If a concept must be interpreted identically across applications, it belongs in agent.core. If a concept may legitimately vary by application, tenant, or deployment, it must remain outside the core and integrate through defined extension points.

This principle governs every boundary and dependency decision in the system.

---

## 5. System Modules

### 5.1 Module Overview

| Module | Role |
|--------|------|
| agent.core | Runtime engine, MCP client, canonical contracts, event/error model, observability primitives, configuration validation |
| agent.workflows | Workflow definitions, skill definitions, shared registry |
| agent.adapters | MCP servers (external integrations), storage adapter, cache adapter, payload normalization |
| agent.api | HTTP interface, DTOs, SSE streaming, authentication |
| agent.web | Workbench UI, run inspection, step trace viewer |

### 5.2 Module Responsibilities

**agent.core**

- Configuration model and startup validation (agent.yaml → AgentConfig)
- Runtime execution engine (Pydantic AI graph module)
- MCP client — connects to all registered MCP servers at startup, manages lifecycle, discovers tools
- Dispatch model — routes requests to workflow or autonomous execution path
- Run lifecycle management (created → running → completed / failed / cancelled)
- Step lifecycle management per step type
- Approval system
- Canonical contracts (single source of truth)
- Event model and structured event emission
- Error model and diagnostic context
- Observability primitives (Logfire instrumentation, trace propagation)
- Schema validation on MCP responses (does not normalize — see agent.adapters)

**agent.workflows**

- Workflow definitions (declarative, registry-driven)
- Skill definitions (reusable behavioral units)
- Workflow and skill registry (registration, discovery, versioning)
- Run context model (shared state passed across steps)

**agent.adapters**

- MCP servers for all external integrations (vendor-specific implementations)
- MongoDB storage adapter
- Valkey cache adapter
- External schema definitions
- Normalization of external payloads to canonical contracts (exactly once, at this boundary — before responses leave the adapter)

**agent.api**

- FastAPI HTTP interface
- Request/response DTOs (transport layer only — never used internally)
- Authentication and request context propagation
- Run execution endpoints
- SSE streaming endpoints
- Approval endpoints
- Workflow and skill discovery endpoints

**agent.web**

- Workbench UI: select workflow or invoke autonomous agent, provide input, submit
- Run inspection: real-time SSE event stream, step-by-step execution trace
- Artifact viewer
- Consumes agent.api exclusively — no direct core or adapter access

---

## 6. Dependency Direction

```
agent.web → agent.api → agent.core → agent.workflows
                                   → agent.adapters
```

Enforced rules:

- agent.web must not call agent.core or agent.adapters directly
- agent.adapters normalizes external payloads to canonical contracts before returning; agent.core validates but does not normalize
- agent.workflows must not implement runtime logic
- agent.api must not own execution semantics
- agent.core is the only module that may drive execution
- The MCP client lives in agent.core — no other module connects to MCP servers directly

---

## 7. agent.yaml — Configuration

All runtime configuration lives in `agent.yaml` at the project root. It is validated at startup against the `AgentConfig` Pydantic model defined in `agent/core/config/model.py`. If validation fails, the runtime does not start.

### 7.1 Required Sections

| Section | Contents |
|---------|----------|
| identity | Agent name, description, deployment version |
| mcp_servers | List of MCP server connections with `required` flag, timeouts, health check intervals |
| registries | Paths to workflow and skill definition directories |
| storage | MongoDB connection (URI, database name) |
| cache | Valkey connection and TTL configuration (SSE buffer, run context, paused graph state) |
| observability | Logfire project, environment, log level |
| execution | Default limits for workflow steps and autonomous mode guardrails |

### 7.2 MCP Server Required Flag

Each MCP server entry carries a `required` boolean (default: `true`):

- **required: true** — startup fails if this server is unreachable
- **required: false** — server is marked UNAVAILABLE in the tool catalog; startup continues

At least one server must be marked `required: true`.

### 7.3 Execution Defaults

The `execution` section defines default limits that apply when not overridden per-workflow or per-step:

| Setting | Default | Description |
|---------|---------|-------------|
| max_tool_calls_per_step | 25 | Maximum MCP tool invocations in a single AGENT step |
| max_tokens_per_step | 16384 | Maximum LLM tokens per AGENT step |
| step_timeout_seconds | 300 | Per-step execution timeout |
| autonomous_max_iterations | 50 | Hard cap on reasoning iterations in autonomous mode |
| autonomous_max_tool_calls | 100 | Hard cap on total tool invocations in autonomous mode |
| autonomous_max_tokens | 131072 | Hard cap on total LLM token spend in autonomous mode |

### 7.4 Loading Behavior

- `agent.yaml` is loaded once at startup; changes require a restart
- Config path can be overridden via the `AGENT_CONFIG_PATH` environment variable
- Workflow and skill definition files referenced in `registries` are loaded from disk and registered into the MongoDB-backed runtime registry at startup (files are the authoring-time source of truth; MongoDB is the runtime source of truth)

---

## 8. Capability Model

The agent has three capability surfaces it draws on at runtime:

```
Agent
├── Skills        — how to reason, respond, and behave
├── Workflows     — structured task sequences to execute
└── MCP Tools     — external systems to interact with
```

In autonomous mode the agent selects freely across all three surfaces based on the goal it receives.

In workflow mode the workflow drives execution, but individual agent steps within a workflow may invoke skills and MCP tools as directed by the step definition.

### 8.1 Skills

A skill is a reusable behavioral unit. It defines:

- A focused system prompt or reasoning approach
- Output format constraints
- Any tool restrictions applicable when the skill is active

Skills are defined in agent.workflows and registered in the shared registry. They are composable — a single agent invocation may activate multiple skills. Skill system prompts are concatenated in the order specified by the `skill_ids` list on the step definition (or `active_skills` on the run for run-level skills).

### 8.2 Workflows

A workflow is a predefined, ordered sequence of steps. It is deterministic in structure — the steps are known before execution begins. Individual steps may be automated (no LLM) or agent-driven (LLM + tools), but the sequence itself is fixed.

Workflows are defined in agent.workflows and registered in the shared registry.

### 8.3 MCP Tools

All external system interactions are exposed as MCP server tools. The MCP client in agent.core connects to registered servers at startup and makes their tools available to both workflow steps and autonomous agent execution uniformly.

---

## 9. Execution Paths

There are exactly two execution paths. All requests resolve to one of them.

### 9.1 Workflow Execution

Triggered when the request specifies a `workflow_id`.

- The workflow is resolved from the registry; the resolved version is recorded on the Run entity
- Steps execute sequentially in defined order
- Each step receives the run context object, which accumulates outputs from all prior steps
- The agent does not decide what to do next — the workflow definition drives sequencing
- Agent steps within the workflow are guided by a skill reference and bounded context
- Each step is bounded by `max_tool_calls_per_step`, `max_tokens_per_step`, and `step_timeout_seconds` (defaults from agent.yaml, overridable per step)

### 9.2 Autonomous Execution

Triggered when no `workflow_id` is specified.

- The agent receives a goal and an optional list of skills to activate
- The agent selects tools, determines its own sequence of actions, and reasons toward the goal
- No predefined steps — the Pydantic AI graph drives execution dynamically
- Behavioral consistency is enforced through active skills, not hardcoded prompts
- Output format is governed by the active skill's output constraints
- Execution is bounded by `autonomous_max_iterations`, `autonomous_max_tool_calls`, and `autonomous_max_tokens` from agent.yaml
- If any limit is reached, the run completes with a truncation indicator — it does not fail

### 9.3 Dispatch

Dispatch logic lives exclusively in agent.core. The dispatch model inspects the incoming request and routes to the correct execution path based on a single signal: the presence or absence of `workflow_id`. No classifier, no heuristics, no confidence thresholds. No other module makes routing decisions.

---

## 10. Step Model

Steps are the atomic unit of workflow execution. Sub-workflows are out of scope.

### 10.1 Step Types

| Type | Description | LLM Involved |
|------|-------------|--------------|
| AUTOMATED | Deterministic MCP tool call or code execution; no LLM | No |
| AGENT | LLM-driven reasoning; may invoke MCP tools; guided by a skill | Yes |
| HUMAN_GATE | Pauses execution pending human approval | No |

### 10.2 Run Context

Each workflow maintains a run context object — a shared state store scoped to the run. Every step may read outputs from any prior step by step ID and write its own output. The runtime passes the run context forward automatically; steps do not manage it directly.

The run context is persisted in MongoDB and cached in Valkey for active runs.

### 10.3 Step Execution Guarantees

- All state transitions must be explicit and recorded
- Every step must emit at minimum a `step.started` and `step.completed` or `step.failed` event
- All side effects must occur via MCP tool calls — never inline in runtime logic
- Every step output must be written to the run context before the next step begins

---

## 11. Approval System

### 11.1 Lifecycle

```
Step reaches HUMAN_GATE
    → Approval entity created (status: PENDING)
    → Run paused (status: AWAITING_APPROVAL)
    → approval.requested event emitted
    → Human resolves via API (APPROVED / REJECTED / MODIFIED)
    → approval.resolved event emitted
    → Run resumes from gate step
```

Paused graph state is preserved in Valkey with a configurable TTL (default: 7 days, set in agent.yaml).

### 11.2 Approval Canonical Entity

| Field | Description |
|-------|-------------|
| approval_id | Unique identifier |
| run_id | Parent run |
| step_id | Gate step that triggered the approval |
| requested_at | Timestamp |
| resolved_at | Timestamp (present on resolution) |
| status | PENDING \| APPROVED \| REJECTED \| MODIFIED |
| context | Data presented to the reviewer |
| resolution | Reviewer's decision payload |
| resolved_by | Resolver identity |

### 11.3 API Surface

```
POST /runs/{run_id}/approvals/{approval_id}/resolve
GET /runs/{run_id}/approvals
```

---

## 12. Streaming Contract

All streaming is delivered via SSE. The event stream is typed JSON. Clients must be able to reconstruct run state from the event stream alone.

### 12.1 SSE Endpoint

```
GET /runs/{run_id}/stream
Content-Type: text/event-stream
```

### 12.2 Event Envelope

```json
{
  "event_type": "string",
  "run_id": "string",
  "step_id": "string | null",
  "timestamp": "ISO-8601",
  "sequence": "integer",
  "payload": {}
}
```

### 12.3 Required Event Types

| Event Type | Emitted When |
|------------|-------------|
| run.created | Run entity created |
| run.started | Execution begins |
| run.completed | Run reaches terminal success |
| run.failed | Run reaches terminal failure |
| run.cancelled | Run is cancelled |
| run.awaiting_approval | Run pauses at HUMAN_GATE |
| step.started | Step execution begins |
| step.completed | Step execution succeeds |
| step.failed | Step execution fails |
| tool.invoked | MCP tool call dispatched |
| tool.result | MCP tool call returns |
| llm.token | Streaming token from LLM |
| approval.requested | Approval entity created |
| approval.resolved | Approval decision received |

### 12.4 Reconnection

- SSE connections are managed per-run
- Clients may reconnect with `?since_sequence=N` to resume from a sequence number
- Events are buffered in Valkey for a configurable window (default: 10 minutes, set in agent.yaml under `cache.sse_buffer_ttl_seconds`)

---

## 13. Workflow and Skill Registry

### 13.1 Design

Workflows and skills share a single registry backed by MongoDB, with type discrimination. Both are defined declaratively as YAML files and loaded at startup from the directories specified in `agent.yaml` under `registries`. Files are the authoring-time source of truth; MongoDB is the runtime source of truth.

### 13.2 Workflow Registration Contract

| Field | Description |
|-------|-------------|
| workflow_id | Unique identifier |
| name | Human-readable name |
| version | Semantic version |
| description | Purpose and behavior summary |
| steps | Ordered list of step definitions |
| input_schema | Pydantic model for validated input |
| output_schema | Pydantic model for validated output |
| tags | Classification tags |

### 13.3 Skill Registration Contract

| Field | Description |
|-------|-------------|
| skill_id | Unique identifier |
| name | Human-readable name |
| version | Semantic version |
| description | When and why to activate this skill |
| system_prompt | The behavioral instruction set |
| output_format | Constraints on response structure |
| tool_restrictions | Optional allowlist/denylist of MCP tools |

### 13.4 Versioning

- Multiple versions of a workflow or skill may coexist
- Unversioned requests resolve to the highest registered version
- The resolved version is recorded on the Run entity (`workflow_version`) for auditability
- Registry state is queryable via `GET /workflows` and `GET /skills`

---

## 14. Canonical Contract Strategy

### 14.1 Authority

All canonical contracts live in:

```
agent/core/contracts/
```

These are the single source of truth. No other module may redefine or duplicate them.

### 14.2 Contract Types by Module

| Type | Location | Purpose |
|------|----------|---------|
| Canonical | agent.core | System-wide meaning |
| Workflow / Skill | agent.workflows | Definition-layer schemas |
| External | agent.adapters | Vendor payloads, normalized before leaving adapter |
| DTO | agent.api | Transport shapes, never used internally |

### 14.3 Required Canonical Entities

| Entity | Description |
|--------|-------------|
| Asset | External resource referenced in a run |
| Run | A single top-level execution instance |
| Step | A unit of execution within a workflow |
| ToolInvocation | A discrete MCP tool call with inputs and outputs |
| ToolCatalogEntry | A tool discovered from an MCP server |
| Artifact | A produced output persisted from a run |
| Approval | A human-gate decision request and its resolution |
| Event | A structured emission from any point in the runtime |
| Error | A structured failure with diagnostic context |
| RunContext | Shared state store scoped to a run |

### 14.4 Canonical Data Flow

```
External Payload
    → MCP Server (agent.adapters) — normalize to canonical
    → MCP Client (agent.core) — validate against canonical schema
    → Runtime
    → DTO at API boundary
    → UI
```

Normalization happens exactly once, at the adapter boundary. agent.core validates that responses conform to canonical schemas but does not perform normalization.

---

## 15. MCP Client

The MCP client is a first-class component of agent.core. It is the sole mechanism by which the runtime interacts with external systems.

### 15.1 Responsibilities

- Connect to all MCP servers declared in agent.yaml at startup
- Enforce the `required` flag: required servers block startup on failure; optional servers degrade gracefully
- Manage server connection lifecycle (connect, health check, reconnect)
- Discover and cache available tools and their schemas at startup
- Validate tool inputs against discovered schemas before invocation
- Validate tool outputs against canonical schemas after invocation
- Invoke tools and return results
- Emit `tool.invoked` and `tool.result` events for every call
- Classify errors: connection failure / tool failure / validation failure / timeout
- Expose the full tool catalog to the Pydantic AI agent

### 15.2 Tool Catalog

At startup the MCP client builds a tool catalog — a unified index of all tools available across all connected servers. The catalog is queryable by the runtime and exposed via `GET /tools`.

Each catalog entry includes:

| Field | Description |
|-------|-------------|
| tool_id | Stable identifier ({server_id}.{tool_name}) |
| server_id | Which MCP server owns this tool |
| name | Tool name |
| description | What the tool does |
| input_schema | Expected input shape |
| output_schema | Expected output shape |
| available | Whether the server is currently reachable |

ToolCatalogEntry is defined in `agent/core/contracts/tool_catalog_entry.py` alongside all other canonical contracts.

---

## 16. Observability Model

Observability is mandatory from the first line of implementation.

### 16.1 Requirements

- Every run carries a correlation ID propagated across all modules and all external calls
- Every step emits structured start and completion/failure events
- All MCP tool calls are traced with latency and outcome
- All API requests propagate trace context
- All failures include structured diagnostic context

### 16.2 Coverage by Module

| Module | Observability Requirement |
|--------|--------------------------|
| agent.core | Run/step lifecycle events, execution telemetry, MCP client call traces |
| agent.adapters | MCP server instrumentation, latency, error classification |
| agent.api | Request tracing, SSE session telemetry, auth events |
| agent.web | Run timeline visualization, step trace viewer |

### 16.3 Logfire Configuration

- Pydantic AI native Logfire instrumentation enabled at startup
- Logfire project and environment read from agent.yaml
- structlog output forwarded to Logfire
- Trace IDs map 1:1 to run_id where possible
- Span names follow `agent.core.{component}.{operation}` (e.g., `agent.core.runtime.execute`, `agent.core.mcp.invoke`)

---

## 17. Boundary Rules

**Required**

- External payloads → canonical form at MCP server / adapter boundary (normalization)
- Canonical schemas → validated at MCP client / agent.core boundary (validation)
- Canonical entities → DTOs at agent.api boundary
- DTOs → UI rendering in agent.web
- All external side effects via MCP client → MCP servers

**Prohibited**

- Raw vendor payloads appearing in runtime or API layers
- Duplicate schema definitions across modules
- DTOs used in internal runtime logic
- agent.web bypassing agent.api
- agent.adapters defining canonical meaning
- Execution logic outside agent.core
- Direct MCP server connections from any module other than agent.core
- Any LangChain, LangGraph, or LangSmith dependency (see Section 2.2)
- Hardcoded TTLs, timeouts, or execution limits in runtime code (all from agent.yaml)

---

## 18. Documentation Requirements

Documentation is generated alongside code, not after.

**Required Documents**

```
docs/
  technical-design-specification.md
  modules/
    agent.core.md
    agent.workflows.md
    agent.adapters.md
    agent.api.md
    agent.web.md
```

**Required Content Per Module Document**

- Purpose and scope
- Responsibilities
- Module boundaries
- Canonical contracts owned or consumed
- Key implementation patterns
- Extension guidance
- Example execution trace or request/response

---

## 19. Implementation Sequence

Implement in this order. Do not begin a phase until the prior phase is complete and tested.

1. Define AgentConfig model and config loader in agent/core/config/
2. Define all canonical contracts in agent/core/contracts/
3. Implement MCP client in agent.core (server connections, tool catalog, invocation)
4. Implement agent.core runtime (Pydantic AI graph, run/step lifecycle, event emission, dispatch)
5. Implement agent.adapters (MongoDB, Valkey, first MCP server)
6. Implement agent.workflows (registry, first workflow, first skill)
7. Implement agent.api (FastAPI, SSE streaming, approval endpoints)
8. Implement agent.web (workbench UI, run inspection)
9. Implement observability end-to-end (Logfire, structlog, trace propagation)
10. Generate module documentation

---

## 20. Acceptance Criteria

**Structure**

- [ ] All five modules exist with correct boundaries enforced
- [ ] Canonical contracts centralized in agent.core/contracts/ — including ToolCatalogEntry
- [ ] No schema duplication across modules
- [ ] No LangChain / LangGraph / LangSmith dependency anywhere in the dependency graph
- [ ] DTOs absent from internal runtime logic
- [ ] agent.yaml validated at startup via AgentConfig; startup fails on invalid config

**Behavior — Workflow Path**

- [ ] A workflow run can be created, executed, and completed via the API
- [ ] Resolved workflow version is recorded on the Run entity
- [ ] Run context is populated by each step and readable by subsequent steps
- [ ] An automated step executes a deterministic MCP tool call without LLM involvement
- [ ] An agent step invokes the LLM with a bound skill and produces guided output
- [ ] A HUMAN_GATE step pauses and resumes correctly on approval resolution
- [ ] SSE stream delivers all required event types in correct sequence order
- [ ] Step execution respects configured limits (max tool calls, max tokens, timeout)

**Behavior — Autonomous Path**

- [ ] An autonomous request with no workflow_id routes correctly
- [ ] The agent selects and invokes MCP tools independently
- [ ] Active skills constrain output format consistently
- [ ] Full execution is observable via the SSE stream
- [ ] Autonomous execution respects configured guardrails (max iterations, max tool calls, max tokens)
- [ ] Hitting a guardrail produces COMPLETED with truncation indicator, not FAILED

**MCP Server Lifecycle**

- [ ] Required servers block startup on connection failure
- [ ] Optional servers degrade gracefully — marked UNAVAILABLE, startup continues
- [ ] Tool catalog reflects only connected servers
- [ ] Health checks run at configured intervals

**Observability**

- [ ] Every run has a correlation ID visible in Logfire
- [ ] All step transitions captured as spans
- [ ] MCP tool call latency recorded per invocation
- [ ] A failed run produces structured diagnostic context
- [ ] All Logfire and structlog configuration reads from agent.yaml

**Reusability**

- [ ] A second project can onboard via agent.yaml, workflow/skill definitions, and MCP server config with no core modification
- [ ] Documentation exists for all modules at the required level of detail

---

## 21. Final Directive

Claude Code must implement the system exactly as specified in this document and in the module documents under docs/modules/.

Key constraints:

- Canonical meaning defined once in agent.core
- LangChain, LangGraph, LangSmith are prohibited — no exceptions
- All external interactions via MCP client → MCP servers
- Normalization at the adapter boundary, validation at the core boundary
- The API is the only external integration surface
- agent.web is a consumer, not a controller
- Observability is mandatory from inception
- Technology stack in Section 2 is locked
- All runtime configuration in agent.yaml, validated by AgentConfig at startup
- No hardcoded timeouts, TTLs, or execution limits — all from configuration
- Documentation is a deliverable, not an afterthought

---

## Appendix: Changes from v4.0.0

| Change | Rationale |
|--------|-----------|
| Replaced AGENT.md with agent.yaml + AgentConfig | Structured, validated config replaces ambiguous markdown parsing |
| Added Section 7.2: MCP server required flag | Resolves startup contradiction with explicit per-server policy |
| Added Section 7.3: execution defaults with autonomous guardrails | Configurable limits for workflow steps and autonomous mode |
| Added AgentConfig as step 1 in implementation sequence | Config validation is a prerequisite for everything else |
| Clarified normalization boundary in Sections 6, 14.4, 15.1, 17 | Adapters normalize, core validates — stated once, enforced everywhere |
| Dropped WorkflowExecution from canonical entities | Run entity carries workflow_version directly; separate entity was redundant |
| Added ToolCatalogEntry to canonical entities (Section 14.3) | All contracts in contracts/ — no exceptions |
| Added workflow_version to Run entity (Section 13.4) | Captures resolved version for auditability |
| Updated span naming convention (Section 16.3) | agent.core.{component}.{operation} is more specific and consistent |
| Clarified registry source of truth (Section 7.4, 13.1) | Files are authoring-time, MongoDB is runtime — loaded at startup |
| Added autonomous truncation behavior (Section 9.2) | Hitting a guardrail produces COMPLETED with indicator, not FAILED |
| Added hardcoded-values prohibition (Section 17) | All TTLs, timeouts, limits must come from agent.yaml |
| Expanded acceptance criteria | MCP lifecycle, autonomous guardrails, config validation |alization)
- Canonical schemas → validated at MCP client / agent.core boundary (validation)
- Canonical entities → DTOs at agent.api boundary
- DTOs → UI rendering in agent.web
- All external side effects via MCP client → MCP servers

**Prohibited**

- Raw vendor payloads appearing in runtime or API layers
- Duplicate schema definitions across modules
- DTOs used in internal runtime logic
- agent.web bypassing agent.api
- agent.adapters defining canonical meaning
- Execution logic outside agent.core
- Direct MCP server connections from any module other than agent.core
- Any LangChain, LangGraph, or LangSmith dependency (see Section 2.2)
- Hardcoded TTLs, timeouts, or execution limits in runtime code (all from agent.yaml)

---

## 18. Documentation Requirements

Documentation is generated alongside code, not after.

**Required Documents**

```
docs/
  technical-design-specification.md
  modules/
    agent.core.md
    agent.workflows.md
    agent.adapters.md
    agent.api.md
    agent.web.md
```

**Required Content Per Module Document**

- Purpose and scope
- Responsibilities
- Module boundaries
- Canonical contracts owned or consumed
- Key implementation patterns
- Extension guidance
- Example execution trace or request/response

---

## 19. Implementation Sequence

Implement in this order. Do not begin a phase until the prior phase is complete and tested.

1. Define AgentConfig model and config loader in agent/core/config/
2. Define all canonical contracts in agent/core/contracts/
3. Implement MCP client in agent.core (server connections, tool catalog, invocation)
4. Implement agent.core runtime (Pydantic AI graph, run/step lifecycle, event emission, dispatch)
5. Implement agent.adapters (MongoDB, Valkey, first MCP server)
6. Implement agent.workflows (registry, first workflow, first skill)
7. Implement agent.api (FastAPI, SSE streaming, approval endpoints)
8. Implement agent.web (workbench UI, run inspection)
9. Implement observability end-to-end (Logfire, structlog, trace propagation)
10. Generate module documentation

---

## 20. Acceptance Criteria

**Structure**

- [ ] All five modules exist with correct boundaries enforced
- [ ] Canonical contracts centralized in agent.core/contracts/ — including ToolCatalogEntry
- [ ] No schema duplication across modules
- [ ] No LangChain / LangGraph / LangSmith dependency anywhere in the dependency graph
- [ ] DTOs absent from internal runtime logic
- [ ] agent.yaml validated at startup via AgentConfig; startup fails on invalid config

**Behavior — Workflow Path**

- [ ] A workflow run can be created, executed, and completed via the API
- [ ] Resolved workflow version is recorded on the Run entity
- [ ] Run context is populated by each step and readable by subsequent steps
- [ ] An automated step executes a deterministic MCP tool call without LLM involvement
- [ ] An agent step invokes the LLM with a bound skill and produces guided output
- [ ] A HUMAN_GATE step pauses and resumes correctly on approval resolution
- [ ] SSE stream delivers all required event types in correct sequence order
- [ ] Step execution respects configured limits (max tool calls, max tokens, timeout)

**Behavior — Autonomous Path**

- [ ] An autonomous request with no workflow_id routes correctly
- [ ] The agent selects and invokes MCP tools independently
- [ ] Active skills constrain output format consistently
- [ ] Full execution is observable via the SSE stream
- [ ] Autonomous execution respects configured guardrails (max iterations, max tool calls, max tokens)
- [ ] Hitting a guardrail produces COMPLETED with truncation indicator, not FAILED

**MCP Server Lifecycle**

- [ ] Required servers block startup on connection failure
- [ ] Optional servers degrade gracefully — marked UNAVAILABLE, startup continues
- [ ] Tool catalog reflects only connected servers
- [ ] Health checks run at configured intervals

**Observability**

- [ ] Every run has a correlation ID visible in Logfire
- [ ] All step transitions captured as spans
- [ ] MCP tool call latency recorded per invocation
- [ ] A failed run produces structured diagnostic context
- [ ] All Logfire and structlog configuration reads from agent.yaml

**Reusability**

- [ ] A second project can onboard via agent.yaml, workflow/skill definitions, and MCP server config with no core modification
- [ ] Documentation exists for all modules at the required level of detail

---

## 21. Final Directive

Claude Code must implement the system exactly as specified in this document and in the module documents under docs/modules/.

Key constraints:

- Canonical meaning defined once in agent.core
- LangChain, LangGraph, LangSmith are prohibited — no exceptions
- All external interactions via MCP client → MCP servers
- Normalization at the adapter boundary, validation at the core boundary
- The API is the only external integration surface
- agent.web is a consumer, not a controller
- Observability is mandatory from inception
- Technology stack in Section 2 is locked
- All runtime configuration in agent.yaml, validated by AgentConfig at startup
- No hardcoded timeouts, TTLs, or execution limits — all from configuration
- Documentation is a deliverable, not an afterthought

---

## Appendix: Changes from v4.0.0

| Change | Rationale |
|--------|-----------|
| Replaced AGENT.md with agent.yaml + AgentConfig | Structured, validated config replaces ambiguous markdown parsing |
| Added Section 7.2: MCP server required flag | Resolves startup contradiction with explicit per-server policy |
| Added Section 7.3: execution defaults with autonomous guardrails | Configurable limits for workflow steps and autonomous mode |
| Added AgentConfig as step 1 in implementation sequence | Config validation is a prerequisite for everything else |
| Clarified normalization boundary in Sections 6, 14.4, 15.1, 17 | Adapters normalize, core validates — stated once, enforced everywhere |
| Dropped WorkflowExecution from canonical entities | Run entity carries workflow_version directly; separate entity was redundant |
| Added ToolCatalogEntry to canonical entities (Section 14.3) | All contracts in contracts/ — no exceptions |
| Added workflow_version to Run entity (Section 13.4) | Captures resolved version for auditability |
| Updated span naming convention (Section 16.3) | agent.core.{component}.{operation} is more specific and consistent |
| Clarified registry source of truth (Section 7.4, 13.1) | Files are authoring-time, MongoDB is runtime — loaded at startup |
| Added autonomous truncation behavior (Section 9.2) | Hitting a guardrail produces COMPLETED with indicator, not FAILED |
| Added hardcoded-values prohibition (Section 17) | All TTLs, timeouts, limits must come from agent.yaml |
| Expanded acceptance criteria | MCP lifecycle, autonomous guardrails, config validation |