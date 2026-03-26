# Example Assessment Snippets

This document provides calibration examples showing how to write different verdict justifications. Use these to maintain consistency across assessments.

---

## Good PASS Justification

### Example: P1.1 — Execution Centralization

> **Verdict:** PASS
> **Evidence:** Execution is centralized in `src/runtime/engine.py`. The `execute()` method is the sole entry point for all runs. Trace confirms: `api/routes/runs.py:45` calls `engine.execute()` — no business logic in the route handler. No other module contains execution initiation code (searched for `create_run`, `start_run`, `execute` across the full codebase).

**Why this is good:** Cites specific file and line. Confirms by tracing from API to runtime. Confirms absence by searching for alternative invocation patterns. Evidence is implementation-level with trace support.

---

## Weak PASS Justification (Avoid This)

### Example: P1.1 — Execution Centralization

> **Verdict:** PASS
> **Evidence:** The design document says execution is centralized in the runtime module.

**Why this is weak:** Based solely on documentation (declarative evidence). No implementation evidence. No trace evidence. Documentation alone cannot justify PASS.

---

## Good PARTIAL Justification

### Example: P3.6 — Observability Coverage

> **Verdict:** PARTIAL
> **Evidence Status:** confirmed
> **Finding:** The runtime module has comprehensive trace instrumentation (6 named spans covering execution, dispatch, step execution for all node types, and tool invocation). The API module is instrumented via framework auto-instrumentation. However, the adapter modules (database adapter, cache adapter) have structured logging only — no trace spans. Adapter operations are invisible in distributed traces.
> **What is in place:** Runtime and API layers are fully instrumented with named trace spans.
> **What is missing:** Adapter layer operations (database queries, cache reads/writes) have no trace spans. The gap between "runtime started" and "step completed" cannot be decomposed to identify whether latency is in the database, cache, or elsewhere.

**Why this is good:** Describes what IS in place before noting what is missing. Specific about which modules are instrumented and which are not. Explains the practical impact of the gap.

---

## Good UNASSESSABLE Justification

### Example: P4.5 — Adapter Self-Contained Authentication

> **Verdict:** UNASSESSABLE
> **Reason:** External integration connections are currently stubbed — `connect()` registers servers but does not perform real HTTP connections or authentication. The adapter directory defines a client structure but real credential management is not exercised. Cannot assess whether adapters manage their own credentials or whether credentials flow through the runtime until production integrations are implemented.
> **Evidence needed:** A working external integration with real API credentials — specifically, check where the API key is read and whether it is contained within the adapter or passed through the runtime's configuration.

**Why this is good:** Explains why assessment is not possible (stubbed implementation). Specifies exactly what evidence would resolve it. Does not stretch thin evidence into a finding.

---

## Correct Use of N/A

### Example: P1.2 — Transport/Runtime Separation (CLI Tool)

> **Verdict:** N/A
> **Justification:** This system is a CLI tool with no HTTP layer. There is no API, no route handlers, and no transport boundary. The CLI entry point directly invokes the agent loop. Transport/runtime separation is architecturally irrelevant.

**Why this is correct:** References a concrete system characteristic (no HTTP layer). Does not confuse N/A with UNASSESSABLE — the principle is genuinely irrelevant, not merely hard to assess.

### Counter-Example: Incorrect N/A

> **Verdict:** N/A
> **Justification:** We couldn't find evidence of transport/runtime separation.

**Why this is wrong:** The assessor couldn't find evidence — that is UNASSESSABLE or GAP, not N/A. N/A requires the principle to be architecturally irrelevant, not merely unobserved.

---

## Strong Evidence Summary

### Example: P5.5 — LLM Output Validation (GAP)

> **Evidence Summary:** Traced LLM output from the agent step through to downstream consumption. In `src/runtime/agent.py:275-287`, LLM output is parsed from JSON (or wrapped as `{"text": raw}`) and written directly to the shared state. A dedicated validation function `validate_output()` exists in `src/schemas/output.py:8` but grep for its name across the entire codebase returns only the definition — zero call sites. The function exists but is never invoked. Evidence status: **contradictory** — the system has the capability but does not use it.

**Why this is strong:** Traces the complete data flow. Cites specific lines. Identifies the validation function AND proves it is unused. Correctly classifies evidence as contradictory (capability exists but is disconnected).

---

## Weak Evidence Summary (Avoid This)

### Example: P5.5 — LLM Output Validation (GAP)

> **Evidence Summary:** LLM output doesn't appear to be validated.

**Why this is weak:** No specific file or code reference. No trace of the data flow. No distinction between "validation doesn't exist" and "validation exists but isn't used." Not actionable for remediation.

---

## Contradictory Evidence Handling

### Example: P2.1 — Externalized Configuration

> **Finding:** 43 of 44 configuration parameters are correctly externalized to `config.yaml` and validated at startup. One value — the LLM model identifier — is hardcoded at `src/runtime/agent.py:265`. Evidence status: **contradictory** — the system demonstrates strong configuration discipline everywhere except this one value, suggesting an oversight rather than a systemic issue.

**Why this is good:** Quantifies the scope (43 of 44). Identifies the specific exception. Notes the contradiction. Provides context (likely oversight vs systemic issue) to guide remediation priority.

---

## Compound Interaction Note

### Example: GAP-2 (P5.5) + PARTIAL-2 (P5.4)

> **Compound note:** GAP-2 (LLM output not validated) and PARTIAL-2 (step output schemas defined but not enforced) together mean that zero output validation exists in the execution pipeline. A malformed LLM response propagates unchecked through step output, into the shared run context, and into downstream step inputs. The compound risk exceeds either finding alone — remediating both together should be the top priority.

**Why this is good:** Identifies the interaction explicitly. Explains the compound effect. Recommends joint remediation. Helps the reader understand that these two findings are one systemic issue, not two independent ones.

---

## Behavioral Verification Flag

### Example: P7.3 — Bounded Autonomous Execution

> **Verdict:** PASS (with behavioral verification recommended)
> **Finding:** Autonomous execution is bounded by three configurable limits: `max_iterations`, `max_tool_calls`, and `max_tokens`. The limits are read from configuration and checked in the agent loop (`src/graph/autonomous.py:42-55`). Static analysis confirms the checks exist, but cannot confirm runtime enforcement — the check code could have edge cases (off-by-one, async race conditions) that only surface under execution.
> **Behavioral verification recommended:** Construct a test that deliberately exhausts each limit individually and verify: (1) the run transitions to COMPLETED (not FAILED), (2) the output includes a truncation indicator, (3) no additional iterations occur after the limit is hit.

**Why this is good:** Awards PASS based on strong static evidence. Honestly flags what static analysis cannot confirm. Provides a specific, actionable test case for runtime verification.
