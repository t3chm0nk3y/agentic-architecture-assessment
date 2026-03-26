# agent.adapters — External Integration and Infrastructure Specification

**Version:** 1.0.0  
**Status:** Authoritative — Behavioral Specification  
**Scope:** MCP Servers, Storage Adapter, Cache Adapter, Normalization Boundary  
**Parent:** technical-design-specification.md

---

## 1. Purpose

agent.adapters is the boundary layer of the platform. It owns all contact with the outside world — external vendor systems, persistent storage, and cache infrastructure.

Every external system the platform interacts with is accessed through an adapter defined here. No other module may make direct calls to external APIs, databases, or cache backends. agent.adapters translates between the outside world and the platform's canonical contracts, ensuring that by the time data reaches agent.core, it is already in canonical form.

agent.adapters is the single source of truth for:

- how external vendor systems are exposed as MCP servers
- how vendor-specific payloads are normalized to canonical contracts
- how MongoDB is accessed for persistent storage
- how Valkey is accessed for caching and SSE event buffering
- what external schemas look like before normalization
- how infrastructure errors are classified and surfaced

---

## 2. What agent.adapters Owns vs Delegates

**Owns**

- MCP server implementations (one per external integration)
- Vendor-specific payload schemas (external schemas)
- Normalization logic — mapping vendor payloads to canonical contracts (exactly once, at this boundary)
- MongoDB storage adapter (connection management, CRUD operations, index management)
- Valkey cache adapter (connection management, key operations, TTL enforcement)
- Infrastructure error classification (connection failures, timeouts, serialization errors)

**Does Not Own**

- Canonical contract definitions → agent.core
- Canonical contract meaning or interpretation → agent.core
- MCP client (connection to servers, tool catalog, invocation) → agent.core
- Runtime execution logic → agent.core
- Workflow or skill definitions → agent.workflows
- HTTP interface or DTOs → agent.api
- UI → agent.web
- What data to store or cache → decided by agent.core; agent.adapters provides the mechanism

---

## 3. Internal Component Map

```
agent.adapters
├── mcp_servers/
│   ├── base.py                    # Shared MCP server utilities and patterns
│   ├── sentinel/
│   │   ├── server.py              # MCP server implementation
│   │   ├── client.py              # Vendor API client (HTTP calls to Sentinel)
│   │   ├── schemas.py             # Vendor-specific response schemas
│   │   └── normalizer.py          # Vendor → canonical mapping
│   ├── crowdstrike/
│   │   ├── server.py
│   │   ├── client.py
│   │   ├── schemas.py
│   │   └── normalizer.py
│   └── ...                        # One directory per integration
├── storage/
│   ├── mongo_adapter.py           # MongoDB adapter
│   └── collections.py            # Collection names, index definitions
└── cache/
    └── valkey_adapter.py          # Valkey adapter
```

---

## 4. Normalization Boundary

This is the most important responsibility of agent.adapters. The normalization boundary is where external vendor payloads become canonical entities. This happens exactly once, inside the adapter, before data is returned to the MCP client in agent.core.

### 4.1 Principle

agent.core must never see a vendor-specific payload. By the time a tool response reaches the MCP client, it must conform to canonical contract schemas. agent.core validates the response against the canonical schema — if validation fails, the error is classified as OUTPUT_VALIDATION_FAILURE and attributed to the adapter.

### 4.2 Normalization Flow

```
Vendor API (e.g., CrowdStrike Falcon API)
    → Vendor API client (adapters/mcp_servers/crowdstrike/client.py)
    → Raw vendor response (vendor-specific schema)
    → Normalizer (adapters/mcp_servers/crowdstrike/normalizer.py)
    → Canonical entity (e.g., Asset, or structured dict conforming to tool output_schema)
    → MCP server returns normalized response
    → MCP client (agent.core) validates against canonical schema
    → Runtime consumes canonical data
```

### 4.3 Normalizer Contract

Every MCP server directory must contain a `normalizer.py` that implements mapping functions from vendor schemas to canonical shapes.

```python
class CrowdStrikeNormalizer:
    """Maps CrowdStrike API responses to canonical contracts."""

    def normalize_device(self, raw: CrowdStrikeDevice) -> dict:
        """Map a CrowdStrike device to canonical Asset shape."""
        return {
            "asset_id": generate_stable_id("crowdstrike", raw.device_id),
            "external_id": raw.device_id,
            "source": "crowdstrike",
            "asset_type": "endpoint",
            "name": raw.hostname or raw.device_id,
            "attributes": {
                "platform": raw.platform_name,
                "os_version": raw.os_version,
                "agent_version": raw.agent_version,
                "last_seen": raw.last_seen,
                "status": raw.status,
            },
            "last_seen": parse_timestamp(raw.last_seen),
        }
```

### 4.4 Rules

- Normalization happens in the adapter, not in agent.core
- Each normalizer maps to canonical contract shapes defined in agent.core/contracts/
- Normalizers must not import agent.core runtime components — only canonical contract schemas for reference
- If a vendor field has no canonical equivalent, it goes into the `attributes` dict (for Asset) or a generic `metadata` field — never dropped silently
- Normalization must be deterministic — same input always produces same output
- Normalizers must handle missing or null vendor fields gracefully with sensible defaults

---

## 5. MCP Server Implementation

Each external integration is implemented as an MCP server. MCP servers run as independent processes and communicate with the MCP client in agent.core via the MCP protocol (SSE or stdio transport).

### 5.1 Server Structure

Every MCP server must:

- Expose one or more tools with well-defined input and output schemas
- Return responses that have been normalized to canonical shapes (via the normalizer)
- Handle vendor authentication (API keys, OAuth tokens) internally
- Handle vendor rate limiting and retry logic internally
- Classify vendor-specific errors into standard categories before returning
- Include structured logging with structlog
- Include Logfire instrumentation for vendor API call latency and outcomes

### 5.2 Tool Definition Pattern

```python
@server.tool(
    name="get_device",
    description="Retrieve endpoint device details by device ID",
    input_schema={
        "type": "object",
        "properties": {
            "device_id": {"type": "string", "description": "CrowdStrike device ID"}
        },
        "required": ["device_id"]
    },
    output_schema={
        "type": "object",
        "properties": {
            "asset_id": {"type": "string"},
            "external_id": {"type": "string"},
            "source": {"type": "string"},
            "asset_type": {"type": "string"},
            "name": {"type": "string"},
            "attributes": {"type": "object"},
            "last_seen": {"type": "string", "format": "date-time"}
        }
    }
)
async def get_device(device_id: str) -> dict:
    raw = await client.get_device(device_id)
    return normalizer.normalize_device(raw)
```

### 5.3 Error Handling in MCP Servers

MCP servers must catch vendor-specific exceptions and return structured error responses that the MCP client in agent.core can classify.

```python
try:
    raw = await client.get_device(device_id)
    return normalizer.normalize_device(raw)
except VendorAuthError as e:
    raise McpError(
        code=McpErrorCode.INTERNAL_ERROR,
        message=f"Authentication failed for CrowdStrike: {e}",
    )
except VendorRateLimitError as e:
    raise McpError(
        code=McpErrorCode.INTERNAL_ERROR,
        message=f"Rate limit exceeded for CrowdStrike: {e}",
    )
except VendorNotFoundError as e:
    raise McpError(
        code=McpErrorCode.INVALID_PARAMS,
        message=f"Device not found: {device_id}",
    )
```

### 5.4 Vendor API Client Pattern

Each MCP server directory contains a `client.py` that wraps the vendor's HTTP API. This keeps vendor-specific HTTP logic isolated from the MCP tool definitions.

```python
class CrowdStrikeClient:
    """HTTP client for the CrowdStrike Falcon API."""

    def __init__(self, base_url: str, client_id: str, client_secret: str):
        self._base_url = base_url
        self._client_id = client_id
        self._client_secret = client_secret
        self._http = httpx.AsyncClient(base_url=base_url, timeout=30)
        self._token: str | None = None

    async def get_device(self, device_id: str) -> CrowdStrikeDevice:
        """Fetch a single device by ID. Returns vendor-specific schema."""
        await self._ensure_authenticated()
        response = await self._http.get(
            f"/devices/entities/devices/v2",
            params={"ids": device_id},
            headers={"Authorization": f"Bearer {self._token}"},
        )
        response.raise_for_status()
        return CrowdStrikeDevice.model_validate(response.json()["resources"][0])
```

### 5.5 Vendor Schema Definitions

Each MCP server directory contains a `schemas.py` with Pydantic models for the vendor's raw API responses. These schemas are internal to the adapter — they must never appear in agent.core, agent.api, or agent.web.

```python
class CrowdStrikeDevice(BaseModel):
    """Raw CrowdStrike device payload — internal to adapter."""
    device_id: str
    hostname: str | None = None
    platform_name: str | None = None
    os_version: str | None = None
    agent_version: str | None = None
    last_seen: str | None = None
    status: str | None = None
```

### 5.6 Server Configuration

MCP servers read their vendor-specific configuration (API keys, base URLs, credentials) from environment variables. They do not read from `agent.yaml` — that file configures the platform, not individual vendor integrations.

```
# Environment variables per MCP server
CROWDSTRIKE_BASE_URL=https://api.crowdfalcon.com
CROWDSTRIKE_CLIENT_ID=...
CROWDSTRIKE_CLIENT_SECRET=...

SENTINEL_WORKSPACE_ID=...
SENTINEL_TENANT_ID=...
SENTINEL_CLIENT_ID=...
SENTINEL_CLIENT_SECRET=...
```

Each MCP server is responsible for validating that its required environment variables are present at startup.

---

## 6. Storage Adapter (MongoDB)

The MongoDB adapter provides the persistence mechanism for all platform entities. It is a thin operational layer — it handles connection management, serialization, and index management. It does not interpret the data it stores.

### 6.1 Responsibilities

- Manage MongoDB connection lifecycle (connect, health check, disconnect)
- Provide typed CRUD operations for each collection
- Manage collection indexes
- Ensure write idempotency where required (upserts keyed by natural keys)
- Handle serialization/deserialization between Pydantic models and MongoDB documents
- Classify infrastructure errors (connection failure, timeout, duplicate key)

### 6.2 Interface

```python
class MongoAdapter:
    """Thin persistence layer over MongoDB."""

    async def connect(self, config: MongoDBConfig) -> None:
        """Establish connection. Called once at startup."""

    async def disconnect(self) -> None:
        """Close connection. Called at shutdown."""

    # --- Runs ---
    async def insert_run(self, run: Run) -> None: ...
    async def update_run(self, run_id: str, updates: dict) -> None: ...
    async def get_run(self, run_id: str) -> Run | None: ...
    async def list_runs(
        self, status: RunStatus | None = None, limit: int = 50, offset: int = 0
    ) -> list[Run]: ...

    # --- Steps ---
    async def insert_step(self, step: Step) -> None: ...
    async def update_step(self, step_id: str, run_id: str, updates: dict) -> None: ...
    async def get_steps_for_run(self, run_id: str) -> list[Step]: ...

    # --- Events ---
    async def insert_events(self, events: list[Event]) -> None:
        """Batch insert. Idempotent on run_id + sequence."""
    async def get_events_for_run(
        self, run_id: str, since_sequence: int = 0
    ) -> list[Event]: ...

    # --- Run Context ---
    async def upsert_run_context(self, context: RunContext) -> None: ...
    async def get_run_context(self, run_id: str) -> RunContext | None: ...

    # --- Approvals ---
    async def insert_approval(self, approval: Approval) -> None: ...
    async def update_approval(self, approval_id: str, updates: dict) -> None: ...
    async def get_approvals_for_run(self, run_id: str) -> list[Approval]: ...

    # --- Artifacts ---
    async def insert_artifact(self, artifact: Artifact) -> None: ...
    async def get_artifacts_for_run(self, run_id: str) -> list[Artifact]: ...

    # --- Assets ---
    async def upsert_asset(self, asset: Asset) -> None:
        """Upsert on external_id + source."""
    async def get_asset(self, asset_id: str) -> Asset | None: ...
    async def get_assets_for_run(self, run_id: str) -> list[Asset]: ...

    # --- Registry (workflow and skill definitions) ---
    async def upsert_workflow(self, definition: dict) -> None:
        """Upsert on workflow_id + version."""
    async def get_workflow(self, workflow_id: str, version: str | None = None) -> dict | None: ...
    async def list_workflows(self, tags: list[str] | None = None) -> list[dict]: ...

    async def upsert_skill(self, definition: dict) -> None:
        """Upsert on skill_id + version."""
    async def get_skill(self, skill_id: str, version: str | None = None) -> dict | None: ...
    async def list_skills(self) -> list[dict]: ...
```

### 6.3 Collections and Indexes

| Collection | Primary Key / Unique Index | Additional Indexes |
|------------|---------------------------|-------------------|
| runs | run_id | status, created_at |
| steps | step_id + run_id | run_id |
| events | run_id + sequence | run_id, event_type |
| run_contexts | run_id | — |
| approvals | approval_id | run_id |
| artifacts | artifact_id | run_id |
| assets | external_id + source | run_id, asset_type |
| workflow_definitions | workflow_id + version | tags |
| skill_definitions | skill_id + version | — |

### 6.4 Rules

- The adapter accepts and returns canonical contract types (from agent.core/contracts/) and workflow/skill definition dicts (from agent.workflows)
- The adapter must not contain business logic — no interpretation of run status, no step sequencing, no event ordering decisions
- All writes that could be retried must be idempotent (upserts on natural keys)
- Connection configuration comes from the validated `MongoDBConfig` in AgentConfig
- The adapter uses motor (async MongoDB driver) for all operations

---

## 7. Cache Adapter (Valkey)

The Valkey adapter provides the caching and buffering mechanism for active run state and SSE event delivery. It is a thin operational layer.

### 7.1 Responsibilities

- Manage Valkey connection lifecycle
- Provide typed key-value operations with TTL enforcement
- Handle SSE event buffer operations (append, read since sequence, trim)
- Handle RunContext caching (set, get, delete)
- Handle paused graph state storage (for AWAITING_APPROVAL runs)
- Handle run status caching
- Classify infrastructure errors

### 7.2 Interface

```python
class ValkeyAdapter:
    """Thin caching and buffering layer over Valkey."""

    async def connect(self, config: ValkeyConfig) -> None:
        """Establish connection. Called once at startup."""

    async def disconnect(self) -> None:
        """Close connection. Called at shutdown."""

    # --- Run Context Cache ---
    async def set_run_context(self, run_id: str, context: RunContext) -> None:
        """Cache with TTL from config.run_context_ttl_seconds."""

    async def get_run_context(self, run_id: str) -> RunContext | None:
        """Returns None on cache miss."""

    async def delete_run_context(self, run_id: str) -> None: ...

    # --- SSE Event Buffer ---
    async def append_event(self, run_id: str, event: Event) -> None:
        """Append to ordered event list. TTL from config.sse_buffer_ttl_seconds."""

    async def get_events_since(self, run_id: str, since_sequence: int) -> list[Event]:
        """Retrieve events with sequence > since_sequence."""

    # --- Run Status ---
    async def set_run_status(self, run_id: str, status: RunStatus) -> None:
        """Cache current status. TTL from config.run_context_ttl_seconds."""

    async def get_run_status(self, run_id: str) -> RunStatus | None: ...

    # --- Paused Graph State ---
    async def set_graph_state(self, run_id: str, state: bytes) -> None:
        """Store serialized graph state. TTL from config.paused_graph_ttl_seconds."""

    async def get_graph_state(self, run_id: str) -> bytes | None: ...

    async def delete_graph_state(self, run_id: str) -> None: ...
```

### 7.3 Key Patterns

| Key Pattern | Contents | TTL Source |
|-------------|----------|-----------|
| run:{run_id}:context | Serialized RunContext | cache.run_context_ttl_seconds |
| run:{run_id}:events | Ordered event list | cache.sse_buffer_ttl_seconds |
| run:{run_id}:state | Current RunStatus | cache.run_context_ttl_seconds |
| run:{run_id}:graph | Serialized paused GraphState | cache.paused_graph_ttl_seconds |

### 7.4 Rules

- The adapter accepts and returns canonical contract types and serialized bytes (for graph state)
- All TTL values come from the validated `ValkeyConfig` in AgentConfig — no hardcoded defaults
- The adapter must not contain business logic — no decisions about when to cache or what to cache
- Serialization format for RunContext and Event is JSON (via Pydantic's `.model_dump_json()`)
- Serialization format for graph state is opaque bytes (Pydantic AI handles serialization; the adapter stores and retrieves without interpretation)
- Cache misses return None — they are not errors
- The adapter uses redis.asyncio (Valkey is Redis-protocol compatible) for all operations
- Write failures to Valkey must be logged but must not raise exceptions that block execution (fire-and-forget for RunContext writes; SSE buffer writes are best-effort)

### 7.5 Failure Mode

If Valkey is unavailable:

- **RunContext writes**: Logged and skipped. MongoDB is the durable store; Valkey is a performance optimization. The runtime continues.
- **SSE event buffer writes**: Logged and skipped. Events are still persisted to MongoDB. SSE reconnection with `?since_sequence=N` falls back to MongoDB query.
- **Run status cache**: Logged and skipped. Status is always available from MongoDB.
- **Paused graph state**: This is the critical path. If Valkey is unavailable when a run reaches AWAITING_APPROVAL, the graph state must be serialized to MongoDB as a fallback. On resume, agent.core checks Valkey first, then MongoDB. Loss of graph state means the run cannot be resumed — it should be transitioned to FAILED with a diagnostic error.

---

## 8. Observability

### 8.1 MCP Server Instrumentation

Every MCP server must instrument:

- **Vendor API call latency**: Span per HTTP call to the vendor, with tool_id, server_id, and HTTP status as attributes
- **Normalization**: Log when normalization produces warnings (e.g., missing fields defaulted)
- **Error classification**: Structured log on vendor errors with error category and raw error detail

Span naming convention: `agent.adapters.{server_id}.{operation}`

Examples:
- `agent.adapters.crowdstrike.get_device`
- `agent.adapters.sentinel.run_query`

### 8.2 Storage Adapter Instrumentation

- Span per MongoDB operation with collection name and operation type as attributes
- Latency tracking per operation
- Error logging with classification (connection failure, timeout, duplicate key)

Span naming: `agent.adapters.storage.{operation}`

### 8.3 Cache Adapter Instrumentation

- Span per Valkey operation with key pattern and operation type as attributes
- Cache hit/miss ratio logging
- Error logging (connection failure, serialization error)

Span naming: `agent.adapters.cache.{operation}`

### 8.4 structlog Integration

All adapter components use structlog with bound context. MCP servers bind `server_id`. Storage and cache adapters bind the operation context.

```python
log = structlog.get_logger().bind(server_id="crowdstrike")
```

---

## 9. Boundary Rules

agent.adapters enforces these rules:

- **No canonical contract definitions** — consumed from agent.core/contracts/, never redefined or duplicated
- **No canonical meaning** — the adapter normalizes data into canonical shapes but does not interpret what those shapes mean
- **No execution logic** — no run management, no step sequencing, no event emission decisions
- **No import of agent.core runtime components** — adapters may import canonical contract schemas for normalization reference, but not AgentRuntime, MCPClient, GraphState, etc.
- **No import of agent.api or agent.web**
- **Vendor schemas are internal** — external Pydantic models in `schemas.py` must not leak into agent.core, agent.api, or agent.web
- **Vendor credentials are self-managed** — each MCP server reads its own credentials from environment variables, not from agent.yaml
- **All vendor HTTP calls are in the vendor client** — MCP tool functions call the client, not HTTP directly

---

## 10. Adding a New Integration

To add a new external integration:

1. Create a new directory under `agent.adapters/mcp_servers/{vendor_name}/`
2. Implement `schemas.py` — Pydantic models for the vendor's raw API responses
3. Implement `client.py` — async HTTP client wrapping the vendor's API
4. Implement `normalizer.py` — mapping functions from vendor schemas to canonical contract shapes
5. Implement `server.py` — MCP server with tool definitions that use the client and normalizer
6. Add the server's URL to `agent.yaml` under `mcp_servers` with appropriate `required` flag
7. Set vendor credentials as environment variables
8. Add Logfire instrumentation (vendor call spans, error classification)
9. Restart the platform — the MCP client in agent.core will discover the new server's tools at startup

No changes to agent.core, agent.api, or agent.web are required. The new tools appear automatically in the tool catalog and are available to both workflow and autonomous execution.

---

## 11. Extension Points

**To add a new storage backend** (e.g., PostgreSQL):

1. Implement a new adapter with the same interface as `MongoAdapter`
2. Add a configuration section to AgentConfig
3. Update agent.core to accept the new adapter via dependency injection
4. This requires an agent.core change — the storage adapter interface should be extracted to an abstract base class if multiple backends are anticipated

**To add a new cache backend:**

Same pattern as storage — implement the `ValkeyAdapter` interface, update config and dependency injection.

---

## 12. Implementation Notes for Claude Code

- Use `httpx.AsyncClient` for all vendor HTTP calls — it is the standard async HTTP client in the stack
- Use `motor` for the MongoDB adapter — the async driver for MongoDB
- Use `redis.asyncio` for the Valkey adapter — Valkey speaks Redis protocol
- MCP servers are independent processes — they have their own `__main__.py` entry point and can be started separately
- Normalizers should be pure functions where possible (no side effects, no I/O) — they take a vendor schema and return a canonical shape
- Vendor schemas should use `None` defaults liberally — vendor APIs frequently omit fields
- The storage adapter should create indexes on first connection if they don't exist (idempotent index creation)
- All adapter operations are async — no synchronous blocking
- Vendor credentials must never be logged, even at DEBUG level

---

## 13. Example Execution Trace

This trace shows how agent.adapters participates in a workflow step that invokes a CrowdStrike tool.

```
1. agent.core AgentNode executes an AGENT step
   └── LLM selects tool: crowdstrike.get_device

2. agent.core MCPClient.invoke("crowdstrike.get_device", {"device_id": "abc123"}, ...)
   ├── Validates input against tool input_schema → passes
   ├── Emits tool.invoked event
   └── Sends MCP request to CrowdStrike MCP server

3. agent.adapters CrowdStrike MCP server receives request
   └── server.py get_device() handler
       ├── client.py → HTTP GET /devices/entities/devices/v2?ids=abc123
       │   ├── Logfire span: agent.adapters.crowdstrike.get_device (latency: 230ms)
       │   └── Returns raw CrowdStrikeDevice
       ├── normalizer.py → normalize_device(raw)
       │   └── Returns canonical Asset shape
       └── Returns normalized response to MCP client

4. agent.core MCPClient receives response
   ├── Validates output against tool output_schema → passes
   ├── Emits tool.result event
   └── Returns ToolInvocationResult to AgentNode

5. agent.core AgentNode writes result to RunContext
   └── RunContext persisted via storage adapter
       ├── MongoAdapter.upsert_run_context() → MongoDB
       └── ValkeyAdapter.set_run_context() → Valkey (fire-and-forget)
```