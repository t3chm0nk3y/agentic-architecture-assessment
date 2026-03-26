# System Classification

Before assessing any principle, the assessor must classify the system under assessment. Classification drives principle applicability — it determines which principles are relevant, which are N/A, and how to interpret ambiguous evidence.

---

## Classification Dimensions

Classify the system along each of these dimensions. A system may have characteristics from multiple options within a dimension (mark as hybrid where applicable).

### 1. Interaction Model

| Classification | Description |
|---------------|-------------|
| **Conversational** | Responds to user messages in a turn-based dialogue; no predefined task structure |
| **Workflow-driven** | Executes predefined, structured task sequences with known steps |
| **Hybrid** | Supports both conversational and workflow-driven interactions |

### 2. Tool Capability

| Classification | Description |
|---------------|-------------|
| **Tool-enabled** | The agent can invoke external tools, APIs, or services during execution |
| **Non-tool** | The agent operates on LLM reasoning alone with no external tool access |

### 3. Approval Model

| Classification | Description |
|---------------|-------------|
| **Approval-gated** | Some actions require human approval before execution |
| **Ungated** | All actions execute without human intervention |

### 4. Autonomy Level

| Classification | Description |
|---------------|-------------|
| **Autonomous** | The agent determines its own sequence of actions toward a goal |
| **Constrained** | The agent follows predefined steps; it executes but does not plan |
| **Hybrid** | Supports both autonomous and constrained execution modes |

### 5. Agent Cardinality

| Classification | Description |
|---------------|-------------|
| **Single-agent** | One agent instance handles a request end to end |
| **Multi-agent** | Multiple agents collaborate, delegate, or orchestrate each other |

### 6. Execution Model

| Classification | Description |
|---------------|-------------|
| **Synchronous** | Request-response; the caller waits for completion |
| **Asynchronous** | Execution happens in the background; results are retrieved later or streamed |
| **Both** | Some operations are synchronous, some are asynchronous |

### 7. Interface Type

| Classification | Description |
|---------------|-------------|
| **API-only** | Exposed only as an HTTP/gRPC/similar API; no user-facing UI |
| **UI-backed** | Has a frontend UI that consumes the API |
| **CLI** | Command-line tool; no API or UI layer |
| **Notebook** | Runs in a Jupyter/Colab/equivalent notebook environment |
| **Library/SDK** | Consumed as a library; no standalone execution interface |
| **Both (API + UI)** | Has both an API and a UI |

### 8. Streaming Capability

| Classification | Description |
|---------------|-------------|
| **Streaming** | Supports real-time event or token streaming during execution |
| **Non-streaming** | Results are available only upon completion |

---

## Classification-Driven Applicability

This table maps system characteristics to domain applicability. Use it to quickly determine which domains and principles require assessment.

### Domains Always Applicable

These domains apply to every agentic system regardless of classification:

- **Configuration** — every system has configuration
- **Error Handling** — every system can fail

### Domains Conditionally Applicable

| Domain | Applicable When | Typically N/A When |
|--------|----------------|-------------------|
| Execution Boundaries | Multi-module system with distinct layers | Single-file script with no module separation |
| Observability | Any system with logging, tracing, or event emission | Trivial single-file scripts (but even these benefit from structured logging) |
| Integration and Tool Model | Tool-enabled systems | Non-tool systems (but assess if external calls exist regardless) |
| Schema and Contract Discipline | Systems with defined data entities across boundaries | Single-module systems with no boundary crossings |
| Orchestration Model | Systems with multi-step execution, state machines, or agent loops | Single-turn, stateless agents with no persistent state |
| Scalability and Extensibility | Systems designed for multiple workflows, tools, or skills | Fixed-purpose, single-function agents |
| Autonomous Agent Safety | Tool-enabled autonomous agents | Constrained agents with no tool access or read-only tools |

### Principle-Level N/A Mapping

| Principle | N/A When |
|-----------|----------|
| P1.2 (Transport/Runtime Separation) | No HTTP/API layer (CLI, notebook, library) |
| P1.3 (UI as Consumer) | No UI layer |
| P1.4 (Execution Errors Separate from Transport) | No HTTP/API layer |
| P2.3 (Startup Safety Gate) | No startup sequence (notebook, REPL) |
| P3.3 (Structured Events) | Synchronous single-turn agent with no persistent state or streaming |
| P3.4 (Event Delivery Reliability) | No real-time event streaming interface |
| P5.2 (Definition vs Runtime Records) | No workflow/skill definitions — fully autonomous, definition-free |
| P5.3 (DTO Isolation) | No HTTP/API layer |
| P7.2 (Explicit State Transitions) | Stateless single-turn agent |
| P7.4 (Deterministic Workflow Structure) | No structured workflow concept — fully autonomous |
| P7.5 (Run State Auditability) | Explicitly ephemeral system with no persistence requirement |
| P8.1 (Workflow Extensibility) | No workflow concept |
| P8.3 (Skill Extensibility) | No skill or persona concept |
| P8.5 (Separation of Definition from Execution) | No definition layer |
| P9.1-P9.2 (Tool Access Control) | Non-tool system or all actions are read-only |
| P9.3 (Human-in-the-Loop) | All actions are read-only |
| P9.5 (Truncation Not Failure) | No guardrails (also captured as GAP under P7.3) |

---

## How to Classify

1. Read the system's documentation, entry points, and configuration
2. Determine the classification for each dimension
3. Record the classification in the report's System Classification section
4. Use the applicability tables above to mark principles as N/A where appropriate
5. For ambiguous cases, err toward assessment (assess the principle rather than marking N/A)
