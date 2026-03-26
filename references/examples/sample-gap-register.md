# Sample Gap Register

This document provides a realistic example of a gap register report produced by the assessment framework. Use it for calibration and consistency when writing real reports.

---

## Example: API-Backed Agent Platform Assessment

### Header

```markdown
# Agentic Architecture Assessment — Example Platform
**Date:** 2026-03-15
**Auditor:** Claude Code
**Evidence Sources:** docs/design-spec.md, src/runtime/, src/api/, src/adapters/, config.yaml, tests/
**System Type:** API-backed agent platform with workflow and autonomous execution
```

### System Classification

| Dimension | Classification |
|-----------|---------------|
| Interaction model | Hybrid (workflow-driven + autonomous) |
| Tool capability | Tool-enabled |
| Approval model | Approval-gated |
| Autonomy level | Hybrid |
| Agent cardinality | Single-agent |
| Execution model | Asynchronous |
| Interface type | Both (API + UI) |
| Streaming | Streaming (SSE) |

### Conformance Overview

| Domain | Assessed | Pass | Gap | Partial | N/A | Unassessable |
|--------|----------|------|-----|---------|-----|--------------|
| Execution Boundaries | 4 | 3 | 0 | 1 | 0 | 0 |
| Configuration | 4 | 3 | 1 | 0 | 0 | 0 |
| Observability | 6 | 4 | 0 | 2 | 0 | 0 |
| Integration and Tool Model | 5 | 4 | 0 | 0 | 0 | 1 |
| Schema and Contract Discipline | 5 | 3 | 1 | 1 | 0 | 0 |
| Error Handling | 5 | 4 | 0 | 1 | 0 | 0 |
| Orchestration Model | 5 | 5 | 0 | 0 | 0 | 0 |
| Scalability and Extensibility | 5 | 5 | 0 | 0 | 0 | 0 |
| Autonomous Agent Safety | 5 | 4 | 0 | 1 | 0 | 0 |
| **Total** | **44** | **35** | **2** | **6** | **0** | **1** |

### Executive Summary

The Example Platform is a well-structured agent runtime with strong module boundaries and centralized execution. Overall conformance is strong: 35 of 44 principles pass cleanly. The two most critical findings are a hardcoded LLM model name that breaks portability (P2.1) and LLM output flowing to downstream steps without schema validation (P5.5). Both are HIGH severity with straightforward remediation paths.

---

### Gap Register

#### HIGH

##### GAP-1: P2.1 — Externalized Configuration
**Verdict:** GAP
**Evidence Status:** confirmed
**Confidence:** high
**Finding:** The LLM model identifier is hardcoded in the runtime source code. All other configuration values (timeouts, execution limits, connection strings, TTLs, server URLs) are correctly externalized to the configuration file via the validated config model. This single hardcoded value means the model cannot be changed without a code deployment.
**Evidence:** `src/runtime/agent.py:265` — `model="vendor:model-name-v1"` hardcoded in the agent execution function. Contrast with `config.yaml` where all other runtime parameters are externalized and validated at startup.
**Risk:** Cannot upgrade model versions, switch models for cost/performance, or test against different models without modifying source code. Breaks the portability guarantee.
**Affected Components:** `src/runtime/agent.py`
**Remediation:** Add a `model` field to the configuration model and read it in the agent execution function. One config addition, one runtime change.
**Verification:** After fix, change the model name in config and verify the runtime uses the new value without code changes. | Behavioral verification needed: No

##### GAP-2: P5.5 — LLM Output Validation
**Verdict:** GAP
**Evidence Status:** contradictory
**Confidence:** high
**Finding:** LLM outputs from the agent step are parsed from JSON (or wrapped as raw text) and written directly to the shared execution state without validation against the skill's output schema. A dedicated validation function exists in the schema module but is never invoked anywhere in the codebase — zero call sites. Raw, unvalidated LLM text can flow into downstream workflow steps, API responses, and persistent storage.
**Evidence:** `src/runtime/agent.py:275-287` (output parsed but not validated); `src/graph/nodes/agent_node.py:98-101` (output written to context without validation); `src/schemas/output_format.py:8` (validation function defined but zero call sites found via grep).
**Risk:** Malformed LLM output corrupts downstream step inputs, producing cascading failures that are hard to diagnose. A classification step returning free text instead of structured JSON would break subsequent steps without any error being raised.
**Affected Components:** `src/runtime/agent.py`, `src/graph/nodes/agent_node.py`, `src/schemas/output_format.py`
**Remediation:** Call the existing validation function after LLM output is received in the agent node. Decide on validation failure behavior: retry, fail step, or warn. The validation function already exists — it just needs to be wired up.
**Verification:** After fix, submit an agent step with a skill that declares an output schema and verify that non-conforming LLM output triggers validation failure. | Behavioral verification needed: Yes

---

#### MEDIUM

##### PARTIAL-1: P3.3 — Structured Events (Dead Event Type)
**Verdict:** PARTIAL
**Evidence Status:** confirmed
**Confidence:** high
**Finding:** The event type enum defines 14 event types. Thirteen are actively emitted throughout the codebase. One event type (`llm.token`) is defined but never emitted — it is dead code. The SSE streaming contract promises clients can reconstruct run state from events alone; missing token streaming means clients cannot display real-time LLM output during agent steps.
**Evidence:** `src/contracts/event.py:21` (type defined); grep for the event type across the entire codebase returns only the enum definition, no emission sites.
**Risk:** Frontend streaming components would receive no events for real-time LLM output display. Not blocking for correctness, but degrades the user experience.
**Affected Components:** `src/contracts/event.py`, streaming infrastructure
**Remediation:** Either implement token streaming by emitting the event during LLM calls, or remove the dead event type from the enum to avoid a false contract.
**Verification:** After fix, start an agent step and verify token events appear in the SSE stream, OR verify the event type is removed from the enum. | Behavioral verification needed: No

---

#### UNASSESSABLE

##### UA-1: P4.5 — Adapter Self-Contained Authentication
**Reason:** External integrations are currently stubbed — connection code registers servers but does not perform real HTTP connections or authentication. Cannot assess whether adapters manage their own credentials until production integrations are implemented. Evidence needed: a working external integration with real API credentials.

---

### Passed Principles

- **P1.1** — Execution centralized in the runtime module; no other module initiates runs.
- **P1.2** — API route handlers delegate to runtime; no business logic in handlers.
- **P1.3** — Frontend imports only from its API client; zero imports from backend runtime.
- **P2.2** — Config model validates all configuration at startup; invalid config halts startup.
- **P2.3** — Required dependencies block startup on failure; optional degrade gracefully.
- **P2.4** — Single config source parsed to validated model; no duplicate config sources.
- **P3.1** — Correlation ID generated at API layer, propagated to all events and trace spans.
- **P3.2** — Structured logging used in all modules; no print statements or unstructured logging found.
- **P3.4** — SSE events buffered with sequence numbers; reconnection supported.
- **P3.5** — Error entity captures classification, message, detail, and all context IDs.
- **P4.1** — All external interactions via centralized tool client; no direct HTTP calls in runtime.
- **P4.2** — Tool catalog built at startup from discovery; queryable via API.
- **P4.3** — Normalization at adapter boundary, validation at runtime boundary.
- **P4.4** — Tool invocations emit events with tool ID, latency, and server ID.
- **P5.1** — All canonical entities defined in one contracts directory; no duplicates found.
- **P5.2** — Definition schemas clearly separated from runtime records.
- **P5.3** — DTOs in API layer; contracts in runtime; no cross-imports.
- **P6.1** — Error model with classification enum, message, detail, retryable flag, context IDs.
- **P6.2** — Error classification enum with 7 distinct types.
- **P6.3** — Step failures propagate to run with error set; run reaches terminal state.
- **P6.4** — Cache writes are fire-and-forget; required services fail startup; optional degrade.
- **P7.1** — Graph-based orchestration framework selected as deliberate architectural choice.
- **P7.2** — Valid transitions enforced by state machine; invalid transitions raise errors.
- **P7.3** — Autonomous execution bounded by max iterations, tool calls, and tokens.
- **P7.4** — Workflow steps defined in YAML, known before execution; autonomous mode separate.
- **P7.5** — Run records persist version, step outputs, tool invocation I/O, events, status.
- **P8.1** — Workflows loaded from directory at startup; adding workflow = adding a file.
- **P8.2** — Tools discovered from servers at startup; adding tools = configuring servers.
- **P8.3** — Skills loaded from directory; system prompts in skill files, not inline.
- **P8.4** — Behavior varied via config and definition files, not code changes.
- **P8.5** — Runtime references no specific definition names; executes any conforming definition.
- **P9.1** — Skills declare tool restrictions; enforced during agent step execution.
- **P9.2** — Tool filtering takes restrictive intersection of active skills.
- **P9.3** — Approval gate step type pauses execution; approval API resolves.
- **P9.4** — System prompts managed as versioned skill files; not inline in code.

---

### Behavioral Verification Register

| Principle | What Needs Verification | Suggested Test Case |
|-----------|------------------------|-------------------|
| P7.3 | Guardrail enforcement under real execution | Exhaust max_iterations and verify COMPLETED with truncation, not FAILED |
| P9.2 | Tool restriction composition under conflict | Activate two skills where one allows and one denies the same tool; verify denied |
| P9.5 | Truncation behavior on guardrail hit | Trigger each guardrail limit; verify truncation indicator in output |

---

### Recommended Next Steps

1. **Address GAP-2 + PARTIAL-2: Wire up output validation.** The validation function exists but is never called. Connect it to close both the LLM output validation gap and the step output schema gap — together they mean zero output validation in the pipeline.

2. **Address GAP-1: Externalize the LLM model name.** Add a model field to the config model. One-line config addition, one-line runtime change. High impact for portability.

3. **Address PARTIAL-3: Resolve dead event type.** Either implement token streaming or remove the dead event type to avoid a false contract.

4. **Address PARTIAL-4: Add tracing to adapter modules.** Wrap key adapter operations with trace spans. Closes the observability blind spot in the persistence layer.

5. **Run behavioral verification tests.** P7.3, P9.2, and P9.5 have code-level evidence of compliance but need runtime confirmation.
