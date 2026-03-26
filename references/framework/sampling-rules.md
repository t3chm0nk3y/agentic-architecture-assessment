# Sampling Rules

This document defines bounded evidence collection rules for assessments. The framework does not require exhaustive reading of every file — it requires representative evidence sufficient for confident verdicts.

---

## Core Sampling Principle

Assess representative paths, not every path. A principle is evaluated against the system's typical behavior, not its complete behavior. Exhaustive coverage is neither practical nor necessary for an architectural audit.

---

## Sampling Targets

For each applicable domain, collect evidence from at least:

### Execution Paths

- **One representative success path** — trace a request from entry to completion through the full execution pipeline
- **One representative failure path** — trace a request that fails and verify error handling, propagation, and state transitions

### Tool-Mediated Flows (if tool-enabled)

- **One representative tool invocation** — trace from tool selection through invocation to result capture and propagation
- **One representative tool failure** — trace what happens when a tool call fails

### Approval-Gated Flows (if approval-gated)

- **One representative approval flow** — trace from gate trigger through pause to resolution and resume

### Schema Boundaries

- **One representative boundary crossing per major layer transition** — trace data from external source through normalization to internal consumption to API response

### Configuration

- **The startup sequence** — trace from config load through validation to runtime initialization
- **One representative config-driven behavior** — confirm that a runtime behavior is controlled by configuration, not hardcoded

---

## Sampling Priority

When time or context is limited, prioritize evidence collection in this order:

1. **Central execution paths** — the main agent loop, runtime entry point, dispatch mechanism
2. **Error handling paths** — what happens when things fail
3. **Schema boundaries** — where data crosses module boundaries
4. **Tool invocation paths** — how external systems are accessed
5. **Configuration and startup** — how the system is initialized
6. **Observability infrastructure** — logging, tracing, event emission
7. **Extension mechanisms** — how new workflows, tools, or skills are added
8. **Safety mechanisms** — tool access control, guardrails, approval gates

---

## When to Sample Deeper

Expand sampling beyond the minimum when:

- An unexpected finding suggests a systemic issue (e.g., finding one unvalidated boundary suggests others may also be unvalidated)
- A principle is borderline between PASS and PARTIAL — additional evidence may resolve it
- The system is large enough that one sample may not be representative
- The existing sample reveals inconsistent behavior across modules

---

## When Not to Over-Sample

Do not expand sampling when:

- The first sample clearly confirms or denies the principle
- Further reading would be repetitive (same pattern in every module)
- The finding is already well-evidenced with high confidence
- The principle is clearly N/A for this system type

---

## Large Codebase Strategy

For codebases exceeding ~10,000 lines:

1. Read architecture docs and entry points first (Phase 1 — Reconnaissance)
2. Form the system hypothesis before reading source code
3. Use the hypothesis to identify the 3-5 highest-risk areas
4. Sample those areas first
5. For remaining domains, use one representative path each
6. If findings are concentrated in one area, sample adjacent areas for the same pattern
7. Do not read utility functions, test files, or generated code unless directly relevant to a finding
