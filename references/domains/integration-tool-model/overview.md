# Domain: Integration and Tool Model

## Domain Name

Integration and Tool Model

## What This Domain Governs

How the system interacts with external services, tools, and data sources. This domain assesses whether external integrations are structured behind a consistent abstraction layer with proper normalization, discoverability, observability, and authentication isolation.

## Why It Matters

Agentic systems are defined by their ability to act on the world through tools. Every external call — whether to an API, a database, a subprocess, or an LLM provider — represents a coupling point. When these integrations are scattered throughout the codebase, the system accumulates hidden dependencies that resist testing, replacement, and monitoring.

A well-structured integration model ensures that every external interaction passes through a known boundary, transforms data into a canonical internal shape, and produces observable traces. Without this, teams discover integration problems only in production, when a vendor changes an API response format and the breakage propagates through layers that should never have seen raw external data.

## Principles in This Domain

| ID | Title | Severity |
|----|-------|----------|
| P4.1 | Single Integration Surface | HIGH |
| P4.2 | Tool Abstraction and Discoverability | HIGH |
| P4.3 | Normalization at the Adapter Boundary | HIGH |
| P4.4 | Tool Observability | MEDIUM |
| P4.5 | Adapter Self-Contained Authentication | MEDIUM |

> **Note on P4.2:** This domain applies regardless of tool implementation. Recognize MCP servers, LangChain tools, OpenAI function calling schemas, plain function registries, plugin systems, tool decorators as equivalent patterns.

## Common Failure Modes

- **Direct external calls from runtime** — HTTP clients, database drivers, or subprocess calls invoked directly from orchestration or business logic rather than through an adapter layer
- **Scattered tool definitions** — tools defined inline at the point of use rather than registered in a discoverable catalog
- **Raw payload propagation** — vendor-specific JSON structures passed through the runtime unchanged, creating implicit coupling to external schemas
- **Invisible tool calls** — tool invocations that produce no log entry, metric, or event, making tool failures impossible to diagnose
- **Shared credential plumbing** — authentication tokens threaded through function parameters or stored in global state rather than encapsulated within the adapter that needs them

## Typical Repository Indicators

Look for these when forming the system hypothesis:

- Adapter or integration directories (adapters/, integrations/, tools/, connectors/)
- Tool registration code (tool registries, function decorators, tool manifest files)
- HTTP client wrappers or SDK initialization code
- Schema transformation or mapping code near integration boundaries
- Credential management (secret loading, auth token refresh, credential configuration)

## Common Trace Types to Inspect

- **Integration surface trace** — follow an external call from the runtime to the adapter boundary, checking whether the call goes through an abstraction layer or directly to an external client
- **Tool registration trace** — follow a tool definition from declaration through registration to invocation, checking for a discoverable catalog
- **Normalization trace** — follow external response data from the raw vendor payload through transformation to the internal model used by the runtime
- **Credential flow trace** — follow authentication credentials from configuration through to the point of use, checking for encapsulation within the adapter

## Relationship to Adjacent Domains

- **Observability (Domain 3):** P4.4 (Tool Observability) is a specialization of P3.6 (Observability Coverage) for tool invocations. P4.4 assesses whether tools are individually observable; P3.6 assesses system-wide coverage.
- **Schema and Contract Discipline (Domain 5):** P4.3 (Normalization at the Adapter Boundary) is the integration-layer complement of P5.1 (Canonical Contract Centralization). P4.3 ensures external data is normalized; P5.1 ensures internal data entities are defined once.
- **Execution Boundaries (Domain 1):** P4.1 (Single Integration Surface) supports clean execution boundaries by ensuring external calls do not bypass the architectural layers.
- **Configuration (Domain 8):** P4.5 (Adapter Self-Contained Authentication) relates to configuration management of secrets and credentials.

## Common False Positives in Assessment

- **Presence of an adapters directory does not mean all calls go through adapters** — check for direct external calls in runtime code, not just adapter file existence
- **Tool type definitions do not mean tools are registered** — check for actual registration mechanism, not just type annotations on tool functions
- **Response parsing does not mean normalization** — extracting a field from a JSON response is not the same as transforming to a canonical internal model
- **Logging a tool name does not mean tool observability** — check that inputs, outputs, latency, and errors are captured, not just a "called tool X" message
- **Environment variable loading does not mean authentication isolation** — check whether credentials are loaded inside the adapter or loaded externally and passed through
