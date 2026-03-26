# Domain: Error Handling

## Domain Name

Error Handling

## What This Domain Governs

How errors are modeled, classified, propagated, and recovered from across the system. This domain assesses whether the system treats errors as first-class entities with consistent structure, whether failures propagate with context rather than being silently swallowed, and whether retry and resilience mechanisms are safe and bounded.

## Why It Matters

Agentic systems operate across multiple failure boundaries: LLM calls can timeout or hallucinate, tools can fail or return unexpected results, infrastructure can become unavailable, and orchestration logic can encounter states it was not designed for. How the system handles these failures determines whether operators can diagnose problems, whether the system degrades gracefully, and whether failures cascade into catastrophic states.

Error handling gaps compound. An unstructured error model means failures lose context as they propagate. Missing classification means the system cannot distinguish between retriable and terminal failures. Silent error swallowing means the system continues in an indeterminate state without any signal that something went wrong. Unbounded retries amplify transient failures into sustained outages.

## Principles in This Domain

| ID | Title | Severity |
|----|-------|----------|
| P6.1 | Structured Error Model | HIGH |
| P6.2 | Error Classification | MEDIUM |
| P6.3 | Failure Propagation | CRITICAL |
| P6.4 | Infrastructure Failure Resilience | HIGH |
| P6.5 | Retry Safety | MEDIUM (unbounded) / LOW (absent) |

## Common Failure Modes

- **Unstructured errors** — errors represented as bare strings or generic exception messages with no classification, context, or diagnostic detail
- **Silent error swallowing** — `except: pass`, empty catch blocks, or errors caught and discarded without recording, leaving the system in an indeterminate state
- **Missing error classification** — all errors treated identically regardless of whether they are transient, terminal, validation failures, or infrastructure failures
- **Context-free propagation** — errors re-raised or returned without the originating step, component, or input context, making root cause analysis impossible
- **Unbounded retries** — retry loops without maximum attempt counts or backoff, turning transient failures into sustained resource consumption
- **Infrastructure failures crashing the system** — a cache miss, metrics endpoint timeout, or logging service outage bringing down the entire agent execution

## Typical Repository Indicators

Look for these when forming the system hypothesis:

- Error model or error class definitions (custom exception hierarchies, error types, result types)
- Error classification enums or constants (error codes, error categories)
- Exception handler patterns (try/catch blocks, error middleware, result unwrapping)
- Retry logic (retry decorators, retry loops, backoff configuration)
- Circuit breaker or resilience patterns (fallback handlers, degradation paths)
- Infrastructure integration error handling (database connection failures, API timeouts, cache failures)

## Common Trace Types to Inspect

- **Error model trace** — find the error model definition and trace how it is constructed, populated, and consumed across modules
- **Failure propagation trace** — trigger or find a failure in a tool call or LLM invocation and follow it from origin through intermediate handlers to the final run-level outcome
- **Retry behavior trace** — find retry logic and check for bounds, backoff, and idempotency safety
- **Infrastructure failure trace** — find external service integrations and check what happens when they fail

## Relationship to Adjacent Domains

- **Observability (Domain 3):** P3.5 (Failure Contextualization) overlaps with P6.1 (Structured Error Model). P3.5 assesses whether failures are observable; P6.1 assesses whether errors are structured as first-class entities. Both can be gaps independently.
- **Orchestration Model (Domain 7):** P7.2 (Explicit State Transitions) depends on P6.3 (Failure Propagation) — if failures do not propagate, state transitions may silently skip from running to a terminal state without recording the failure.
- **Integration and Tool Model (Domain 4):** Tool invocation failures are a primary source of errors. P6.2 (Error Classification) determines whether tool failures can be distinguished from LLM failures or validation errors.

## Common False Positives in Assessment

- **Presence of a custom exception class does not mean structured errors** — check that the class carries classification, context, and detail fields, not just a message string
- **Retry logic existence does not mean retry safety** — check for bounds, backoff, and idempotency considerations
- **Error logging does not mean error propagation** — logging an error and continuing execution is not the same as propagating the failure to the run level
- **Try/catch blocks do not mean error handling** — an empty catch block or a catch that only logs and re-raises a generic exception does not constitute structured error handling
