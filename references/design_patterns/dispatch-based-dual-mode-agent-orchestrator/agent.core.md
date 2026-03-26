# agent.core — Runtime and Canonical Contract Specification

**Version:** 2.0.0  
**Status:** Authoritative — Behavioral Specification  
**Scope:** Runtime Engine, MCP Client, Execution Semantics, Canonical Contracts, Observability  
**Parent:** technical-design-specification.md

---

## 1. Purpose

agent.core is the behavioral engine of the platform. It owns execution — nothing else does.

Every run that enters the system is created, driven, and completed by agent.core. No other module may initiate execution, transition run state, or emit lifecycle events. All other modules either supply definitions (agent.workflows), supply external capabilities (agent.adapters), expose results (agent.api), or display them (agent.web).

agent.core is the single source of truth for:

- how runs are created and managed
- how workflows are executed step by step
- how the agent reasons and acts in autonomous mode
- how MCP tools are discovered and invoked
- how state transitions occur
- how events are emitted
- how errors are classified and propagated
- what canonical entities mean

---

## 2. What agent.core Owns vs Delegates

**Owns**

- AgentRuntime — the primary execution orchestrator
- MCPClient — sole mechanism for all external tool interactions
- DispatchEngine — routes requests to workflow or autonomous execution path
- Run lifecycle and state machine
- Step lifecycle per step type
- Run context object (shared state across steps)
- Approval coordination
- Canonical contracts (agent/core/contracts/)
- Event model and event emission
- Error model and diagnostic context
- Logfire instrumentation and trace propagation
- Configuration model and startup validation (agent.yaml → AgentConfig)

**Does Not Own**

- Workflow and skill definitions → agent.workflows
- Workflow and skill registry → agent.workflows
- MCP server implementations → agent.adapters
- Storage and cache adapters → agent.adapters
- Normalization of external payloads → agent.adapters (MCP servers normalize before returning; agent.core validates against canonical schemas)
- HTTP interface and DTOs → agent.api
- UI → agent.web

---

## 3. Internal Component Map

```
agent.core
├── config/
│   ├── model.py               # AgentConfig Pydantic model (validates agent.yaml)
│   └── loader.py              # Load and validate agent.yaml at startup
├── runtime/
│   ├── agent_runtime.py       # Primary orchestrator
│   ├── dispatch_engine.py     # Request routing
│   ├── run_manager.py         # Run lifecycle
│   └── step_executor.py       # Step type dispatch
├── graph/
│   ├── workflow_graph.py      # Pydantic AI graph for workflow execution
│   ├── autonomous_graph.py    # Pydantic AI graph for autonomous execution
│   ├── nodes/
│   │   ├── automated_node.py
│   │   ├── agent_node.py
│   │   └── gate_node.py
│   └── state.py               # GraphState definition
├── mcp/
│   ├── client.py              # MCPClient — connection management, tool catalog
│   └── invoker.py             # Tool invocation with validation and tracing
├── contracts/
│   ├── run.py
│   ├── step.py
│   ├── tool_invocation.py
│   ├── tool_catalog_entry.py
│   ├── artifact.py
│   ├── approval.py
│   ├── event.py
│   ├── error.py
│   ├── run_context.py
│   └── asset.py
├── events/
│   └── emitter.py             # Structured event emission, Valkey buffer
└── observability/
    └── instrumentation.py     # Logfire setup, span helpers, trace propagation
```

---

## 4. Configuration

### 4.1 agent.yaml

All runtime configuration lives in `agent.yaml` at the project root, validated at startup against the `AgentConfig` Pydantic model. If validation fails, the runtime does not start.

`agent.yaml` defines:

- **identity** — agent name, description, deployment version
- **mcp_servers** — list of MCP server connections with `required` flag, timeouts, and health check intervals
- **registries** — paths to workflow and skill definition directories
- **storage** — MongoDB connection
- **cache** — Valkey connection and TTL configuration
- **observability** — Logfire project, environment, log level
- **execution** — default execution limits for workflow steps and autonomous mode guardrails

The config path can be overridden via the `AGENT_CONFIG_PATH` environment variable.

### 4.2 AgentConfig Model

AgentConfig is the Pydantic model that validates `agent.yaml`. It lives in `agent/core/config/model.py`. Key validation rules:

- At least one MCP server must be declared
- At least one MCP server must be marked `required: true`
- Registry directories must exist on disk
- All timeouts and limits must be positive integers within defined bounds

### 4.3 MCP Server Required Flag

Each MCP server in `agent.yaml` carries a `required` boolean (default: `true`):

- **required: true** — startup fails if this server is unreachable. The runtime cannot operate without it.
- **required: false** — the server is marked UNAVAILABLE in the tool catalog and startup continues. Workflows referencing tools from unavailable servers will fail at validation time, not at startup.

This replaces the prior ambiguity around MCP startup failure behavior with an explicit per-server policy.

---

## 5. Pydantic AI Graph Model

agent.core uses the Pydantic AI graph module as the execution substrate for both execution paths. This section defines how the graph is structured, what state it carries, and how nodes map to step types.

### 5.1 GraphState

GraphState is the single object that flows through the graph for a given run. It is the authoritative in-memory representation of a run in progress.

```python
class GraphState(BaseModel):
    run_id: str
    correlation_id: str
    workflow_id: str | None          # None in autonomous execution
    workflow_version: str | None     # Resolved version at run creation time
    active_skills: list[str]         # Skill IDs active for this run
    run_context: RunContext          # Shared state; steps read/write here
    current_step_index: int          # Workflow path only
    steps: list[StepDefinition]      # Workflow path only; empty in autonomous
    pending_approval: Approval | None
    status: RunStatus
    events: list[Event]              # Buffer for batch persistence; primary emission is to Valkey
    error: Error | None
```

GraphState is not persisted directly — it is the live execution state. The RunContext within it is persisted to MongoDB and cached in Valkey throughout execution.

The `events` list on GraphState is a secondary buffer used for batch persistence to MongoDB after step completion. The primary event path is immediate emission to Valkey for SSE delivery (see Section 13).

### 5.2 Workflow Graph

The workflow graph is constructed dynamically at run creation time from the resolved workflow definition. It is a linear directed graph — one node per step, edges determined by step order.

```
START → AutomatedNode | AgentNode | GateNode → ... → END
```

Node selection per step:

| Step Type  | Graph Node    |
|------------|---------------|
| AUTOMATED  | AutomatedNode |
| AGENT      | AgentNode     |
| HUMAN_GATE | GateNode      |

Each node receives the current GraphState, executes its logic, updates GraphState, emits events, and returns the updated state. The graph runner advances to the next node unless the run is paused (at a GateNode) or failed.

### 5.3 Autonomous Graph

The autonomous graph is a single-node loop. There are no predefined steps. The AgentNode runs continuously, invoking tools and reasoning until it determines the goal is satisfied or a configured limit is reached.

```
START → AgentNode (loop) → END
```

The AgentNode in autonomous mode has no step_index constraint. It reasons over the full tool catalog and active skills until it produces a final output.

**Autonomous execution is bounded by the limits defined in agent.yaml:**

- `autonomous_max_iterations` — hard cap on reasoning loop iterations
- `autonomous_max_tool_calls` — hard cap on total MCP tool invocations
- `autonomous_max_tokens` — hard cap on total LLM token spend

If any limit is reached, the run transitions to COMPLETED with a truncation indicator in the output, not FAILED — the agent did work, it just hit a boundary.

### 5.4 Node Contracts

Every graph node must:

- Accept GraphState as its sole input
- Return an updated GraphState
- Emit at minimum `step.started` and `step.completed` or `step.failed`
- Write its output to `GraphState.run_context` before returning
- Never perform direct external calls — all external interactions via MCPClient

---

## 6. AgentRuntime

AgentRuntime is the primary orchestrator. It is the entry point for all execution within agent.core.

### 6.1 Responsibilities

- Receive an execution request from the dispatch engine
- Create the Run canonical entity and persist it
- Initialize GraphState for the run
- Construct the appropriate Pydantic AI graph (workflow or autonomous)
- Drive graph execution
- Handle graph completion, failure, and pause states
- Emit `run.created`, `run.started`, `run.completed`, `run.failed` events
- Propagate correlation ID through all child operations

### 6.2 Startup Sequence

On application startup, AgentRuntime must:

1. Load and validate `agent.yaml` via AgentConfig
2. Initialize MCPClient and connect to all declared servers
   - **Required servers** (`required: true`): connection failure is fatal — startup aborts
   - **Optional servers** (`required: false`): connection failure is logged; server marked UNAVAILABLE in catalog
3. Build the tool catalog from all connected servers
4. Load the workflow and skill registry from agent.workflows (definitions loaded from directories specified in `agent.yaml`, registered into MongoDB)
5. Confirm all step-referenced skill IDs exist in the registry
6. Signal readiness — the API must not accept requests until this sequence completes

### 6.3 Execution Entry Point

```python
async def execute(request: ExecutionRequest) -> Run:
    run = await run_manager.create(request)
    state = GraphState.from_request(run, request)
    graph = graph_factory.build(state)
    await graph.run(state)
    return await run_manager.finalize(run.run_id)
```

---

## 7. Dispatch Engine

The dispatch engine is the first component to handle an incoming request. It makes one decision: which execution path to use.

### 7.1 Routing Logic

```
Incoming request
    ├── workflow_id present → WORKFLOW EXECUTION PATH
    └── workflow_id absent  → AUTONOMOUS EXECUTION PATH
```

No classifier. No heuristics. No confidence thresholds. The presence or absence of `workflow_id` is the sole routing signal. This is intentional — ambiguous routing is a caller error, not a runtime decision.

### 7.2 Request Validation

Before routing, the dispatch engine validates:

- If `workflow_id` is present: the workflow exists in the registry and its `input_schema` is satisfied
- If skills are specified: all `skill_id` values exist in the registry
- `correlation_id` is present (generated by the API layer if not supplied by the caller)

Validation failures return structured Error entities — they do not create Run entities.

---

## 8. Run Lifecycle

### 8.1 State Machine

```
CREATED → RUNNING → COMPLETED
                  → FAILED
                  → CANCELLED
         RUNNING → AWAITING_APPROVAL → RUNNING
```

### 8.2 State Definitions

| State              | Description                                           |
|--------------------|-------------------------------------------------------|
| CREATED            | Run entity exists; execution not yet started          |
| RUNNING            | Execution in progress                                 |
| AWAITING_APPROVAL  | Paused at a HUMAN_GATE step; resumable                |
| COMPLETED          | Terminal success                                      |
| FAILED             | Terminal failure; includes diagnostic context          |
| CANCELLED          | Externally cancelled before completion                |

### 8.3 Transition Rules

- All transitions must be recorded with a timestamp on the Run entity
- No transition may be skipped
- COMPLETED and FAILED are terminal — no further transitions permitted
- AWAITING_APPROVAL may only transition back to RUNNING via an approval resolution
- CANCELLED may be triggered from RUNNING or AWAITING_APPROVAL only

### 8.4 Run Canonical Entity

```python
class Run(BaseModel):
    run_id: str
    correlation_id: str
    workflow_id: str | None
    workflow_version: str | None      # Resolved version at run creation
    active_skills: list[str]
    status: RunStatus
    input: dict
    created_at: datetime
    started_at: datetime | None
    completed_at: datetime | None
    error: Error | None
    artifacts: list[Artifact]
    step_count: int
    current_step_index: int | None    # Workflow path only
```

The `workflow_version` field captures the exact version of the workflow definition resolved at run creation. This provides auditability without requiring a separate WorkflowExecution entity.

---

## 9. Run Context

The RunContext is the shared state object scoped to a single run. It accumulates step outputs throughout workflow execution and provides the mechanism by which later steps access earlier results.

### 9.1 Structure

```python
class RunContext(BaseModel):
    run_id: str
    inputs: dict                          # Original run input
    step_outputs: dict[str, Any]          # Keyed by step_id
    metadata: dict[str, Any]             # Arbitrary run-scoped metadata
    updated_at: datetime
```

### 9.2 Access Pattern

Steps read from `run_context.step_outputs[step_id]` and write their own output via:

```python
run_context.step_outputs[self.step_id] = output
```

The runtime — not the step — is responsible for writing outputs to the context after each step completes. Steps return their output; the runtime commits it.

### 9.3 Persistence

- RunContext is persisted to MongoDB after every step completion
- The active RunContext for a running execution is cached in Valkey
- On reconnection or resume (e.g. after approval), RunContext is loaded from Valkey first, MongoDB as fallback

---

## 10. Step Execution Model

### 10.1 Step Types

**AUTOMATED**

Deterministic execution. No LLM involvement.

- Invokes one or more MCP tools via MCPClient
- Input is resolved from RunContext per the step's `input_mapping`
- Output is a structured result written to RunContext
- Must complete within the configured `step_timeout_seconds` (default from agent.yaml, overridable per step)
- Errors are classified as TOOL_FAILURE or VALIDATION_FAILURE

**AGENT**

LLM-driven reasoning. May invoke MCP tools.

- Activates the skill(s) referenced in the step definition
- Constructs a prompt from the active skill's `system_prompt` plus resolved step input
- Runs the Pydantic AI agent with the active tool catalog (filtered by skill `tool_restrictions` if set)
- The agent reasons, selects tools, invokes them via MCPClient, integrates results
- Produces output conforming to the step's `output_schema`
- Bounded by `max_tool_calls_per_step` and `max_tokens_per_step` (defaults from agent.yaml, overridable per step)

**HUMAN_GATE**

Pauses execution pending human review.

- Creates an Approval entity with status PENDING
- Transitions run to AWAITING_APPROVAL
- Emits `approval.requested` event
- Suspends graph execution — no timeout by default (configurable)
- On resolution: emits `approval.resolved`, transitions run back to RUNNING, resumes graph from next step

### 10.2 Step Input Resolution

Each step definition may specify an `input_mapping` — a declarative description of how to populate the step's input from the RunContext. The runtime resolves this mapping before the step executes.

```yaml
# Example step input_mapping in workflow definition
input_mapping:
  asset_list: steps.fetch_assets.output.assets
  scan_policy: inputs.policy_id
```

If no `input_mapping` is specified, the full RunContext is passed to the step.

### 10.3 Step Canonical Entity

```python
class Step(BaseModel):
    step_id: str
    run_id: str
    step_type: StepType
    name: str
    status: StepStatus
    input: dict
    output: dict | None
    skill_ids: list[str]             # AGENT steps only
    started_at: datetime | None
    completed_at: datetime | None
    error: Error | None
    tool_invocations: list[ToolInvocation]
```

---

## 11. MCP Client

MCPClient is the sole mechanism for all external tool interactions. It is initialized at startup and shared across all executions.

### 11.1 Startup Sequence

1. Read MCP server declarations from the validated AgentConfig
2. Connect to each server (concurrent connections, per-server timeout from config)
3. For required servers: fail startup on connection failure
4. For optional servers: log the error, mark the server as UNAVAILABLE, continue
5. For each connected server: discover available tools and their schemas
6. Build the tool catalog — unified index across all servers
7. Register the tool catalog with the Pydantic AI agent configuration
8. Log catalog summary to Logfire (server count, tool count per server, unavailable servers)
9. Begin health check loop (per-server interval from config)

### 11.2 Normalization Boundary

MCP servers (implemented in agent.adapters) are responsible for normalizing external vendor payloads into canonical contract shapes before returning responses. The MCP client in agent.core validates that responses conform to canonical schemas but does not perform normalization itself.

This means:
- agent.adapters owns the mapping from vendor-specific shapes to canonical entities
- agent.core owns the validation that canonical contracts are satisfied
- If a response fails validation, it is classified as OUTPUT_VALIDATION_FAILURE

### 11.3 Tool Catalog

The tool catalog is built at startup and refreshed on server reconnection.

```python
class ToolCatalogEntry(BaseModel):
    tool_id: str                  # Stable identifier: {server_id}.{tool_name}
    server_id: str
    name: str
    description: str
    input_schema: dict            # JSON Schema
    output_schema: dict           # JSON Schema
    available: bool
```

ToolCatalogEntry is defined in `agent/core/contracts/tool_catalog_entry.py` alongside all other canonical contracts.

The catalog is exposed at runtime via `GET /tools`.

### 11.4 Tool Invocation Contract

Every tool invocation must go through `MCPClient.invoke()`. Direct MCP server calls from anywhere else in agent.core or any other module are prohibited.

```python
async def invoke(
    tool_id: str,
    input: dict,
    run_id: str,
    step_id: str,
    correlation_id: str
) -> ToolInvocationResult:
    ...
```

Invocation steps:

1. Validate `tool_id` exists in catalog and is available
2. Validate input against the tool's `input_schema`
3. Emit `tool.invoked` event
4. Execute via Pydantic AI MCP client
5. Validate output against `output_schema`
6. Emit `tool.result` event
7. Return ToolInvocationResult

### 11.5 Error Classification

| Error Class              | Condition                                        |
|--------------------------|--------------------------------------------------|
| CONNECTION_FAILURE       | Server unreachable at invocation time             |
| TOOL_NOT_FOUND           | tool_id absent from catalog                       |
| INPUT_VALIDATION_FAILURE | Input does not satisfy input_schema               |
| TOOL_EXECUTION_FAILURE   | Server returned an error response                 |
| OUTPUT_VALIDATION_FAILURE| Output does not satisfy output_schema             |
| TIMEOUT                  | Server did not respond within configured timeout  |

### 11.6 ToolInvocation Canonical Entity

```python
class ToolInvocation(BaseModel):
    invocation_id: str
    tool_id: str
    server_id: str
    run_id: str
    step_id: str
    input: dict
    output: dict | None
    error: Error | None
    started_at: datetime
    completed_at: datetime | None
    latency_ms: int | None
```

---

## 12. Skill Activation Model

Skills are resolved by agent.core at execution time. They are defined in agent.workflows but applied here.

### 12.1 Activation

Skills are activated per-run or per-step:

- **Run-level**: specified in the `ExecutionRequest.active_skills` list; apply to all agent steps unless overridden
- **Step-level**: specified in the step definition's `skill_ids` list; override run-level skills for that step

### 12.2 Application

When an AGENT step executes, the runtime:

1. Resolves the active skill(s) for the step
2. Concatenates their `system_prompt` values (ordered by the `skill_ids` list on the step definition, or the `active_skills` list on the run for run-level skills)
3. Applies their `output_format` constraints to the step's `output_schema`
4. Applies their `tool_restrictions` to filter the available tool catalog for this step
5. Passes the composed system prompt and filtered tool catalog to the Pydantic AI agent

### 12.3 Autonomous Mode

In autonomous mode with no predefined steps, run-level skills govern the agent's behavior for the entire execution. If no skills are specified, the agent operates with the base system prompt only — no output format constraints, full tool catalog access.

Autonomous execution is subject to the guardrails defined in `agent.yaml` under `execution` (see Section 5.3).

---

## 13. Event Model

### 13.1 Emission Rules

- Every run state transition emits a run-level event
- Every step emits `step.started` at entry and `step.completed` or `step.failed` at exit
- Every MCP tool invocation emits `tool.invoked` and `tool.result`
- Approval lifecycle emits `approval.requested` and `approval.resolved`
- LLM streaming emits `llm.token` per token (configurable — may be suppressed)
- No event may be emitted without a `run_id`

### 13.2 Event Canonical Entity

```python
class Event(BaseModel):
    event_id: str
    event_type: EventType
    run_id: str
    step_id: str | None
    timestamp: datetime
    sequence: int                 # Monotonically increasing per run
    payload: dict
    correlation_id: str
```

### 13.3 Event Types

| Event Type            | Trigger                                      |
|-----------------------|----------------------------------------------|
| run.created           | Run entity created                           |
| run.started           | Graph execution begins                       |
| run.completed         | Graph reaches END node successfully           |
| run.failed            | Graph reaches END node via failure path       |
| run.cancelled         | External cancellation received               |
| run.awaiting_approval | GateNode suspends execution                  |
| step.started          | Node begins execution                        |
| step.completed        | Node completes successfully                  |
| step.failed           | Node execution fails                         |
| tool.invoked          | MCPClient.invoke() called                    |
| tool.result           | MCPClient.invoke() returns                   |
| llm.token             | Streaming token received from LLM            |
| approval.requested    | Approval entity created                      |
| approval.resolved     | Approval resolution received                 |

### 13.4 SSE Buffer

Events are written to Valkey in sequence order immediately upon emission. This is the primary event delivery path — the SSE endpoint in agent.api reads from this buffer.

The `events` list on GraphState is a secondary buffer used for batch persistence to MongoDB after step completion. It is not the primary delivery mechanism.

Events are retained in Valkey for a configurable window (default from `agent.yaml`: `sse_buffer_ttl_seconds`, default 10 minutes) to support client reconnection with `?since_sequence=N`.

---

## 14. Approval System

### 14.1 GateNode Behavior

When the workflow graph reaches a GateNode:

1. Create Approval entity with status PENDING
2. Write Approval to MongoDB
3. Emit `run.awaiting_approval` and `approval.requested` events
4. Transition run to AWAITING_APPROVAL
5. Suspend graph execution — the Pydantic AI graph is paused, not terminated

Graph state is preserved in Valkey during the pause (TTL: `paused_graph_ttl_seconds` from agent.yaml, default 7 days). On resume, the graph continues from the node immediately following the GateNode.

### 14.2 Resolution

Resolution arrives via `POST /runs/{run_id}/approvals/{approval_id}/resolve`.

On receipt:

1. Validate the approval exists and is in PENDING state
2. Update Approval entity with resolution payload and `resolved_by`
3. Transition Approval to APPROVED, REJECTED, or MODIFIED
4. Emit `approval.resolved` event
5. Transition run back to RUNNING
6. Resume graph from the node following the GateNode

If REJECTED: the run transitions to FAILED with a structured error indicating rejection reason. If MODIFIED: the resolution payload is written to RunContext and the next step receives the modified data.

### 14.3 Approval Canonical Entity

```python
class Approval(BaseModel):
    approval_id: str
    run_id: str
    step_id: str
    status: ApprovalStatus          # PENDING | APPROVED | REJECTED | MODIFIED
    context: dict                   # Data presented to reviewer
    resolution: dict | None         # Present on resolve
    resolved_by: str | None
    requested_at: datetime
    resolved_at: datetime | None
```

---

## 15. Error Model

### 15.1 Error Canonical Entity

```python
class Error(BaseModel):
    error_id: str
    error_class: ErrorClass
    message: str
    detail: dict                    # Structured diagnostic context
    run_id: str | None
    step_id: str | None
    tool_id: str | None
    occurred_at: datetime
    retryable: bool
```

### 15.2 Error Classes

| Class              | Description                                           |
|--------------------|-------------------------------------------------------|
| VALIDATION_FAILURE | Input or output schema violation                      |
| TOOL_FAILURE       | MCP tool execution error (see Section 11.5)           |
| LLM_FAILURE        | Anthropic API error or refusal                        |
| WORKFLOW_FAILURE    | Workflow definition error (missing step, bad reference)|
| APPROVAL_REJECTED  | Human gate rejected the run                           |
| TIMEOUT            | Step or tool exceeded configured time limit            |
| RUNTIME_ERROR      | Unexpected internal error                             |

### 15.3 Propagation Rules

- Step failure emits `step.failed` and writes the Error to the Step entity
- By default, step failure transitions the run to FAILED
- Workflows may declare `continue_on_failure: true` at the step level — in this case the failure is recorded in RunContext but execution continues to the next step
- Run failure emits `run.failed` with the Error entity in the payload

---

## 16. Asset Canonical Entity

An Asset represents an external resource referenced during a run — a host, endpoint, user account, vulnerability, or any other entity from an integrated system that the agent reasons about or acts upon.

```python
class Asset(BaseModel):
    asset_id: str                     # Platform-assigned stable identifier
    external_id: str                  # Identifier from the source system
    source: str                       # MCP server_id that provided this asset
    asset_type: str                   # Freeform type (e.g., "host", "endpoint", "user", "vulnerability")
    name: str                         # Human-readable display name
    attributes: dict[str, Any]        # Source-specific attributes (normalized by adapter)
    last_seen: datetime
    run_id: str | None                # Run that first surfaced this asset (if applicable)
```

Assets are created by agent steps when external data is retrieved and are persisted to MongoDB in the `assets` collection. They are referenced by run_id to maintain traceability. The `attributes` dict carries source-specific data that has been normalized by the adapter (MCP server) before reaching agent.core.

---

## 17. Canonical Contracts

All canonical contracts live in `agent/core/contracts/`. No other module may redefine these entities.

| Contract          | File                     | Description                                   |
|-------------------|--------------------------|-----------------------------------------------|
| Run               | run.py                   | Top-level execution instance                  |
| Step              | step.py                  | Unit of execution                             |
| RunContext        | run_context.py           | Shared state across steps                     |
| ToolInvocation    | tool_invocation.py       | Discrete MCP tool call                        |
| ToolCatalogEntry  | tool_catalog_entry.py    | Tool discovered from MCP server               |
| Artifact          | artifact.py              | Produced output persisted from a run          |
| Approval          | approval.py              | Human-gate decision                           |
| Event             | event.py                 | Structured runtime emission                   |
| Error             | error.py                 | Structured failure with diagnostic context    |
| Asset             | asset.py                 | External resource referenced in a run         |

---

## 18. Observability

### 18.1 Logfire Instrumentation

Logfire is configured once in `observability/instrumentation.py` and imported by all components. Pydantic AI's native Logfire integration is enabled at initialization. Logfire project and environment are read from the validated AgentConfig.

All spans follow the naming convention: `agent.core.{component}.{operation}`

Examples:

- `agent.core.runtime.execute`
- `agent.core.step.automated.execute`
- `agent.core.step.agent.execute`
- `agent.core.mcp.invoke`
- `agent.core.approval.request`

### 18.2 Required Instrumentation Points

| Component             | Instrumentation                                                   |
|-----------------------|-------------------------------------------------------------------|
| AgentRuntime.execute  | Span wrapping full run; run_id, correlation_id as attributes      |
| AutomatedNode         | Span per execution; tool IDs invoked as attributes                |
| AgentNode             | Span per execution; skill IDs, token count as attributes          |
| GateNode              | Span from pause to resume; approval_id as attribute               |
| MCPClient.invoke      | Span per tool call; tool_id, server_id, latency_ms as attributes  |
| DispatchEngine.route  | Span; execution path and workflow_id as attributes                |

### 18.3 structlog Integration

All log statements use structlog with bound context. At minimum, `run_id` and `correlation_id` must be bound to the logger at run creation and propagated through all child operations.

```python
log = structlog.get_logger().bind(run_id=run.run_id, correlation_id=run.correlation_id)
```

---

## 19. Persistence

### 19.1 MongoDB Collections

| Collection    | Contents                                  |
|---------------|-------------------------------------------|
| runs          | Run entities                              |
| steps         | Step entities, indexed by run_id          |
| events        | Event entities, indexed by run_id and sequence |
| run_contexts  | RunContext entities, indexed by run_id    |
| approvals     | Approval entities, indexed by run_id      |
| artifacts     | Artifact entities, indexed by run_id      |
| assets        | Asset entities, indexed by run_id and source |

All writes must be idempotent. `run_id` + `sequence` is the natural idempotency key for events.

### 19.2 Valkey Keys

| Key Pattern                  | Contents                                      | TTL                                    |
|------------------------------|-----------------------------------------------|----------------------------------------|
| run:{run_id}:context         | Serialized RunContext                          | From config: run_context_ttl_seconds   |
| run:{run_id}:events          | Ordered event list for SSE buffer              | From config: sse_buffer_ttl_seconds    |
| run:{run_id}:state           | Current RunStatus                              | From config: run_context_ttl_seconds   |
| run:{run_id}:graph           | Serialized paused graph state (AWAITING_APPROVAL) | From config: paused_graph_ttl_seconds |

All TTL values are configured in `agent.yaml` under the `cache` section.

---

## 20. Boundary Rules

agent.core enforces these rules absolutely:

- No direct HTTP calls to external services — all external interactions via MCPClient
- No import of agent.api, agent.web, or their DTOs
- No duplication of canonical contracts — all system-wide entities defined here
- No workflow or skill definitions stored here — those belong to agent.workflows
- MCPClient is the only component that may call Pydantic AI's MCP client primitives
- agent.core validates canonical schemas on MCP responses but does not normalize — normalization is the adapter's responsibility

---

## 21. Extension Points

**To add a new step type:**

1. Define the step type in StepType enum
2. Implement a new graph node in `graph/nodes/`
3. Register the node in `graph/workflow_graph.py` node selector
4. Add step-level event types to EventType if needed
5. Document in agent.core.md

**To support a new execution mode:**

1. Define the mode in DispatchEngine
2. Implement a new Pydantic AI graph in `graph/`
3. Register graph construction in graph_factory
4. Update dispatch routing logic

---

## 22. Implementation Notes for Claude Code

- Load and validate `agent.yaml` via AgentConfig before any other initialization
- Initialize MCPClient before AgentRuntime — the runtime depends on the tool catalog
- Respect the `required` flag on MCP servers: required servers block startup, optional servers degrade gracefully
- GraphState must be fully serializable — Pydantic AI requires this for graph pause/resume
- All async — no synchronous blocking anywhere in agent.core
- The Pydantic AI graph for workflow execution is constructed per-run, not shared
- The autonomous graph is a singleton template; a fresh state object is created per run
- RunContext writes to Valkey are fire-and-forget with error logging — they must not block execution
- MongoDB writes for Step and Event entities are awaited — they are the audit record
- `correlation_id` is generated by agent.api if not provided by the caller and must be present on every entity created within a run
- All Valkey TTLs are read from the validated AgentConfig — no hardcoded defaults in runtime code

---

## Appendix: Changes from v1.0.0

| Change | Rationale |
|--------|-----------|
| Replaced all AGENT.md references with agent.yaml + AgentConfig | Structured, validated config replaces ambiguous markdown parsing |
| Added `required` flag per MCP server | Resolves startup contradiction: required servers fail startup, optional servers degrade |
| Added config/ directory with model.py and loader.py | Configuration validation is a first-class concern |
| Moved ToolCatalogEntry to contracts/tool_catalog_entry.py | All canonical contracts in contracts/ — no exceptions |
| Defined Asset entity (Section 16) | Was listed but never defined |
| Added workflow_version to Run and GraphState | Captures resolved version for auditability; replaces WorkflowExecution entity |
| Added autonomous mode guardrails (Section 5.3) | max_iterations, max_tool_calls, max_tokens — all configurable in agent.yaml |
| Clarified normalization boundary (Section 11.2) | Adapters normalize, core validates |
| Clarified GraphState.events as secondary buffer (Section 5.1, 13.4) | Primary emission is to Valkey; in-memory list is for batch MongoDB persistence |
| Clarified skill composition order (Section 12.2) | Order follows skill_ids list on step or active_skills on run |
| All Valkey TTLs reference agent.yaml config | No hardcoded values in runtime code |