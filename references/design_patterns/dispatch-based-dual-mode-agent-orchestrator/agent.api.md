# agent.api — HTTP Interface Specification

**Version:** 1.0.0  
**Status:** Authoritative — Behavioral Specification  
**Scope:** HTTP Endpoints, DTOs, SSE Streaming, Authentication, Request Context  
**Parent:** technical-design-specification.md

---

## 1. Purpose

agent.api is the external interface of the platform. It owns the HTTP surface — nothing else is exposed to callers.

Every interaction with the platform from outside the process boundary arrives through agent.api. It translates between the external world (HTTP requests, SSE connections) and the internal world (canonical contracts, agent.core execution). It is a thin transport layer: it validates input, delegates to agent.core, and formats output. It does not make execution decisions.

agent.api is the single source of truth for:

- what HTTP endpoints exist and their contracts
- how requests are validated and translated to internal calls
- how canonical entities are projected to external DTOs
- how SSE streaming is managed
- how authentication and authorization are enforced
- how correlation IDs are generated and propagated
- what error responses look like

---

## 2. What agent.api Owns vs Delegates

**Owns**

- FastAPI application and router configuration
- HTTP endpoint definitions and OpenAPI documentation
- Request/response DTOs (Data Transfer Objects)
- Request validation (beyond what Pydantic handles automatically)
- Correlation ID generation (when not provided by the caller)
- SSE connection lifecycle and event delivery
- Authentication middleware
- HTTP error response formatting
- Rate limiting (if applicable)

**Does Not Own**

- Execution logic → agent.core
- Run/step lifecycle management → agent.core
- Canonical contract definitions → agent.core
- Workflow/skill registry operations → agent.workflows (accessed via agent.core)
- Storage/cache operations → agent.adapters (accessed via agent.core)
- Normalization of external payloads → agent.adapters
- UI → agent.web

---

## 3. Internal Component Map

```
agent.api
├── app.py                         # FastAPI application factory
├── routers/
│   ├── runs.py                    # Run execution and management endpoints
│   ├── approvals.py               # Approval resolution endpoints
│   ├── stream.py                  # SSE streaming endpoint
│   ├── workflows.py               # Workflow discovery endpoints
│   ├── skills.py                  # Skill discovery endpoints
│   ├── tools.py                   # Tool catalog endpoint
│   └── health.py                  # Health check endpoint
├── dtos/
│   ├── requests.py                # Inbound request DTOs
│   ├── responses.py               # Outbound response DTOs
│   └── events.py                  # SSE event envelope DTO
├── middleware/
│   ├── correlation.py             # Correlation ID generation/propagation
│   ├── auth.py                    # Authentication middleware
│   └── error_handler.py          # Global exception → HTTP error mapping
└── dependencies.py                # FastAPI dependency injection (runtime, adapters)
```

---

## 4. DTO Strategy

DTOs are the transport shapes used at the API boundary. They exist to decouple the external HTTP contract from internal canonical entities. DTOs are never used inside agent.core or agent.adapters.

### 4.1 Principle

```
Inbound:  Request DTO → validated → mapped to internal call (ExecutionRequest, etc.)
Outbound: Canonical entity → projected to Response DTO → serialized to JSON
```

DTOs may omit internal fields (e.g., `correlation_id` is not in the response unless requested), rename fields for API clarity, or flatten nested structures. The mapping between DTOs and canonical entities is explicit — no automatic passthrough.

### 4.2 Request DTOs

```python
class CreateRunRequest(BaseModel):
    """Inbound request to create and execute a run."""
    workflow_id: str | None = None
    input: dict
    skills: list[str] = []
    correlation_id: str | None = None    # Optional; generated if absent

class ResolveApprovalRequest(BaseModel):
    """Inbound request to resolve a pending approval."""
    resolution: str                       # "APPROVED" | "REJECTED" | "MODIFIED"
    payload: dict = {}                    # Decision data (required for MODIFIED)
    resolved_by: str
```

### 4.3 Response DTOs

```python
class RunResponse(BaseModel):
    """Outbound representation of a run."""
    run_id: str
    workflow_id: str | None
    workflow_version: str | None
    status: str
    input: dict
    created_at: datetime
    started_at: datetime | None
    completed_at: datetime | None
    step_count: int
    current_step_index: int | None
    error: ErrorResponse | None
    artifacts: list[ArtifactResponse]

class StepResponse(BaseModel):
    """Outbound representation of a step."""
    step_id: str
    run_id: str
    step_type: str
    name: str
    status: str
    started_at: datetime | None
    completed_at: datetime | None
    error: ErrorResponse | None
    tool_invocations: list[ToolInvocationResponse]

class ErrorResponse(BaseModel):
    """Outbound representation of an error."""
    error_id: str
    error_class: str
    message: str
    detail: dict
    retryable: bool

class ApprovalResponse(BaseModel):
    """Outbound representation of an approval."""
    approval_id: str
    run_id: str
    step_id: str
    status: str
    context: dict
    resolution: dict | None
    resolved_by: str | None
    requested_at: datetime
    resolved_at: datetime | None

class ArtifactResponse(BaseModel):
    """Outbound representation of an artifact."""
    artifact_id: str
    run_id: str
    name: str
    content_type: str
    created_at: datetime

class ToolInvocationResponse(BaseModel):
    """Outbound representation of a tool invocation."""
    invocation_id: str
    tool_id: str
    server_id: str
    started_at: datetime
    completed_at: datetime | None
    latency_ms: int | None
    error: ErrorResponse | None

class WorkflowSummaryResponse(BaseModel):
    """Outbound representation of a workflow in list endpoints."""
    workflow_id: str
    name: str
    version: str
    description: str
    tags: list[str]

class SkillSummaryResponse(BaseModel):
    """Outbound representation of a skill in list endpoints."""
    skill_id: str
    name: str
    version: str
    description: str

class ToolCatalogEntryResponse(BaseModel):
    """Outbound representation of a tool in the catalog."""
    tool_id: str
    server_id: str
    name: str
    description: str
    available: bool

class HealthResponse(BaseModel):
    """Health check response."""
    status: str                           # "healthy" | "degraded" | "unhealthy"
    mcp_servers: dict[str, str]           # server_id → "connected" | "unavailable"
    storage: str                          # "connected" | "unavailable"
    cache: str                            # "connected" | "unavailable"
```

---

## 5. Endpoint Specification

### 5.1 Runs

**Create and execute a run**

```
POST /runs
```

Request body: `CreateRunRequest`

Behavior:
1. Validate request
2. Generate `correlation_id` if not provided
3. Delegate to agent.core `AgentRuntime.execute()` via the dispatch engine
4. Return `RunResponse` with status CREATED (execution proceeds asynchronously)

Response: `201 Created` → `RunResponse`

The run begins execution immediately after creation. The caller monitors progress via the SSE stream.

**Get a run**

```
GET /runs/{run_id}
```

Response: `200 OK` → `RunResponse`

**List runs**

```
GET /runs?status={status}&limit={limit}&offset={offset}
```

Query parameters:
- `status` (optional): filter by RunStatus
- `limit` (default: 50, max: 200)
- `offset` (default: 0)

Response: `200 OK` → `list[RunResponse]`

**Cancel a run**

```
POST /runs/{run_id}/cancel
```

Behavior:
1. Validate run exists and is in a cancellable state (RUNNING or AWAITING_APPROVAL)
2. Delegate to agent.core to transition run to CANCELLED
3. Return updated `RunResponse`

Response: `200 OK` → `RunResponse`

**Get steps for a run**

```
GET /runs/{run_id}/steps
```

Response: `200 OK` → `list[StepResponse]`

**Get artifacts for a run**

```
GET /runs/{run_id}/artifacts
```

Response: `200 OK` → `list[ArtifactResponse]`

### 5.2 Approvals

**Resolve an approval**

```
POST /runs/{run_id}/approvals/{approval_id}/resolve
```

Request body: `ResolveApprovalRequest`

Behavior:
1. Validate approval exists, belongs to the run, and is in PENDING state
2. Delegate to agent.core approval system
3. Return updated `ApprovalResponse`

Response: `200 OK` → `ApprovalResponse`

**List approvals for a run**

```
GET /runs/{run_id}/approvals
```

Response: `200 OK` → `list[ApprovalResponse]`

### 5.3 Streaming

**Stream run events**

```
GET /runs/{run_id}/stream?since_sequence={N}
Content-Type: text/event-stream
```

Query parameters:
- `since_sequence` (optional, default: 0): resume from a specific sequence number

Behavior:
1. Validate run exists
2. If `since_sequence` is provided, load buffered events from Valkey (or MongoDB fallback)
3. Deliver all buffered events with sequence > `since_sequence`
4. Stream new events in real time as they are emitted
5. Send keepalive comments every 15 seconds to prevent connection timeout
6. Close the stream when the run reaches a terminal state (COMPLETED, FAILED, CANCELLED)

Event format:
```
event: {event_type}
data: {"event_type":"step.started","run_id":"...","step_id":"...","timestamp":"...","sequence":5,"payload":{...}}

```

Each SSE event uses the `event_type` as the SSE event name and the full event envelope as the data field.

### 5.4 Workflows

**List workflows**

```
GET /workflows?tags={tag1,tag2}
```

Query parameters:
- `tags` (optional): comma-separated filter

Response: `200 OK` → `list[WorkflowSummaryResponse]`

**Get workflow detail**

```
GET /workflows/{workflow_id}?version={version}
```

Query parameters:
- `version` (optional): specific version; defaults to highest

Response: `200 OK` → full workflow definition (steps included)

### 5.5 Skills

**List skills**

```
GET /skills
```

Response: `200 OK` → `list[SkillSummaryResponse]`

**Get skill detail**

```
GET /skills/{skill_id}?version={version}
```

Query parameters:
- `version` (optional): specific version; defaults to highest

Response: `200 OK` → full skill definition (system_prompt included)

### 5.6 Tools

**List tool catalog**

```
GET /tools
```

Response: `200 OK` → `list[ToolCatalogEntryResponse]`

### 5.7 Health

**Health check**

```
GET /health
```

Response: `200 OK` → `HealthResponse`

The health endpoint reports the status of each infrastructure dependency:

- **healthy**: all required MCP servers connected, storage connected, cache connected
- **degraded**: all required MCP servers connected but one or more optional servers or cache unavailable
- **unhealthy**: any required MCP server disconnected or storage unavailable

---

## 6. Correlation ID

Every request to the platform carries a correlation ID that propagates through all internal operations, events, log entries, and Logfire spans.

### 6.1 Generation

- If the caller provides `correlation_id` in the request body, it is used as-is
- If the caller provides an `X-Correlation-ID` header, it is used as-is
- If neither is present, the correlation middleware generates a UUID v4
- Request body `correlation_id` takes precedence over the header

### 6.2 Propagation

- The correlation ID is set on the Run entity at creation
- It is bound to the structlog logger for all operations within the run
- It is included as an attribute on all Logfire spans
- It is present on every Event entity emitted during the run
- It is returned in the `X-Correlation-ID` response header

### 6.3 Middleware

```python
@app.middleware("http")
async def correlation_middleware(request: Request, call_next):
    correlation_id = (
        request.headers.get("X-Correlation-ID")
        or str(uuid.uuid4())
    )
    request.state.correlation_id = correlation_id
    response = await call_next(request)
    response.headers["X-Correlation-ID"] = correlation_id
    return response
```

Note: for `POST /runs`, the `correlation_id` in the request body (if present) overrides the header value. This is handled in the endpoint, not the middleware.

---

## 7. Error Handling

### 7.1 HTTP Error Mapping

All errors returned by agent.core are mapped to appropriate HTTP status codes. The global error handler catches exceptions and produces consistent error responses.

| Error Condition | HTTP Status | Body |
|----------------|-------------|------|
| Request validation failure | 400 Bad Request | Pydantic validation errors |
| Run/approval/workflow not found | 404 Not Found | `{"detail": "..."}` |
| Invalid state transition (e.g., resolve non-PENDING approval) | 409 Conflict | `{"detail": "..."}` |
| Run not cancellable (terminal state) | 409 Conflict | `{"detail": "..."}` |
| Authentication failure | 401 Unauthorized | `{"detail": "..."}` |
| Internal runtime error | 500 Internal Server Error | `ErrorResponse` |
| Agent.core execution failure | Returned in run status, not as HTTP error | Via SSE stream |

### 7.2 Execution Errors vs HTTP Errors

Execution failures (step failures, tool failures, LLM errors) are not HTTP errors. They are part of the run lifecycle and are communicated through:

- The run's `status` field (FAILED) and `error` field
- The SSE event stream (`step.failed`, `run.failed` events)

The `POST /runs` endpoint returns `201 Created` even if the run subsequently fails during execution. The caller monitors the SSE stream for outcome.

---

## 8. Authentication

### 8.1 Strategy

Authentication is implemented as middleware. The specific mechanism is deployment-dependent:

- **Development**: no authentication (configurable bypass)
- **Production**: API key validation via `Authorization: Bearer {key}` header

The auth middleware extracts the caller identity and attaches it to the request state. This identity is used for:

- `resolved_by` on approval resolutions
- Audit logging
- Future authorization decisions

### 8.2 Middleware

```python
@app.middleware("http")
async def auth_middleware(request: Request, call_next):
    if settings.auth_enabled:
        token = request.headers.get("Authorization", "").removeprefix("Bearer ").strip()
        if not token:
            return JSONResponse(status_code=401, content={"detail": "Missing API key"})
        identity = await validate_token(token)
        if not identity:
            return JSONResponse(status_code=401, content={"detail": "Invalid API key"})
        request.state.identity = identity
    else:
        request.state.identity = "anonymous"
    return await call_next(request)
```

### 8.3 Configuration

Authentication settings are not in `agent.yaml` (which is platform configuration). They are provided via environment variables:

```
AUTH_ENABLED=true
AUTH_API_KEYS=key1,key2,key3
```

This keeps authentication deployment-specific and out of the platform configuration.

---

## 9. SSE Implementation Details

### 9.1 Connection Management

- One SSE connection per run per client
- Connections are tracked in memory (not persisted)
- If a client disconnects and reconnects with `?since_sequence=N`, buffered events are replayed
- The buffer source is Valkey first, MongoDB fallback (consistent with agent.core spec)

### 9.2 Event Delivery

```python
async def stream_run_events(run_id: str, since_sequence: int = 0):
    # 1. Replay buffered events
    buffered = await valkey.get_events_since(run_id, since_sequence)
    if not buffered:
        buffered = await mongo.get_events_for_run(run_id, since_sequence)
    for event in buffered:
        yield format_sse(event)

    # 2. Stream live events
    last_sequence = buffered[-1].sequence if buffered else since_sequence
    while True:
        new_events = await valkey.get_events_since(run_id, last_sequence)
        for event in new_events:
            yield format_sse(event)
            last_sequence = event.sequence

        # Check for terminal state
        status = await valkey.get_run_status(run_id)
        if status in (RunStatus.COMPLETED, RunStatus.FAILED, RunStatus.CANCELLED):
            # Drain any final events
            final = await valkey.get_events_since(run_id, last_sequence)
            for event in final:
                yield format_sse(event)
            return

        # Keepalive
        yield ": keepalive\n\n"
        await asyncio.sleep(1)
```

### 9.3 SSE Format

```python
def format_sse(event: Event) -> str:
    data = event.model_dump_json()
    return f"event: {event.event_type}\ndata: {data}\n\n"
```

### 9.4 Backpressure

If a client cannot consume events fast enough:

- Events continue to buffer in Valkey (up to TTL)
- The SSE connection is not throttled — events are sent as fast as the client reads
- If the buffer overflo# agent.api — HTTP Interface Specification

**Version:** 1.0.0  
**Status:** Authoritative — Behavioral Specification  
**Scope:** HTTP Endpoints, DTOs, SSE Streaming, Authentication, Request Context  
**Parent:** technical-design-specification.md

---

## 1. Purpose

agent.api is the external interface of the platform. It owns the HTTP surface — nothing else is exposed to callers.

Every interaction with the platform from outside the process boundary arrives through agent.api. It translates between the external world (HTTP requests, SSE connections) and the internal world (canonical contracts, agent.core execution). It is a thin transport layer: it validates input, delegates to agent.core, and formats output. It does not make execution decisions.

agent.api is the single source of truth for:

- what HTTP endpoints exist and their contracts
- how requests are validated and translated to internal calls
- how canonical entities are projected to external DTOs
- how SSE streaming is managed
- how authentication and authorization are enforced
- how correlation IDs are generated and propagated
- what error responses look like

---

## 2. What agent.api Owns vs Delegates

**Owns**

- FastAPI application and router configuration
- HTTP endpoint definitions and OpenAPI documentation
- Request/response DTOs (Data Transfer Objects)
- Request validation (beyond what Pydantic handles automatically)
- Correlation ID generation (when not provided by the caller)
- SSE connection lifecycle and event delivery
- Authentication middleware
- HTTP error response formatting
- Rate limiting (if applicable)

**Does Not Own**

- Execution logic → agent.core
- Run/step lifecycle management → agent.core
- Canonical contract definitions → agent.core
- Workflow/skill registry operations → agent.workflows (accessed via agent.core)
- Storage/cache operations → agent.adapters (accessed via agent.core)
- Normalization of external payloads → agent.adapters
- UI → agent.web

---

## 3. Internal Component Map

```
agent.api
├── app.py                         # FastAPI application factory
├── routers/
│   ├── runs.py                    # Run execution and management endpoints
│   ├── approvals.py               # Approval resolution endpoints
│   ├── stream.py                  # SSE streaming endpoint
│   ├── workflows.py               # Workflow discovery endpoints
│   ├── skills.py                  # Skill discovery endpoints
│   ├── tools.py                   # Tool catalog endpoint
│   └── health.py                  # Health check endpoint
├── dtos/
│   ├── requests.py                # Inbound request DTOs
│   ├── responses.py               # Outbound response DTOs
│   └── events.py                  # SSE event envelope DTO
├── middleware/
│   ├── correlation.py             # Correlation ID generation/propagation
│   ├── auth.py                    # Authentication middleware
│   └── error_handler.py          # Global exception → HTTP error mapping
└── dependencies.py                # FastAPI dependency injection (runtime, adapters)
```

---

## 4. DTO Strategy

DTOs are the transport shapes used at the API boundary. They exist to decouple the external HTTP contract from internal canonical entities. DTOs are never used inside agent.core or agent.adapters.

### 4.1 Principle

```
Inbound:  Request DTO → validated → mapped to internal call (ExecutionRequest, etc.)
Outbound: Canonical entity → projected to Response DTO → serialized to JSON
```

DTOs may omit internal fields (e.g., `correlation_id` is not in the response unless requested), rename fields for API clarity, or flatten nested structures. The mapping between DTOs and canonical entities is explicit — no automatic passthrough.

### 4.2 Request DTOs

```python
class CreateRunRequest(BaseModel):
    """Inbound request to create and execute a run."""
    workflow_id: str | None = None
    input: dict
    skills: list[str] = []
    correlation_id: str | None = None    # Optional; generated if absent

class ResolveApprovalRequest(BaseModel):
    """Inbound request to resolve a pending approval."""
    resolution: str                       # "APPROVED" | "REJECTED" | "MODIFIED"
    payload: dict = {}                    # Decision data (required for MODIFIED)
    resolved_by: str
```

### 4.3 Response DTOs

```python
class RunResponse(BaseModel):
    """Outbound representation of a run."""
    run_id: str
    workflow_id: str | None
    workflow_version: str | None
    status: str
    input: dict
    created_at: datetime
    started_at: datetime | None
    completed_at: datetime | None
    step_count: int
    current_step_index: int | None
    error: ErrorResponse | None
    artifacts: list[ArtifactResponse]

class StepResponse(BaseModel):
    """Outbound representation of a step."""
    step_id: str
    run_id: str
    step_type: str
    name: str
    status: str
    started_at: datetime | None
    completed_at: datetime | None
    error: ErrorResponse | None
    tool_invocations: list[ToolInvocationResponse]

class ErrorResponse(BaseModel):
    """Outbound representation of an error."""
    error_id: str
    error_class: str
    message: str
    detail: dict
    retryable: bool

class ApprovalResponse(BaseModel):
    """Outbound representation of an approval."""
    approval_id: str
    run_id: str
    step_id: str
    status: str
    context: dict
    resolution: dict | None
    resolved_by: str | None
    requested_at: datetime
    resolved_at: datetime | None

class ArtifactResponse(BaseModel):
    """Outbound representation of an artifact."""
    artifact_id: str
    run_id: str
    name: str
    content_type: str
    created_at: datetime

class ToolInvocationResponse(BaseModel):
    """Outbound representation of a tool invocation."""
    invocation_id: str
    tool_id: str
    server_id: str
    started_at: datetime
    completed_at: datetime | None
    latency_ms: int | None
    error: ErrorResponse | None

class WorkflowSummaryResponse(BaseModel):
    """Outbound representation of a workflow in list endpoints."""
    workflow_id: str
    name: str
    version: str
    description: str
    tags: list[str]

class SkillSummaryResponse(BaseModel):
    """Outbound representation of a skill in list endpoints."""
    skill_id: str
    name: str
    version: str
    description: str

class ToolCatalogEntryResponse(BaseModel):
    """Outbound representation of a tool in the catalog."""
    tool_id: str
    server_id: str
    name: str
    description: str
    available: bool

class HealthResponse(BaseModel):
    """Health check response."""
    status: str                           # "healthy" | "degraded" | "unhealthy"
    mcp_servers: dict[str, str]           # server_id → "connected" | "unavailable"
    storage: str                          # "connected" | "unavailable"
    cache: str                            # "connected" | "unavailable"
```

---

## 5. Endpoint Specification

### 5.1 Runs

**Create and execute a run**

```
POST /runs
```

Request body: `CreateRunRequest`

Behavior:
1. Validate request
2. Generate `correlation_id` if not provided
3. Delegate to agent.core `AgentRuntime.execute()` via the dispatch engine
4. Return `RunResponse` with status CREATED (execution proceeds asynchronously)

Response: `201 Created` → `RunResponse`

The run begins execution immediately after creation. The caller monitors progress via the SSE stream.

**Get a run**

```
GET /runs/{run_id}
```

Response: `200 OK` → `RunResponse`

**List runs**

```
GET /runs?status={status}&limit={limit}&offset={offset}
```

Query parameters:
- `status` (optional): filter by RunStatus
- `limit` (default: 50, max: 200)
- `offset` (default: 0)

Response: `200 OK` → `list[RunResponse]`

**Cancel a run**

```
POST /runs/{run_id}/cancel
```

Behavior:
1. Validate run exists and is in a cancellable state (RUNNING or AWAITING_APPROVAL)
2. Delegate to agent.core to transition run to CANCELLED
3. Return updated `RunResponse`

Response: `200 OK` → `RunResponse`

**Get steps for a run**

```
GET /runs/{run_id}/steps
```

Response: `200 OK` → `list[StepResponse]`

**Get artifacts for a run**

```
GET /runs/{run_id}/artifacts
```

Response: `200 OK` → `list[ArtifactResponse]`

### 5.2 Approvals

**Resolve an approval**

```
POST /runs/{run_id}/approvals/{approval_id}/resolve
```

Request body: `ResolveApprovalRequest`

Behavior:
1. Validate approval exists, belongs to the run, and is in PENDING state
2. Delegate to agent.core approval system
3. Return updated `ApprovalResponse`

Response: `200 OK` → `ApprovalResponse`

**List approvals for a run**

```
GET /runs/{run_id}/approvals
```

Response: `200 OK` → `list[ApprovalResponse]`

### 5.3 Streaming

**Stream run events**

```
GET /runs/{run_id}/stream?since_sequence={N}
Content-Type: text/event-stream
```

Query parameters:
- `since_sequence` (optional, default: 0): resume from a specific sequence number

Behavior:
1. Validate run exists
2. If `since_sequence` is provided, load buffered events from Valkey (or MongoDB fallback)
3. Deliver all buffered events with sequence > `since_sequence`
4. Stream new events in real time as they are emitted
5. Send keepalive comments every 15 seconds to prevent connection timeout
6. Close the stream when the run reaches a terminal state (COMPLETED, FAILED, CANCELLED)

Event format:
```
event: {event_type}
data: {"event_type":"step.started","run_id":"...","step_id":"...","timestamp":"...","sequence":5,"payload":{...}}

```

Each SSE event uses the `event_type` as the SSE event name and the full event envelope as the data field.

### 5.4 Workflows

**List workflows**

```
GET /workflows?tags={tag1,tag2}
```

Query parameters:
- `tags` (optional): comma-separated filter

Response: `200 OK` → `list[WorkflowSummaryResponse]`

**Get workflow detail**

```
GET /workflows/{workflow_id}?version={version}
```

Query parameters:
- `version` (optional): specific version; defaults to highest

Response: `200 OK` → full workflow definition (steps included)

### 5.5 Skills

**List skills**

```
GET /skills
```

Response: `200 OK` → `list[SkillSummaryResponse]`

**Get skill detail**

```
GET /skills/{skill_id}?version={version}
```

Query parameters:
- `version` (optional): specific version; defaults to highest

Response: `200 OK` → full skill definition (system_prompt included)

### 5.6 Tools

**List tool catalog**

```
GET /tools
```

Response: `200 OK` → `list[ToolCatalogEntryResponse]`

### 5.7 Health

**Health check**

```
GET /health
```

Response: `200 OK` → `HealthResponse`

The health endpoint reports the status of each infrastructure dependency:

- **healthy**: all required MCP servers connected, storage connected, cache connected
- **degraded**: all required MCP servers connected but one or more optional servers or cache unavailable
- **unhealthy**: any required MCP server disconnected or storage unavailable

---

## 6. Correlation ID

Every request to the platform carries a correlation ID that propagates through all internal operations, events, log entries, and Logfire spans.

### 6.1 Generation

- If the caller provides `correlation_id` in the request body, it is used as-is
- If the caller provides an `X-Correlation-ID` header, it is used as-is
- If neither is present, the correlation middleware generates a UUID v4
- Request body `correlation_id` takes precedence over the header

### 6.2 Propagation

- The correlation ID is set on the Run entity at creation
- It is bound to the structlog logger for all operations within the run
- It is included as an attribute on all Logfire spans
- It is present on every Event entity emitted during the run
- It is returned in the `X-Correlation-ID` response header

### 6.3 Middleware

```python
@app.middleware("http")
async def correlation_middleware(request: Request, call_next):
    correlation_id = (
        request.headers.get("X-Correlation-ID")
        or str(uuid.uuid4())
    )
    request.state.correlation_id = correlation_id
    response = await call_next(request)
    response.headers["X-Correlation-ID"] = correlation_id
    return response
```

Note: for `POST /runs`, the `correlation_id` in the request body (if present) overrides the header value. This is handled in the endpoint, not the middleware.

---

## 7. Error Handling

### 7.1 HTTP Error Mapping

All errors returned by agent.core are mapped to appropriate HTTP status codes. The global error handler catches exceptions and produces consistent error responses.

| Error Condition | HTTP Status | Body |
|----------------|-------------|------|
| Request validation failure | 400 Bad Request | Pydantic validation errors |
| Run/approval/workflow not found | 404 Not Found | `{"detail": "..."}` |
| Invalid state transition (e.g., resolve non-PENDING approval) | 409 Conflict | `{"detail": "..."}` |
| Run not cancellable (terminal state) | 409 Conflict | `{"detail": "..."}` |
| Authentication failure | 401 Unauthorized | `{"detail": "..."}` |
| Internal runtime error | 500 Internal Server Error | `ErrorResponse` |
| Agent.core execution failure | Returned in run status, not as HTTP error | Via SSE stream |

### 7.2 Execution Errors vs HTTP Errors

Execution failures (step failures, tool failures, LLM errors) are not HTTP errors. They are part of the run lifecycle and are communicated through:

- The run's `status` field (FAILED) and `error` field
- The SSE event stream (`step.failed`, `run.failed` events)

The `POST /runs` endpoint returns `201 Created` even if the run subsequently fails during execution. The caller monitors the SSE stream for outcome.

---

## 8. Authentication

### 8.1 Strategy

Authentication is implemented as middleware. The specific mechanism is deployment-dependent:

- **Development**: no authentication (configurable bypass)
- **Production**: API key validation via `Authorization: Bearer {key}` header

The auth middleware extracts the caller identity and attaches it to the request state. This identity is used for:

- `resolved_by` on approval resolutions
- Audit logging
- Future authorization decisions

### 8.2 Middleware

```python
@app.middleware("http")
async def auth_middleware(request: Request, call_next):
    if settings.auth_enabled:
        token = request.headers.get("Authorization", "").removeprefix("Bearer ").strip()
        if not token:
            return JSONResponse(status_code=401, content={"detail": "Missing API key"})
        identity = await validate_token(token)
        if not identity:
            return JSONResponse(status_code=401, content={"detail": "Invalid API key"})
        request.state.identity = identity
    else:
        request.state.identity = "anonymous"
    return await call_next(request)
```

### 8.3 Configuration

Authentication settings are not in `agent.yaml` (which is platform configuration). They are provided via environment variables:

```
AUTH_ENABLED=true
AUTH_API_KEYS=key1,key2,key3
```

This keeps authentication deployment-specific and out of the platform configuration.

---

## 9. SSE Implementation Details

### 9.1 Connection Management

- One SSE connection per run per client
- Connections are tracked in memory (not persisted)
- If a client disconnects and reconnects with `?since_sequence=N`, buffered events are replayed
- The buffer source is Valkey first, MongoDB fallback (consistent with agent.core spec)

### 9.2 Event Delivery

```python
async def stream_run_events(run_id: str, since_sequence: int = 0):
    # 1. Replay buffered events
    buffered = await valkey.get_events_since(run_id, since_sequence)
    if not buffered:
        buffered = await mongo.get_events_for_run(run_id, since_sequence)
    for event in buffered:
        yield format_sse(event)

    # 2. Stream live events
    last_sequence = buffered[-1].sequence if buffered else since_sequence
    while True:
        new_events = await valkey.get_events_since(run_id, last_sequence)
        for event in new_events:
            yield format_sse(event)
            last_sequence = event.sequence

        # Check for terminal state
        status = await valkey.get_run_status(run_id)
        if status in (RunStatus.COMPLETED, RunStatus.FAILED, RunStatus.CANCELLED):
            # Drain any final events
            final = await valkey.get_events_since(run_id, last_sequence)
            for event in final:
                yield format_sse(event)
            return

        # Keepalive
        yield ": keepalive\n\n"
        await asyncio.sleep(1)
```

### 9.3 SSE Format

```python
def format_sse(event: Event) -> str:
    data = event.model_dump_json()
    return f"event: {event.event_type}\ndata: {data}\n\n"
```

### 9.4 Backpressure

If a client cannot consume events fast enough:

- Events continue to buffer in Valkey (up to TTL)
- The SSE connection is not throttled — events are sent as fast as the client reads
- If the buffer overflows (TTL expiry), the client must re-query via `GET /runs/{run_id}` and `GET /runs/{run_id}/steps` to reconstruct state

---

## 10. API Versioning

### 10.1 Strategy

The API uses URL path versioning:

```
/v1/runs
/v1/workflows
/v1/skills
...
```

All endpoints described in this document are under `/v1`. When breaking changes are introduced, a `/v2` prefix is added. Previous versions are maintained for a documented deprecation period.

### 10.2 Router Registration

```python
app = FastAPI(title="Agent Platform API", version="1.0.0")
app.include_router(runs_router, prefix="/v1")
app.include_router(approvals_router, prefix="/v1")
app.include_router(stream_router, prefix="/v1")
app.include_router(workflows_router, prefix="/v1")
app.include_router(skills_router, prefix="/v1")
app.include_router(tools_router, prefix="/v1")
app.include_router(health_router)          # Health is unversioned
```

---

## 11. Dependency Injection

agent.api uses FastAPI's dependency injection to access agent.core and agent.adapters.

### 11.1 Pattern

```python
from fastapi import Depends

async def get_runtime() -> AgentRuntime:
    """Provides the singleton AgentRuntime instance."""
    return app.state.runtime

async def get_registry() -> Registry:
    """Provides the singleton Registry instance."""
    return app.state.registry

@router.post("/runs", status_code=201)
async def create_run(
    request: CreateRunRequest,
    runtime: AgentRuntime = Depends(get_runtime),
    correlation_id: str = Depends(get_correlation_id),
) -> RunResponse:
    ...
```

### 11.2 Startup

The FastAPI application factory initializes all dependencies at startup and stores them on `app.state`:

```python
@app.on_event("startup")
async def startup():
    config = load_config()
    # agent.core initializes runtime, MCP client, tool catalog
    # agent.workflows loads registry
    # agent.adapters connects storage and cache
    app.state.runtime = runtime
    app.state.registry = registry
```

---

## 12. Observability

### 12.1 Request Tracing

Every HTTP request is traced via Logfire with:

- HTTP method and path
- Response status code
- Request latency
- Correlation ID as a span attribute

### 12.2 SSE Session Telemetry

Each SSE connection is tracked with:

- Connection duration
- Events delivered count
- Reconnection count (via `since_sequence` presence)
- Run ID as a span attribute

### 12.3 structlog Integration

All log statements bind `correlation_id` from the request state:

```python
log = structlog.get_logger().bind(correlation_id=request.state.correlation_id)
```

### 12.4 Span Naming

Span naming convention: `agent.api.{router}.{operation}`

Examples:
- `agent.api.runs.create`
- `agent.api.runs.get`
- `agent.api.stream.connect`
- `agent.api.approvals.resolve`
- `agent.api.health.check`

---

## 13. Boundary Rules

agent.api enforces these rules:

- **DTOs are the only shapes that cross the HTTP boundary** — canonical entities are never serialized directly to HTTP responses
- **No execution logic** — agent.api delegates all execution to agent.core
- **No direct storage or cache access** — all data access goes through agent.core or is injected via dependencies that wrap agent.adapters
- **No import of agent.web**
- **No import of agent.adapters directly** — access is via agent.core dependency injection
- **No canonical contract modification** — agent.api consumes canonical entities and projects them to DTOs
- **Correlation ID is generated here if not provided** — agent.core expects it to always be present
- **Authentication credentials and configuration are not in agent.yaml** — they use environment variables

---

## 14. Implementation Notes for Claude Code

- Use `fastapi.responses.StreamingResponse` with `media_type="text/event-stream"` for the SSE endpoint
- Use `asyncio.Queue` or polling for live event delivery in the SSE stream — choose based on complexity; polling against Valkey is simpler and sufficient for v1
- DTO → canonical and canonical → DTO mappings should be explicit factory methods, not automatic serialization
- The `POST /runs` endpoint should return immediately after run creation — execution is async. The caller monitors via SSE.
- All endpoints must return within a reasonable timeout — long-running operations are modeled as runs, not as HTTP requests
- OpenAPI schema is auto-generated by FastAPI from the DTO models — ensure descriptions and examples are present on DTO fields
- Health endpoint must not require authentication
- Rate limiting is out of scope for v1 but the middleware slot exists for future implementation
- CORS configuration should be included for agent.web (origins configurable via environment variable)

---

## 15. Example Request/Response Traces

### 15.1 Create and Monitor a Workflow Run

```
1. Client → POST /v1/runs
   Headers: Authorization: Bearer {key}
   Body: {
     "workflow_id": "classify-ttp",
     "input": {"description": "Attacker used PowerShell to download and execute a remote payload"},
     "skills": [],
     "correlation_id": "req-abc-123"
   }

   ← 201 Created
   Headers: X-Correlation-ID: req-abc-123
   Body: {
     "run_id": "run-001",
     "workflow_id": "classify-ttp",
     "workflow_version": "1.0.0",
     "status": "CREATED",
     "input": {"description": "..."},
     "created_at": "2026-03-24T12:00:00Z",
     "started_at": null,
     "completed_at": null,
     "step_count": 3,
     "current_step_index": 0,
     "error": null,
     "artifacts": []
   }

2. Client → GET /v1/runs/run-001/stream
   ← 200 OK (text/event-stream)

   event: run.started
   data: {"event_type":"run.started","run_id":"run-001","step_id":null,"timestamp":"...","sequence":1,"payload":{}}

   event: step.started
   data: {"event_type":"step.started","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":2,"payload":{"step_type":"AGENT","name":"Normalize Input"}}

   event: tool.invoked
   data: {"event_type":"tool.invoked","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":3,"payload":{"tool_id":"sentinel.run_query","input":{...}}}

   event: tool.result
   data: {"event_type":"tool.result","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":4,"payload":{"tool_id":"sentinel.run_query","latency_ms":230}}

   event: step.completed
   data: {"event_type":"step.completed","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":5,"payload":{"output":{...}}}

   ... (more steps) ...

   event: run.awaiting_approval
   data: {"event_type":"run.awaiting_approval","run_id":"run-001","step_id":"review","timestamp":"...","sequence":12,"payload":{"approval_id":"apr-001"}}

3. Client → POST /v1/runs/run-001/approvals/apr-001/resolve
   Body: {
     "resolution": "APPROVED",
     "payload": {},
     "resolved_by": "analyst@example.com"
   }
# agent.api — HTTP Interface Specification

**Version:** 1.0.0  
**Status:** Authoritative — Behavioral Specification  
**Scope:** HTTP Endpoints, DTOs, SSE Streaming, Authentication, Request Context  
**Parent:** technical-design-specification.md

---

## 1. Purpose

agent.api is the external interface of the platform. It owns the HTTP surface — nothing else is exposed to callers.

Every interaction with the platform from outside the process boundary arrives through agent.api. It translates between the external world (HTTP requests, SSE connections) and the internal world (canonical contracts, agent.core execution). It is a thin transport layer: it validates input, delegates to agent.core, and formats output. It does not make execution decisions.

agent.api is the single source of truth for:

- what HTTP endpoints exist and their contracts
- how requests are validated and translated to internal calls
- how canonical entities are projected to external DTOs
- how SSE streaming is managed
- how authentication and authorization are enforced
- how correlation IDs are generated and propagated
- what error responses look like

---

## 2. What agent.api Owns vs Delegates

**Owns**

- FastAPI application and router configuration
- HTTP endpoint definitions and OpenAPI documentation
- Request/response DTOs (Data Transfer Objects)
- Request validation (beyond what Pydantic handles automatically)
- Correlation ID generation (when not provided by the caller)
- SSE connection lifecycle and event delivery
- Authentication middleware
- HTTP error response formatting
- Rate limiting (if applicable)

**Does Not Own**

- Execution logic → agent.core
- Run/step lifecycle management → agent.core
- Canonical contract definitions → agent.core
- Workflow/skill registry operations → agent.workflows (accessed via agent.core)
- Storage/cache operations → agent.adapters (accessed via agent.core)
- Normalization of external payloads → agent.adapters
- UI → agent.web

---

## 3. Internal Component Map

```
agent.api
├── app.py                         # FastAPI application factory
├── routers/
│   ├── runs.py                    # Run execution and management endpoints
│   ├── approvals.py               # Approval resolution endpoints
│   ├── stream.py                  # SSE streaming endpoint
│   ├── workflows.py               # Workflow discovery endpoints
│   ├── skills.py                  # Skill discovery endpoints
│   ├── tools.py                   # Tool catalog endpoint
│   └── health.py                  # Health check endpoint
├── dtos/
│   ├── requests.py                # Inbound request DTOs
│   ├── responses.py               # Outbound response DTOs
│   └── events.py                  # SSE event envelope DTO
├── middleware/
│   ├── correlation.py             # Correlation ID generation/propagation
│   ├── auth.py                    # Authentication middleware
│   └── error_handler.py          # Global exception → HTTP error mapping
└── dependencies.py                # FastAPI dependency injection (runtime, adapters)
```

---

## 4. DTO Strategy

DTOs are the transport shapes used at the API boundary. They exist to decouple the external HTTP contract from internal canonical entities. DTOs are never used inside agent.core or agent.adapters.

### 4.1 Principle

```
Inbound:  Request DTO → validated → mapped to internal call (ExecutionRequest, etc.)
Outbound: Canonical entity → projected to Response DTO → serialized to JSON
```

DTOs may omit internal fields (e.g., `correlation_id` is not in the response unless requested), rename fields for API clarity, or flatten nested structures. The mapping between DTOs and canonical entities is explicit — no automatic passthrough.

### 4.2 Request DTOs

```python
class CreateRunRequest(BaseModel):
    """Inbound request to create and execute a run."""
    workflow_id: str | None = None
    input: dict
    skills: list[str] = []
    correlation_id: str | None = None    # Optional; generated if absent

class ResolveApprovalRequest(BaseModel):
    """Inbound request to resolve a pending approval."""
    resolution: str                       # "APPROVED" | "REJECTED" | "MODIFIED"
    payload: dict = {}                    # Decision data (required for MODIFIED)
    resolved_by: str
```

### 4.3 Response DTOs

```python
class RunResponse(BaseModel):
    """Outbound representation of a run."""
    run_id: str
    workflow_id: str | None
    workflow_version: str | None
    status: str
    input: dict
    created_at: datetime
    started_at: datetime | None
    completed_at: datetime | None
    step_count: int
    current_step_index: int | None
    error: ErrorResponse | None
    artifacts: list[ArtifactResponse]

class StepResponse(BaseModel):
    """Outbound representation of a step."""
    step_id: str
    run_id: str
    step_type: str
    name: str
    status: str
    started_at: datetime | None
    completed_at: datetime | None
    error: ErrorResponse | None
    tool_invocations: list[ToolInvocationResponse]

class ErrorResponse(BaseModel):
    """Outbound representation of an error."""
    error_id: str
    error_class: str
    message: str
    detail: dict
    retryable: bool

class ApprovalResponse(BaseModel):
    """Outbound representation of an approval."""
    approval_id: str
    run_id: str
    step_id: str
    status: str
    context: dict
    resolution: dict | None
    resolved_by: str | None
    requested_at: datetime
    resolved_at: datetime | None

class ArtifactResponse(BaseModel):
    """Outbound representation of an artifact."""
    artifact_id: str
    run_id: str
    name: str
    content_type: str
    created_at: datetime

class ToolInvocationResponse(BaseModel):
    """Outbound representation of a tool invocation."""
    invocation_id: str
    tool_id: str
    server_id: str
    started_at: datetime
    completed_at: datetime | None
    latency_ms: int | None
    error: ErrorResponse | None

class WorkflowSummaryResponse(BaseModel):
    """Outbound representation of a workflow in list endpoints."""
    workflow_id: str
    name: str
    version: str
    description: str
    tags: list[str]

class SkillSummaryResponse(BaseModel):
    """Outbound representation of a skill in list endpoints."""
    skill_id: str
    name: str
    version: str
    description: str

class ToolCatalogEntryResponse(BaseModel):
    """Outbound representation of a tool in the catalog."""
    tool_id: str
    server_id: str
    name: str
    description: str
    available: bool

class HealthResponse(BaseModel):
    """Health check response."""
    status: str                           # "healthy" | "degraded" | "unhealthy"
    mcp_servers: dict[str, str]           # server_id → "connected" | "unavailable"
    storage: str                          # "connected" | "unavailable"
    cache: str                            # "connected" | "unavailable"
```

---

## 5. Endpoint Specification

### 5.1 Runs

**Create and execute a run**

```
POST /runs
```

Request body: `CreateRunRequest`

Behavior:
1. Validate request
2. Generate `correlation_id` if not provided
3. Delegate to agent.core `AgentRuntime.execute()` via the dispatch engine
4. Return `RunResponse` with status CREATED (execution proceeds asynchronously)

Response: `201 Created` → `RunResponse`

The run begins execution immediately after creation. The caller monitors progress via the SSE stream.

**Get a run**

```
GET /runs/{run_id}
```

Response: `200 OK` → `RunResponse`

**List runs**

```
GET /runs?status={status}&limit={limit}&offset={offset}
```

Query parameters:
- `status` (optional): filter by RunStatus
- `limit` (default: 50, max: 200)
- `offset` (default: 0)

Response: `200 OK` → `list[RunResponse]`

**Cancel a run**

```
POST /runs/{run_id}/cancel
```

Behavior:
1. Validate run exists and is in a cancellable state (RUNNING or AWAITING_APPROVAL)
2. Delegate to agent.core to transition run to CANCELLED
3. Return updated `RunResponse`

Response: `200 OK` → `RunResponse`

**Get steps for a run**

```
GET /runs/{run_id}/steps
```

Response: `200 OK` → `list[StepResponse]`

**Get artifacts for a run**

```
GET /runs/{run_id}/artifacts
```

Response: `200 OK` → `list[ArtifactResponse]`

### 5.2 Approvals

**Resolve an approval**

```
POST /runs/{run_id}/approvals/{approval_id}/resolve
```

Request body: `ResolveApprovalRequest`

Behavior:
1. Validate approval exists, belongs to the run, and is in PENDING state
2. Delegate to agent.core approval system
3. Return updated `ApprovalResponse`

Response: `200 OK` → `ApprovalResponse`

**List approvals for a run**

```
GET /runs/{run_id}/approvals
```

Response: `200 OK` → `list[ApprovalResponse]`

### 5.3 Streaming

**Stream run events**

```
GET /runs/{run_id}/stream?since_sequence={N}
Content-Type: text/event-stream
```

Query parameters:
- `since_sequence` (optional, default: 0): resume from a specific sequence number

Behavior:
1. Validate run exists
2. If `since_sequence` is provided, load buffered events from Valkey (or MongoDB fallback)
3. Deliver all buffered events with sequence > `since_sequence`
4. Stream new events in real time as they are emitted
5. Send keepalive comments every 15 seconds to prevent connection timeout
6. Close the stream when the run reaches a terminal state (COMPLETED, FAILED, CANCELLED)

Event format:
```
event: {event_type}
data: {"event_type":"step.started","run_id":"...","step_id":"...","timestamp":"...","sequence":5,"payload":{...}}

```

Each SSE event uses the `event_type` as the SSE event name and the full event envelope as the data field.

### 5.4 Workflows

**List workflows**

```
GET /workflows?tags={tag1,tag2}
```

Query parameters:
- `tags` (optional): comma-separated filter

Response: `200 OK` → `list[WorkflowSummaryResponse]`

**Get workflow detail**

```
GET /workflows/{workflow_id}?version={version}
```

Query parameters:
- `version` (optional): specific version; defaults to highest

Response: `200 OK` → full workflow definition (steps included)

### 5.5 Skills

**List skills**

```
GET /skills
```

Response: `200 OK` → `list[SkillSummaryResponse]`

**Get skill detail**

```
GET /skills/{skill_id}?version={version}
```

Query parameters:
- `version` (optional): specific version; defaults to highest

Response: `200 OK` → full skill definition (system_prompt included)

### 5.6 Tools

**List tool catalog**

```
GET /tools
```

Response: `200 OK` → `list[ToolCatalogEntryResponse]`

### 5.7 Health

**Health check**

```
GET /health
```

Response: `200 OK` → `HealthResponse`

The health endpoint reports the status of each infrastructure dependency:

- **healthy**: all required MCP servers connected, storage connected, cache connected
- **degraded**: all required MCP servers connected but one or more optional servers or cache unavailable
- **unhealthy**: any required MCP server disconnected or storage unavailable

---

## 6. Correlation ID

Every request to the platform carries a correlation ID that propagates through all internal operations, events, log entries, and Logfire spans.

### 6.1 Generation

- If the caller provides `correlation_id` in the request body, it is used as-is
- If the caller provides an `X-Correlation-ID` header, it is used as-is
- If neither is present, the correlation middleware generates a UUID v4
- Request body `correlation_id` takes precedence over the header

### 6.2 Propagation

- The correlation ID is set on the Run entity at creation
- It is bound to the structlog logger for all operations within the run
- It is included as an attribute on all Logfire spans
- It is present on every Event entity emitted during the run
- It is returned in the `X-Correlation-ID` response header

### 6.3 Middleware

```python
@app.middleware("http")
async def correlation_middleware(request: Request, call_next):
    correlation_id = (
        request.headers.get("X-Correlation-ID")
        or str(uuid.uuid4())
    )
    request.state.correlation_id = correlation_id
    response = await call_next(request)
    response.headers["X-Correlation-ID"] = correlation_id
    return response
```

Note: for `POST /runs`, the `correlation_id` in the request body (if present) overrides the header value. This is handled in the endpoint, not the middleware.

---

## 7. Error Handling

### 7.1 HTTP Error Mapping

All errors returned by agent.core are mapped to appropriate HTTP status codes. The global error handler catches exceptions and produces consistent error responses.

| Error Condition | HTTP Status | Body |
|----------------|-------------|------|
| Request validation failure | 400 Bad Request | Pydantic validation errors |
| Run/approval/workflow not found | 404 Not Found | `{"detail": "..."}` |
| Invalid state transition (e.g., resolve non-PENDING approval) | 409 Conflict | `{"detail": "..."}` |
| Run not cancellable (terminal state) | 409 Conflict | `{"detail": "..."}` |
| Authentication failure | 401 Unauthorized | `{"detail": "..."}` |
| Internal runtime error | 500 Internal Server Error | `ErrorResponse` |
| Agent.core execution failure | Returned in run status, not as HTTP error | Via SSE stream |

### 7.2 Execution Errors vs HTTP Errors

Execution failures (step failures, tool failures, LLM errors) are not HTTP errors. They are part of the run lifecycle and are communicated through:

- The run's `status` field (FAILED) and `error` field
- The SSE event stream (`step.failed`, `run.failed` events)

The `POST /runs` endpoint returns `201 Created` even if the run subsequently fails during execution. The caller monitors the SSE stream for outcome.

---

## 8. Authentication

### 8.1 Strategy

Authentication is implemented as middleware. The specific mechanism is deployment-dependent:

- **Development**: no authentication (configurable bypass)
- **Production**: API key validation via `Authorization: Bearer {key}` header

The auth middleware extracts the caller identity and attaches it to the request state. This identity is used for:

- `resolved_by` on approval resolutions
- Audit logging
- Future authorization decisions

### 8.2 Middleware

```python
@app.middleware("http")
async def auth_middleware(request: Request, call_next):
    if settings.auth_enabled:
        token = request.headers.get("Authorization", "").removeprefix("Bearer ").strip()
        if not token:
            return JSONResponse(status_code=401, content={"detail": "Missing API key"})
        identity = await validate_token(token)
        if not identity:
            return JSONResponse(status_code=401, content={"detail": "Invalid API key"})
        request.state.identity = identity
    else:
        request.state.identity = "anonymous"
    return await call_next(request)
```

### 8.3 Configuration

Authentication settings are not in `agent.yaml` (which is platform configuration). They are provided via environment variables:

```
AUTH_ENABLED=true
AUTH_API_KEYS=key1,key2,key3
```

This keeps authentication deployment-specific and out of the platform configuration.

---

## 9. SSE Implementation Details

### 9.1 Connection Management

- One SSE connection per run per client
- Connections are tracked in memory (not persisted)
- If a client disconnects and reconnects with `?since_sequence=N`, buffered events are replayed
- The buffer source is Valkey first, MongoDB fallback (consistent with agent.core spec)

### 9.2 Event Delivery

```python
async def stream_run_events(run_id: str, since_sequence: int = 0):
    # 1. Replay buffered events
    buffered = await valkey.get_events_since(run_id, since_sequence)
    if not buffered:
        buffered = await mongo.get_events_for_run(run_id, since_sequence)
    for event in buffered:
        yield format_sse(event)

    # 2. Stream live events
    last_sequence = buffered[-1].sequence if buffered else since_sequence
    while True:
        new_events = await valkey.get_events_since(run_id, last_sequence)
        for event in new_events:
            yield format_sse(event)
            last_sequence = event.sequence

        # Check for terminal state
        status = await valkey.get_run_status(run_id)
        if status in (RunStatus.COMPLETED, RunStatus.FAILED, RunStatus.CANCELLED):
            # Drain any final events
            final = await valkey.get_events_since(run_id, last_sequence)
            for event in final:
                yield format_sse(event)
            return

        # Keepalive
        yield ": keepalive\n\n"
        await asyncio.sleep(1)
```

### 9.3 SSE Format

```python
def format_sse(event: Event) -> str:
    data = event.model_dump_json()
    return f"event: {event.event_type}\ndata: {data}\n\n"
```

### 9.4 Backpressure

If a client cannot consume events fast enough:

- Events continue to buffer in Valkey (up to TTL)
- The SSE connection is not throttled — events are sent as fast as the client reads
- If the buffer overflows (TTL expiry), the client must re-query via `GET /runs/{run_id}` and `GET /runs/{run_id}/steps` to reconstruct state

---

## 10. API Versioning

### 10.1 Strategy

The API uses URL path versioning:

```
/v1/runs
/v1/workflows
/v1/skills
...
```

All endpoints described in this document are under `/v1`. When breaking changes are introduced, a `/v2` prefix is added. Previous versions are maintained for a documented deprecation period.

### 10.2 Router Registration

```python
app = FastAPI(title="Agent Platform API", version="1.0.0")
app.include_router(runs_router, prefix="/v1")
app.include_router(approvals_router, prefix="/v1")
app.include_router(stream_router, prefix="/v1")
app.include_router(workflows_router, prefix="/v1")
app.include_router(skills_router, prefix="/v1")
app.include_router(tools_router, prefix="/v1")
app.include_router(health_router)          # Health is unversioned
```

---

## 11. Dependency Injection

agent.api uses FastAPI's dependency injection to access agent.core and agent.adapters.

### 11.1 Pattern

```python
from fastapi import Depends

async def get_runtime() -> AgentRuntime:
    """Provides the singleton AgentRuntime instance."""
    return app.state.runtime

async def get_registry() -> Registry:
    """Provides the singleton Registry instance."""
    return app.state.registry

@router.post("/runs", status_code=201)
async def create_run(
    request: CreateRunRequest,
    runtime: AgentRuntime = Depends(get_runtime),
    correlation_id: str = Depends(get_correlation_id),
) -> RunResponse:
    ...
```

### 11.2 Startup

The FastAPI application factory initializes all dependencies at startup and stores them on `app.state`:

```python
@app.on_event("startup")
async def startup():
    config = load_config()
    # agent.core initializes runtime, MCP client, tool catalog
    # agent.workflows loads registry
    # agent.adapters connects storage and cache
    app.state.runtime = runtime
    app.state.registry = registry
```

---

## 12. Observability

### 12.1 Request Tracing

Every HTTP request is traced via Logfire with:

- HTTP method and path
- Response status code
- Request latency
- Correlation ID as a span attribute

### 12.2 SSE Session Telemetry

Each SSE connection is tracked with:

- Connection duration
- Events delivered count
- Reconnection count (via `since_sequence` presence)
- Run ID as a span attribute

### 12.3 structlog Integration

All log statements bind `correlation_id` from the request state:

```python
log = structlog.get_logger().bind(correlation_id=request.state.correlation_id)
```

### 12.4 Span Naming

Span naming convention: `agent.api.{router}.{operation}`

Examples:
- `agent.api.runs.create`
- `agent.api.runs.get`
- `agent.api.stream.connect`
- `agent.api.approvals.resolve`
- `agent.api.health.check`

---

## 13. Boundary Rules

agent.api enforces these rules:

- **DTOs are the only shapes that cross the HTTP boundary** — canonical entities are never serialized directly to HTTP responses
- **No execution logic** — agent.api delegates all execution to agent.core
- **No direct storage or cache access** — all data access goes through agent.core or is injected via dependencies that wrap agent.adapters
- **No import of agent.web**
- **No import of agent.adapters directly** — access is via agent.core dependency injection
- **No canonical contract modification** — agent.api consumes canonical entities and projects them to DTOs
- **Correlation ID is generated here if not provided** — agent.core expects it to always be present
- **Authentication credentials and configuration are not in agent.yaml** — they use environment variables

---

## 14. Implementation Notes for Claude Code

- Use `fastapi.responses.StreamingResponse` with `media_type="text/event-stream"` for the SSE endpoint
- Use `asyncio.Queue` or polling for live event delivery in the SSE stream — choose based on complexity; polling against Valkey is simpler and sufficient for v1
- DTO → canonical and canonical → DTO mappings should be explicit factory methods, not automatic serialization
- The `POST /runs` endpoint should return immediately after run creation — execution is async. The caller monitors via SSE.
- All endpoints must return within a reasonable timeout — long-running operations are modeled as runs, not as HTTP requests
- OpenAPI schema is auto-generated by FastAPI from the DTO models — ensure descriptions and examples are present on DTO fields
- Health endpoint must not require authentication
- Rate limiting is out of scope for v1 but the middleware slot exists for future implementation
- CORS configuration should be included for agent.web (origins configurable via environment variable)

---

## 15. Example Request/Response Traces

### 15.1 Create and Monitor a Workflow Run

```
1. Client → POST /v1/runs
   Headers: Authorization: Bearer {key}
   Body: {
     "workflow_id": "classify-ttp",
     "input": {"description": "Attacker used PowerShell to download and execute a remote payload"},
     "skills": [],
     "correlation_id": "req-abc-123"
   }

   ← 201 Created
   Headers: X-Correlation-ID: req-abc-123
   Body: {
     "run_id": "run-001",
     "workflow_id": "classify-ttp",
     "workflow_version": "1.0.0",
     "status": "CREATED",
     "input": {"description": "..."},
     "created_at": "2026-03-24T12:00:00Z",
     "started_at": null,
     "completed_at": null,
     "step_count": 3,
     "current_step_index": 0,
     "error": null,
     "artifacts": []
   }

2. Client → GET /v1/runs/run-001/stream
   ← 200 OK (text/event-stream)

   event: run.started
   data: {"event_type":"run.started","run_id":"run-001","step_id":null,"timestamp":"...","sequence":1,"payload":{}}

   event: step.started
   data: {"event_type":"step.started","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":2,"payload":{"step_type":"AGENT","name":"Normalize Input"}}

   event: tool.invoked
   data: {"event_type":"tool.invoked","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":3,"payload":{"tool_id":"sentinel.run_query","input":{...}}}

   event: tool.result
   data: {"event_type":"tool.result","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":4,"payload":{"tool_id# agent.api — HTTP Interface Specification

**Version:** 1.0.0  
**Status:** Authoritative — Behavioral Specification  
**Scope:** HTTP Endpoints, DTOs, SSE Streaming, Authentication, Request Context  
**Parent:** technical-design-specification.md

---

## 1. Purpose

agent.api is the external interface of the platform. It owns the HTTP surface — nothing else is exposed to callers.

Every interaction with the platform from outside the process boundary arrives through agent.api. It translates between the external world (HTTP requests, SSE connections) and the internal world (canonical contracts, agent.core execution). It is a thin transport layer: it validates input, delegates to agent.core, and formats output. It does not make execution decisions.

agent.api is the single source of truth for:

- what HTTP endpoints exist and their contracts
- how requests are validated and translated to internal calls
- how canonical entities are projected to external DTOs
- how SSE streaming is managed
- how authentication and authorization are enforced
- how correlation IDs are generated and propagated
- what error responses look like

---

## 2. What agent.api Owns vs Delegates

**Owns**

- FastAPI application and router configuration
- HTTP endpoint definitions and OpenAPI documentation
- Request/response DTOs (Data Transfer Objects)
- Request validation (beyond what Pydantic handles automatically)
- Correlation ID generation (when not provided by the caller)
- SSE connection lifecycle and event delivery
- Authentication middleware
- HTTP error response formatting
- Rate limiting (if applicable)

**Does Not Own**

- Execution logic → agent.core
- Run/step lifecycle management → agent.core
- Canonical contract definitions → agent.core
- Workflow/skill registry operations → agent.workflows (accessed via agent.core)
- Storage/cache operations → agent.adapters (accessed via agent.core)
- Normalization of external payloads → agent.adapters
- UI → agent.web

---

## 3. Internal Component Map

```
agent.api
├── app.py                         # FastAPI application factory
├── routers/
│   ├── runs.py                    # Run execution and management endpoints
│   ├── approvals.py               # Approval resolution endpoints
│   ├── stream.py                  # SSE streaming endpoint
│   ├── workflows.py               # Workflow discovery endpoints
│   ├── skills.py                  # Skill discovery endpoints
│   ├── tools.py                   # Tool catalog endpoint
│   └── health.py                  # Health check endpoint
├── dtos/
│   ├── requests.py                # Inbound request DTOs
│   ├── responses.py               # Outbound response DTOs
│   └── events.py                  # SSE event envelope DTO
├── middleware/
│   ├── correlation.py             # Correlation ID generation/propagation
│   ├── auth.py                    # Authentication middleware
│   └── error_handler.py          # Global exception → HTTP error mapping
└── dependencies.py                # FastAPI dependency injection (runtime, adapters)
```

---

## 4. DTO Strategy

DTOs are the transport shapes used at the API boundary. They exist to decouple the external HTTP contract from internal canonical entities. DTOs are never used inside agent.core or agent.adapters.

### 4.1 Principle

```
Inbound:  Request DTO → validated → mapped to internal call (ExecutionRequest, etc.)
Outbound: Canonical entity → projected to Response DTO → serialized to JSON
```

DTOs may omit internal fields (e.g., `correlation_id` is not in the response unless requested), rename fields for API clarity, or flatten nested structures. The mapping between DTOs and canonical entities is explicit — no automatic passthrough.

### 4.2 Request DTOs

```python
class CreateRunRequest(BaseModel):
    """Inbound request to create and execute a run."""
    workflow_id: str | None = None
    input: dict
    skills: list[str] = []
    correlation_id: str | None = None    # Optional; generated if absent

class ResolveApprovalRequest(BaseModel):
    """Inbound request to resolve a pending approval."""
    resolution: str                       # "APPROVED" | "REJECTED" | "MODIFIED"
    payload: dict = {}                    # Decision data (required for MODIFIED)
    resolved_by: str
```

### 4.3 Response DTOs

```python
class RunResponse(BaseModel):
    """Outbound representation of a run."""
    run_id: str
    workflow_id: str | None
    workflow_version: str | None
    status: str
    input: dict
    created_at: datetime
    started_at: datetime | None
    completed_at: datetime | None
    step_count: int
    current_step_index: int | None
    error: ErrorResponse | None
    artifacts: list[ArtifactResponse]

class StepResponse(BaseModel):
    """Outbound representation of a step."""
    step_id: str
    run_id: str
    step_type: str
    name: str
    status: str
    started_at: datetime | None
    completed_at: datetime | None
    error: ErrorResponse | None
    tool_invocations: list[ToolInvocationResponse]

class ErrorResponse(BaseModel):
    """Outbound representation of an error."""
    error_id: str
    error_class: str
    message: str
    detail: dict
    retryable: bool

class ApprovalResponse(BaseModel):
    """Outbound representation of an approval."""
    approval_id: str
    run_id: str
    step_id: str
    status: str
    context: dict
    resolution: dict | None
    resolved_by: str | None
    requested_at: datetime
    resolved_at: datetime | None

class ArtifactResponse(BaseModel):
    """Outbound representation of an artifact."""
    artifact_id: str
    run_id: str
    name: str
    content_type: str
    created_at: datetime

class ToolInvocationResponse(BaseModel):
    """Outbound representation of a tool invocation."""
    invocation_id: str
    tool_id: str
    server_id: str
    started_at: datetime
    completed_at: datetime | None
    latency_ms: int | None
    error: ErrorResponse | None

class WorkflowSummaryResponse(BaseModel):
    """Outbound representation of a workflow in list endpoints."""
    workflow_id: str
    name: str
    version: str
    description: str
    tags: list[str]

class SkillSummaryResponse(BaseModel):
    """Outbound representation of a skill in list endpoints."""
    skill_id: str
    name: str
    version: str
    description: str

class ToolCatalogEntryResponse(BaseModel):
    """Outbound representation of a tool in the catalog."""
    tool_id: str
    server_id: str
    name: str
    description: str
    available: bool

class HealthResponse(BaseModel):
    """Health check response."""
    status: str                           # "healthy" | "degraded" | "unhealthy"
    mcp_servers: dict[str, str]           # server_id → "connected" | "unavailable"
    storage: str                          # "connected" | "unavailable"
    cache: str                            # "connected" | "unavailable"
```

---

## 5. Endpoint Specification

### 5.1 Runs

**Create and execute a run**

```
POST /runs
```

Request body: `CreateRunRequest`

Behavior:
1. Validate request
2. Generate `correlation_id` if not provided
3. Delegate to agent.core `AgentRuntime.execute()` via the dispatch engine
4. Return `RunResponse` with status CREATED (execution proceeds asynchronously)

Response: `201 Created` → `RunResponse`

The run begins execution immediately after creation. The caller monitors progress via the SSE stream.

**Get a run**

```
GET /runs/{run_id}
```

Response: `200 OK` → `RunResponse`

**List runs**

```
GET /runs?status={status}&limit={limit}&offset={offset}
```

Query parameters:
- `status` (optional): filter by RunStatus
- `limit` (default: 50, max: 200)
- `offset` (default: 0)

Response: `200 OK` → `list[RunResponse]`

**Cancel a run**

```
POST /runs/{run_id}/cancel
```

Behavior:
1. Validate run exists and is in a cancellable state (RUNNING or AWAITING_APPROVAL)
2. Delegate to agent.core to transition run to CANCELLED
3. Return updated `RunResponse`

Response: `200 OK` → `RunResponse`

**Get steps for a run**

```
GET /runs/{run_id}/steps
```

Response: `200 OK` → `list[StepResponse]`

**Get artifacts for a run**

```
GET /runs/{run_id}/artifacts
```

Response: `200 OK` → `list[ArtifactResponse]`

### 5.2 Approvals

**Resolve an approval**

```
POST /runs/{run_id}/approvals/{approval_id}/resolve
```

Request body: `ResolveApprovalRequest`

Behavior:
1. Validate approval exists, belongs to the run, and is in PENDING state
2. Delegate to agent.core approval system
3. Return updated `ApprovalResponse`

Response: `200 OK` → `ApprovalResponse`

**List approvals for a run**

```
GET /runs/{run_id}/approvals
```

Response: `200 OK` → `list[ApprovalResponse]`

### 5.3 Streaming

**Stream run events**

```
GET /runs/{run_id}/stream?since_sequence={N}
Content-Type: text/event-stream
```

Query parameters:
- `since_sequence` (optional, default: 0): resume from a specific sequence number

Behavior:
1. Validate run exists
2. If `since_sequence` is provided, load buffered events from Valkey (or MongoDB fallback)
3. Deliver all buffered events with sequence > `since_sequence`
4. Stream new events in real time as they are emitted
5. Send keepalive comments every 15 seconds to prevent connection timeout
6. Close the stream when the run reaches a terminal state (COMPLETED, FAILED, CANCELLED)

Event format:
```
event: {event_type}
data: {"event_type":"step.started","run_id":"...","step_id":"...","timestamp":"...","sequence":5,"payload":{...}}

```

Each SSE event uses the `event_type` as the SSE event name and the full event envelope as the data field.

### 5.4 Workflows

**List workflows**

```
GET /workflows?tags={tag1,tag2}
```

Query parameters:
- `tags` (optional): comma-separated filter

Response: `200 OK` → `list[WorkflowSummaryResponse]`

**Get workflow detail**

```
GET /workflows/{workflow_id}?version={version}
```

Query parameters:
- `version` (optional): specific version; defaults to highest

Response: `200 OK` → full workflow definition (steps included)

### 5.5 Skills

**List skills**

```
GET /skills
```

Response: `200 OK` → `list[SkillSummaryResponse]`

**Get skill detail**

```
GET /skills/{skill_id}?version={version}
```

Query parameters:
- `version` (optional): specific version; defaults to highest

Response: `200 OK` → full skill definition (system_prompt included)

### 5.6 Tools

**List tool catalog**

```
GET /tools
```

Response: `200 OK` → `list[ToolCatalogEntryResponse]`

### 5.7 Health

**Health check**

```
GET /health
```

Response: `200 OK` → `HealthResponse`

The health endpoint reports the status of each infrastructure dependency:

- **healthy**: all required MCP servers connected, storage connected, cache connected
- **degraded**: all required MCP servers connected but one or more optional servers or cache unavailable
- **unhealthy**: any required MCP server disconnected or storage unavailable

---

## 6. Correlation ID

Every request to the platform carries a correlation ID that propagates through all internal operations, events, log entries, and Logfire spans.

### 6.1 Generation

- If the caller provides `correlation_id` in the request body, it is used as-is
- If the caller provides an `X-Correlation-ID` header, it is used as-is
- If neither is present, the correlation middleware generates a UUID v4
- Request body `correlation_id` takes precedence over the header

### 6.2 Propagation

- The correlation ID is set on the Run entity at creation
- It is bound to the structlog logger for all operations within the run
- It is included as an attribute on all Logfire spans
- It is present on every Event entity emitted during the run
- It is returned in the `X-Correlation-ID` response header

### 6.3 Middleware

```python
@app.middleware("http")
async def correlation_middleware(request: Request, call_next):
    correlation_id = (
        request.headers.get("X-Correlation-ID")
        or str(uuid.uuid4())
    )
    request.state.correlation_id = correlation_id
    response = await call_next(request)
    response.headers["X-Correlation-ID"] = correlation_id
    return response
```

Note: for `POST /runs`, the `correlation_id` in the request body (if present) overrides the header value. This is handled in the endpoint, not the middleware.

---

## 7. Error Handling

### 7.1 HTTP Error Mapping

All errors returned by agent.core are mapped to appropriate HTTP status codes. The global error handler catches exceptions and produces consistent error responses.

| Error Condition | HTTP Status | Body |
|----------------|-------------|------|
| Request validation failure | 400 Bad Request | Pydantic validation errors |
| Run/approval/workflow not found | 404 Not Found | `{"detail": "..."}` |
| Invalid state transition (e.g., resolve non-PENDING approval) | 409 Conflict | `{"detail": "..."}` |
| Run not cancellable (terminal state) | 409 Conflict | `{"detail": "..."}` |
| Authentication failure | 401 Unauthorized | `{"detail": "..."}` |
| Internal runtime error | 500 Internal Server Error | `ErrorResponse` |
| Agent.core execution failure | Returned in run status, not as HTTP error | Via SSE stream |

### 7.2 Execution Errors vs HTTP Errors

Execution failures (step failures, tool failures, LLM errors) are not HTTP errors. They are part of the run lifecycle and are communicated through:

- The run's `status` field (FAILED) and `error` field
- The SSE event stream (`step.failed`, `run.failed` events)

The `POST /runs` endpoint returns `201 Created` even if the run subsequently fails during execution. The caller monitors the SSE stream for outcome.

---

## 8. Authentication

### 8.1 Strategy

Authentication is implemented as middleware. The specific mechanism is deployment-dependent:

- **Development**: no authentication (configurable bypass)
- **Production**: API key validation via `Authorization: Bearer {key}` header

The auth middleware extracts the caller identity and attaches it to the request state. This identity is used for:

- `resolved_by` on approval resolutions
- Audit logging
- Future authorization decisions

### 8.2 Middleware

```python
@app.middleware("http")
async def auth_middleware(request: Request, call_next):
    if settings.auth_enabled:
        token = request.headers.get("Authorization", "").removeprefix("Bearer ").strip()
        if not token:
            return JSONResponse(status_code=401, content={"detail": "Missing API key"})
        identity = await validate_token(token)
        if not identity:
            return JSONResponse(status_code=401, content={"detail": "Invalid API key"})
        request.state.identity = identity
    else:
        request.state.identity = "anonymous"
    return await call_next(request)
```

### 8.3 Configuration

Authentication settings are not in `agent.yaml` (which is platform configuration). They are provided via environment variables:

```
AUTH_ENABLED=true
AUTH_API_KEYS=key1,key2,key3
```

This keeps authentication deployment-specific and out of the platform configuration.

---

## 9. SSE Implementation Details

### 9.1 Connection Management

- One SSE connection per run per client
- Connections are tracked in memory (not persisted)
- If a client disconnects and reconnects with `?since_sequence=N`, buffered events are replayed
- The buffer source is Valkey first, MongoDB fallback (consistent with agent.core spec)

### 9.2 Event Delivery

```python
async def stream_run_events(run_id: str, since_sequence: int = 0):
    # 1. Replay buffered events
    buffered = await valkey.get_events_since(run_id, since_sequence)
    if not buffered:
        buffered = await mongo.get_events_for_run(run_id, since_sequence)
    for event in buffered:
        yield format_sse(event)

    # 2. Stream live events
    last_sequence = buffered[-1].sequence if buffered else since_sequence
    while True:
        new_events = await valkey.get_events_since(run_id, last_sequence)
        for event in new_events:
            yield format_sse(event)
            last_sequence = event.sequence

        # Check for terminal state
        status = await valkey.get_run_status(run_id)
        if status in (RunStatus.COMPLETED, RunStatus.FAILED, RunStatus.CANCELLED):
            # Drain any final events
            final = await valkey.get_events_since(run_id, last_sequence)
            for event in final:
                yield format_sse(event)
            return

        # Keepalive
        yield ": keepalive\n\n"
        await asyncio.sleep(1)
```

### 9.3 SSE Format

```python
def format_sse(event: Event) -> str:
    data = event.model_dump_json()
    return f"event: {event.event_type}\ndata: {data}\n\n"
```

### 9.4 Backpressure

If a client cannot consume events fast enough:

- Events continue to buffer in Valkey (up to TTL)
- The SSE connection is not throttled — events are sent as fast as the client reads
- If the buffer overflows (TTL expiry), the client must re-query via `GET /runs/{run_id}` and `GET /runs/{run_id}/steps` to reconstruct state

---

## 10. API Versioning

### 10.1 Strategy

The API uses URL path versioning:

```
/v1/runs
/v1/workflows
/v1/skills
...
```

All endpoints described in this document are under `/v1`. When breaking changes are introduced, a `/v2` prefix is added. Previous versions are maintained for a documented deprecation period.

### 10.2 Router Registration

```python
app = FastAPI(title="Agent Platform API", version="1.0.0")
app.include_router(runs_router, prefix="/v1")
app.include_router(approvals_router, prefix="/v1")
app.include_router(stream_router, prefix="/v1")
app.include_router(workflows_router, prefix="/v1")
app.include_router(skills_router, prefix="/v1")
app.include_router(tools_router, prefix="/v1")
app.include_router(health_router)          # Health is unversioned
```

---

## 11. Dependency Injection

agent.api uses FastAPI's dependency injection to access agent.core and agent.adapters.

### 11.1 Pattern

```python
from fastapi import Depends

async def get_runtime() -> AgentRuntime:
    """Provides the singleton AgentRuntime instance."""
    return app.state.runtime

async def get_registry() -> Registry:
    """Provides the singleton Registry instance."""
    return app.state.registry

@router.post("/runs", status_code=201)
async def create_run(
    request: CreateRunRequest,
    runtime: AgentRuntime = Depends(get_runtime),
    correlation_id: str = Depends(get_correlation_id),
) -> RunResponse:
    ...
```

### 11.2 Startup

The FastAPI application factory initializes all dependencies at startup and stores them on `app.state`:

```python
@app.on_event("startup")
async def startup():
    config = load_config()
    # agent.core initializes runtime, MCP client, tool catalog
    # agent.workflows loads registry
    # agent.adapters connects storage and cache
    app.state.runtime = runtime
    app.state.registry = registry
```

---

## 12. Observability

### 12.1 Request Tracing

Every HTTP request is traced via Logfire with:

- HTTP method and path
- Response status code
- Request latency
- Correlation ID as a span attribute

### 12.2 SSE Session Telemetry

Each SSE connection is tracked with:

- Connection duration
- Events delivered count
- Reconnection count (via `since_sequence` presence)
- Run ID as a span attribute

### 12.3 structlog Integration

All log statements bind `correlation_id` from the request state:

```python
log = structlog.get_logger().bind(correlation_id=request.state.correlation_id)
```

### 12.4 Span Naming

Span naming convention: `agent.api.{router}.{operation}`

Examples:
- `agent.api.runs.create`
- `agent.api.runs.get`
- `agent.api.stream.connect`
- `agent.api.approvals.resolve`
- `agent.api.health.check`

---

## 13. Boundary Rules

agent.api enforces these rules:

- **DTOs are the only shapes that cross the HTTP boundary** — canonical entities are never serialized directly to HTTP responses
- **No execution logic** — agent.api delegates all execution to agent.core
- **No direct storage or cache access** — all data access goes through agent.core or is injected via dependencies that wrap agent.adapters
- **No import of agent.web**
- **No import of agent.adapters directly** — access is via agent.core dependency injection
- **No canonical contract modification** — agent.api consumes canonical entities and projects them to DTOs
- **Correlation ID is generated here if not provided** — agent.core expects it to always be present
- **Authentication credentials and configuration are not in agent.yaml** — they use environment variables

---

## 14. Implementation Notes for Claude Code

- Use `fastapi.responses.StreamingResponse` with `media_type="text/event-stream"` for the SSE endpoint
- Use `asyncio.Queue` or polling for live event delivery in the SSE stream — choose based on complexity; polling against Valkey is simpler and sufficient for v1
- DTO → canonical and canonical → DTO mappings should be explicit factory methods, not automatic serialization
- The `POST /runs` endpoint should return immediately after run creation — execution is async. The caller monitors via SSE.
- All endpoints must return within a reasonable timeout — long-running operations are modeled as runs, not as HTTP requests
- OpenAPI schema is auto-generated by FastAPI from the DTO models — ensure descriptions and examples are present on DTO fields
- Health endpoint must not require authentication
- Rate limiting is out of scope for v1 but the middleware slot exists for future implementation
- CORS configuration should be included for agent.web (origins configurable via environment variable)

---

## 15. Example Request/Response Traces

### 15.1 Create and Monitor a Workflow Run

```
1. Client → POST /v1/runs
   Headers: Authorization: Bearer {key}
   Body: {
     "workflow_id": "classify-ttp",
     "input": {"description": "Attacker used PowerShell to download and execute a remote payload"},
     "skills": [],
     "correlation_id": "req-abc-123"
   }

   ← 201 Created
   Headers: X-Correlation-ID: req-abc-123
   Body: {
     "run_id": "run-001",
     "workflow_id": "classify-ttp",
     "workflow_version": "1.0.0",
     "status": "CREATED",
     "input": {"description": "..."},
     "created_at": "2026-03-24T12:00:00Z",
     "started_at": null,
     "completed_at": null,
     "step_count": 3,
     "current_step_index": 0,
     "error": null,
     "artifacts": []
   }

2. Client → GET /v1/runs/run-001/stream
   ← 200 OK (text/event-stream)

   event: run.started
   data: {"event_type":"run.started","run_id":"run-001","step_id":null,"timestamp":"...","sequence":1,"payload":{}}

   event: step.started
   data: {"event_type":"step.started","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":2,"payload":{"step_type":"AGENT","name":"Normalize Input"}}

   event: tool.invoked
   data: {"event_type":"tool.invoked","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":3,"payload":{"tool_id":"sentinel.run_query","input":{...}}}

   event: tool.result
   data: {"event_type":"tool.result","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":4,"payload":{"tool_id":"sentinel.run_query","latency_ms":230}}

   event: step.completed
   data: {"event_type":"step.completed","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":5,"payload":{"output":{...}}}

   ... (more steps) ...

   event: run.awaiting_approval
   data: {"event_type":"run.awaiting_approval","run_id":"run-001","step_id":"review","timestamp":"...","sequence":12,"payload":{"approval_id":"apr-001"}}

3. Client → POST /v1/runs/run-001/approvals/apr-001/resolve
   Body: {
     "resolution": "APPROVED",
     "payload": {},
     "resolved_by": "analyst@example.com"
   }

   ← 200 OK
   Body: {
     "approval_id": "apr-001",
     "run_id": "run-001",
     "step_id": "review",
     "status": "APPROVED",
     "resolution": {},
     "resolved_by": "analyst@example.com",
     "requested_at": "...",
     "resolved_at": "..."
   }

4. SSE stream continues:

   event: approval.resolved
   data: {"event_type":"approval.resolved","run_id":"run-001","step_id":"review","timestamp":"...","sequence":13,"payload":{"status":"APPROVED"}}

   event: run.completed
   data: {"event_type":"run.completed","run_id":"run-001","step_id":null,"timestamp":"...","sequence":14,"payload":{}}

   (stream closes)
```

### 15.2 SSE Reconnection

```
1. Client connects: GET /v1/runs/run-001/stream
   Receives events with sequences 1-8
   Connection drops

2. Client reconnects: GET /v1/runs/run-001/stream?since_sequence=8
   Receives buffered events 9-12 immediately
   Continues streaming live events from 13 onward
```:"step.completed","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":5,"payload":{"output":{...}}}

   ... (more steps) ...

   event: run.awaiting_approval
   data: {"event_type":"run.awaiting_approval","run_id":"run-001","step_id":"review","timestamp":"...","sequence":12,"payload":{"approval_id":"apr-001"}}

3. Client → POST /v1/runs/run-001/approvals/apr-001/resolve
   Body: {
     "resolution": "APPROVED",
     "payload": {},
     "resolved_by": "analyst@example.com"
   }

   ← 200 OK
   Body: {
     "approval_id": "apr-001",
     "run_id": "run-001",
     "step_id": "review",
     "status": "APPROVED",
     "resolution": {},
     "resolved_by": "analyst@example.com",
     "requested_at": "...",
     "resolved_at": "..."
   }

4. SSE stream continues:

   event: approval.resolved
   data: {"event_type":"approval.resolved","run_id":"run-001","step_id":"review","timestamp":"...","sequence":13,"payload":{"status":"APPROVED"}}

   event: run.completed
   data: {"event_type":"run.completed","run_id":"run-001","step_id":null,"timestamp":"...","sequence":14,"payload":{}}

   (stream closes)
```

### 15.2 SSE Reconnection

```
1. Client connects: GET /v1/runs/run-001/stream
   Receives events with sequences 1-8
   Connection drops

2. Client reconnects: GET /v1/runs/run-001/stream?since_sequence=8
   Receives buffered events 9-12 immediately
   Continues streaming live events from 13 onward
```
     "approval_id": "apr-001",
     "run_id": "run-001",
     "step_id": "review",
     "status": "APPROVED",
     "resolution": {},
     "resolved_by": "analyst@example.com",
     "requested_at": "...",
     "resolved_at": "..."
   }

4. SSE stream continues:

   event: approval.resolved
   data: {"event_type":"approval.resolved","run_id":"run-001","step_id":"review","timestamp":"...","sequence":13,"payload":{"status":"APPROVED"}}

   event: run.completed
   data: {"event_type":"run.completed","run_id":"run-001","step_id":null,"timestamp":"...","sequence":14,"payload":{}}

   (stream closes)
```

### 15.2 SSE Reconnection

```
1. Client connects: GET /v1/runs/run-001/stream
   Receives events with sequences 1-8
   Connection drops

2. Client reconnects: GET /v1/runs/run-001/stream?since_sequence=8
   Receives buffered events 9-12 immediately
   Continues streaming live events from 13 onward
```
---

## 10. API Versioning

### 10.1 Strategy

The API uses URL path versioning:

```
/v1/runs
/v1/workflows
/v1/skills
...
```

All endpoints described in this document are under `/v1`. When breaking changes are introduced, a `/v2` prefix is added. Previous versions are maintained for a documented deprecation period.

### 10.2 Router Registration

```python
app = FastAPI(title="Agent Platform API", version="1.0.0")
app.include_router(runs_router, prefix="/v1")
app.include_router(approvals_router, prefix="/v1")
app.include_router(stream_router, prefix="/v1")
app.include_router(workflows_router, prefix="/v1")
app.include_router(skills_router, prefix="/v1")
app.include_router(tools_router, prefix="/v1")
app.include_router(health_router)          # Health is unversioned
```

---

## 11. Dependency Injection

agent.api uses FastAPI's dependency injection to access agent.core and agent.adapters.

### 11.1 Pattern

```python
from fastapi import Depends

async def get_runtime() -> AgentRuntime:
    """Provides the singleton AgentRuntime instance."""
    return app.state.runtime

async def get_registry() -> Registry:
    """Provides the singleton Registry instance."""
    return app.state.registry

@router.post("/runs", status_code=201)
async def create_run(
    request: CreateRunRequest,
    runtime: AgentRuntime = Depends(get_runtime),
    correlation_id: str = Depends(get_correlation_id),
) -> RunResponse:
    ...
```

### 11.2 Startup

The FastAPI application factory initializes all dependencies at startup and stores them on `app.state`:

```python
@app.on_event("startup")
async def startup():
    config = load_config()
    # agent.core initializes runtime, MCP client, tool catalog
    # agent.workflows loads registry
    # agent.adapters connects storage and cache
    app.state.runtime = runtime
    app.state.registry = registry
```

---

## 12. Observability

### 12.1 Request Tracing

Every HTTP request is traced via Logfire with:

- HTTP method and path
- Response status code
- Request latency
- Correlation ID as a span attribute

### 12.2 SSE Session Telemetry

Each SSE connection is tracked with:

- Connection duration
- Events delivered count
- Reconnection count (via `since_sequence` presence)
- Run ID as a span attribute

### 12.3 structlog Integration

All log statements bind `correlation_id` from the request state:

```python
log = structlog.get_logger().bind(correlation_id=request.state.correlation_id)
```

### 12.4 Span Naming

Span naming convention: `agent.api.{router}.{operation}`

Examples:
- `agent.api.runs.create`
- `agent.api.runs.get`
- `agent.api.stream.connect`
- `agent.api.approvals.resolve`
- `agent.api.health.check`

---

## 13. Boundary Rules

agent.api enforces these rules:

- **DTOs are the only shapes that cross the HTTP boundary** — canonical entities are never serialized directly to HTTP responses
- **No execution logic** — agent.api delegates all execution to agent.core
- **No direct storage or cache access** — all data access goes through agent.core or is injected via dependencies that wrap agent.adapters
- **No import of agent.web**
- **No import of agent.adapters directly** — access is via agent.core dependency injection
- **No canonical contract modification** — agent.api consumes canonical entities and projects them to DTOs
- **Correlation ID is generated here if not provided** — agent.core expects it to always be present
- **Authentication credentials and configuration are not in agent.yaml** — they use environment variables

---

## 14. Implementation Notes for Claude Code

- Use `fastapi.responses.StreamingResponse` with `media_type="text/event-stream"` for the SSE endpoint
- Use `asyncio.Queue` or polling for live event delivery in the SSE stream — choose based on complexity; polling against Valkey is simpler and sufficient for v1
- DTO → canonical and canonical → DTO mappings should be explicit factory methods, not automatic serialization
- The `POST /runs` endpoint should return immediately after run creation — execution is async. The caller monitors via SSE.
- All endpoints must return within a reasonable timeout — long-running operations are modeled as runs, not as HTTP requests
- OpenAPI schema is auto-generated by FastAPI from the DTO models — ensure descriptions and examples are present on DTO fields
- Health endpoint must not require authentication
- Rate limiting is out of scope for v1 but the middleware slot exists for future implementation
- CORS configuration should be included for agent.web (origins configurable via environment variable)

---

## 15. Example Request/Response Traces

### 15.1 Create and Monitor a Workflow Run

```
1. Client → POST /v1/runs
   Headers: Authorization: Bearer {key}
   Body: {
     "workflow_id": "classify-ttp",
     "input": {"description": "Attacker used PowerShell to download and execute a remote payload"},
     "skills": [],
     "correlation_id": "req-abc-123"
   }

   ← 201 Created
   Headers: X-Correlation-ID: req-abc-123
   Body: {
     "run_id": "run-001",
     "workflow_id": "classify-ttp",
     "workflow_version": "1.0.0",
     "status": "CREATED",
     "input": {"description": "..."},
     "created_at": "2026-03-24T12:00:00Z",
     "started_at": null,
     "completed_at": null,
     "step_count": 3,
     "current_step_index": 0,
     "error": null,
     "artifacts": []
   }

2. Client → GET /v1/runs/run-001/stream
   ← 200 OK (text/event-stream)

   event: run.started
   data: {"event_type":"run.started","run_id":"run-001","step_id":null,"timestamp":"...","sequence":1,"payload":{}}

   event: step.started
   data: {"event_type":"step.started","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":2,"payload":{"step_type":"AGENT","name":"Normalize Input"}}

   event: tool.invoked
   data: {"event_type":"tool.invoked","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":3,"payload":{"tool_id":"sentinel.run_query","input":{...}}}

   event: tool.result
   data: {"event_type":"tool.result","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":4,"payload":{"tool_id":"sentinel.run_query","latency_ms":230}}

   event: step.completed
   data: {"event_type":"step.completed","run_id":"run-001","step_id":"normalize-input","timestamp":"...","sequence":5,"payload":{"output":{...}}}

   ... (more steps) ...

   event: run.awaiting_approval
   data: {"event_type":"run.awaiting_approval","run_id":"run-001","step_id":"review","timestamp":"...","sequence":12,"payload":{"approval_id":"apr-001"}}

3. Client → POST /v1/runs/run-001/approvals/apr-001/resolve
   Body: {
     "resolution": "APPROVED",
     "payload": {},
     "resolved_by": "analyst@example.com"
   }

   ← 200 OK
   Body: {
     "approval_id": "apr-001",
     "run_id": "run-001",
     "step_id": "review",
     "status": "APPROVED",
     "resolution": {},
     "resolved_by": "analyst@example.com",
     "requested_at": "...",
     "resolved_at": "..."
   }

4. SSE stream continues:

   event: approval.resolved
   data: {"event_type":"approval.resolved","run_id":"run-001","step_id":"review","timestamp":"...","sequence":13,"payload":{"status":"APPROVED"}}

   event: run.completed
   data: {"event_type":"run.completed","run_id":"run-001","step_id":null,"timestamp":"...","sequence":14,"payload":{}}

   (stream closes)
```

### 15.2 SSE Reconnection

```
1. Client connects: GET /v1/runs/run-001/stream
   Receives events with sequences 1-8
   Connection drops

2. Client reconnects: GET /v1/runs/run-001/stream?since_sequence=8
   Receives buffered events 9-12 immediately
   Continues streaming live events from 13 onward
```