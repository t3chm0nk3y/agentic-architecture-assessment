# Domain: Observability

## Domain Name

Observability

## What This Domain Governs

How execution is traced, logged, monitored, and surfaced across the system. This domain assesses whether the system produces enough structured, correlated information for operators to understand what happened during any given execution — during and after the fact.

## Why It Matters

An agentic system that cannot be observed in production is an agentic system that cannot be debugged, audited, or trusted. Agent executions involve multiple steps, tool invocations, LLM reasoning, and state transitions. Without structured observability, failures manifest as black boxes — operators know something went wrong but cannot determine what, where, or why.

Observability gaps compound. A missing trace identifier makes structured logs useless for correlation. Unstructured logs make event streams the only debugging path. Incomplete event coverage makes the event stream unreliable. Each gap erodes the next layer of defense.

## Principles in This Domain

| ID | Title | Severity |
|----|-------|----------|
| P3.1 | Run Traceability | CRITICAL |
| P3.2 | Structured Logging | HIGH |
| P3.3 | Structured Events | HIGH |
| P3.4 | Event Delivery Reliability | MEDIUM |
| P3.5 | Failure Contextualization | HIGH |
| P3.6 | Observability Coverage | MEDIUM |

## Common Failure Modes

- **No correlation identifier** — logs from concurrent runs are interleaved and impossible to separate
- **Free-form logging** — `print()` or `logging.info("something happened")` without structured context
- **Incomplete event model** — some lifecycle transitions emit events, others are silent
- **No reconnection support** — SSE/WebSocket clients that disconnect lose events permanently
- **Bare exceptions** — `except Exception: pass` or error messages with no execution context
- **Observability concentrated in one module** — the runtime is fully traced but adapters and integrations are dark

## Typical Repository Indicators

Look for these when forming the system hypothesis:

- Logging framework configuration (structlog, winston, zerolog, slog, etc.)
- Event emission code (event buses, SSE endpoints, WebSocket handlers)
- Tracing setup (OpenTelemetry, Logfire, Datadog, Jaeger)
- Correlation/trace ID generation (middleware, context propagation)
- Error model definitions (error classes, diagnostic context)

## Common Trace Types to Inspect

- **Correlation propagation trace** — follow a trace/correlation ID from generation through logging, events, tool calls, and error handlers
- **Event lifecycle trace** — follow a run from creation through steps to completion, checking for event emission at each transition
- **Error context trace** — trigger or find a failure path and check what diagnostic information is captured

## Relationship to Adjacent Domains

- **Error Handling (Domain 6):** P3.5 (Failure Contextualization) overlaps with P6.1 (Structured Error Model). P3.5 assesses whether failures are observable; P6.1 assesses whether errors are structured. Both can be gaps independently.
- **Execution Boundaries (Domain 1):** P3.6 (Observability Coverage) is affected by execution boundaries — if execution is scattered, observability is harder to achieve consistently.
- **Integration and Tool Model (Domain 4):** P4.4 (Tool Observability) is a specialization of P3.6 for tool invocations.

## Common False Positives in Assessment

- **Presence of a logging library does not mean structured logging** — check that log calls include context binding, not just that the library is imported
- **Event type definitions do not mean events are emitted** — check for actual emission sites, not just enum definitions
- **Trace IDs in documentation do not mean trace IDs in code** — verify propagation in implementation
- **Console output does not mean observability** — `print()` statements are not structured logging
