# Domain: Execution Boundaries

## Domain Name

Execution Boundaries

## What This Domain Governs

How execution responsibility is distributed across system layers. This domain assesses whether the system maintains clear ownership of agent execution in a single runtime module, separates transport concerns from execution semantics, and ensures that consumer layers (API clients, UIs) do not bypass architectural boundaries to drive execution directly.

## Why It Matters

An agentic system where execution logic leaks into multiple layers becomes impossible to reason about, debug, or safely modify. When API route handlers contain agent invocation logic, when UI components call runtime modules directly, or when execution failures are conflated with transport failures, the system loses its architectural integrity. Changes in one layer propagate unpredictably into others. Error handling becomes ambiguous — operators cannot distinguish between a failed HTTP request and a failed agent execution. Testing becomes fragile because execution behavior depends on which entry point was used.

Execution boundary violations compound over time. A small amount of execution logic in a route handler attracts more. A UI that directly imports one runtime module will soon import others. Each boundary crossing makes the next one easier to justify and harder to detect.

## Principles in This Domain

| ID | Title | Severity |
|----|-------|----------|
| P1.1 | Execution Centralization | HIGH |
| P1.2 | Transport / Runtime Separation | HIGH |
| P1.3 | UI as Consumer | MEDIUM |
| P1.4 | Execution Errors Separate from Transport Errors | MEDIUM |

## Common Failure Modes

- **Execution logic scattered across modules** — agent invocation happens in route handlers, background workers, and CLI commands with no shared runtime, making behavior inconsistent across entry points
- **Route handlers that own execution semantics** — API endpoints contain conditional branching, step sequencing, or tool selection logic instead of delegating to a runtime
- **UI importing runtime modules directly** — frontend or client code bypasses the API layer and calls orchestration or integration components, creating invisible coupling
- **Execution failures surfaced as HTTP errors** — a tool failure or LLM timeout returns HTTP 500 instead of a run-level failure state, making it impossible for clients to distinguish transport problems from execution outcomes
- **Multiple agent loop implementations** — different entry points (API, CLI, scheduler) each implement their own version of the execution loop, leading to behavioral drift

## Typical Repository Indicators

Look for these when forming the system hypothesis:

- A dedicated runtime or engine module (runner, executor, engine, orchestrator)
- Route handler files that delegate to a runtime module vs. those that contain execution logic inline
- Import patterns in UI/client code — whether they import API clients or backend runtime modules
- Error response shaping in API handlers — whether execution failures are wrapped in run state or translated to HTTP status codes
- Entry point analysis — how many distinct code paths can initiate an agent execution

## Common Trace Types to Inspect

- **Execution entry point trace** — follow every code path that can start an agent execution to verify they all converge on the same runtime module
- **Route handler trace** — read API route handlers end to end to check whether they delegate execution or contain execution logic
- **Error response trace** — trigger or find a failure path and check whether the HTTP response reflects transport status or execution outcome
- **UI import trace** — examine UI/client module imports to verify they reach only API client layers, not backend runtime modules

## Relationship to Adjacent Domains

- **Orchestration Model (Domain 7):** P1.1 (Execution Centralization) defines where execution lives; Domain 7 defines how execution is structured (steps, loops, state machines). A system can centralize execution in one module but still have a poor orchestration model.
- **Observability (Domain 3):** P3.6 (Observability Coverage) is affected by execution boundaries — if execution is scattered across modules, consistent observability is harder to achieve because each execution path must independently emit traces and events.
- **Error Handling (Domain 6):** P1.4 (Execution Errors Separate from Transport Errors) overlaps with P6.1 (Structured Error Model). P1.4 assesses whether error categories are correctly separated by layer; P6.1 assesses whether errors are structured as first-class entities.
- **Integration and Tool Model (Domain 4):** Execution centralization (P1.1) constrains where tool invocations originate. If tools are called from route handlers or UI code, both P1.1 and P4 principles are likely violated.

## Common False Positives in Assessment

- **A file named "runtime" does not mean execution is centralized** — check that all execution paths converge on that module, not just that a runtime file exists
- **Thin route handlers are not automatically compliant** — a handler that calls a service which itself contains execution logic has merely moved the boundary violation one level deeper
- **Separate error types do not mean separate error handling** — check that execution errors are actually returned through run state, not just that distinct error classes exist
- **Monorepo structure does not imply boundary violations** — UI and runtime code may share a repository while maintaining proper API-only consumption patterns
