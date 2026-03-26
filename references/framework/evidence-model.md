# Evidence Model

This document defines the evidence hierarchy used throughout the assessment framework. Evidence type determines the weight an observation carries toward a verdict.

---

## Evidence Hierarchy

Evidence types are ranked from strongest to weakest. Higher-ranked evidence overrides lower-ranked evidence when they conflict.

### 1. Runtime Evidence (Strongest)

Evidence collected from a running system: execution logs, trace output, runtime behavior under test, observed API responses, event stream content.

- Directly confirms or denies behavioral invariants
- Required for principles flagged with "Behavioral Verification Required"
- Not always available during static assessment — its absence does not automatically lower a verdict

### 2. End-to-End Trace Evidence

Evidence collected by tracing an execution path from entry to exit across module boundaries: following a request from API endpoint through dispatch to runtime to tool invocation to persistence.

- Confirms that enforcement mechanisms are connected and functioning across boundaries
- Stronger than inspecting a single module in isolation
- The primary evidence type for most principle assessments

### 3. Local Implementation Evidence

Evidence collected from inspecting code within a single module or component: reading a function body, checking a class structure, inspecting an error handler.

- Confirms that a mechanism exists at a specific location
- Cannot alone confirm that the mechanism is connected to the broader system
- Sufficient for PASS when the mechanism is self-contained (e.g., a config validation model)

### 4. Declarative Intent Evidence (Weakest)

Evidence from documentation, specifications, comments, design docs, README files, ADRs, or inline comments.

- Demonstrates intent, not implementation
- Cannot justify PASS on its own for any principle
- Useful for understanding design goals and identifying discrepancies between intent and implementation
- Contradictions between declarative evidence and implementation evidence always favor the implementation

---

## Evidence Status Values

Every finding in the gap register must include an `evidence_status` field with one of these values:

### confirmed

Strong evidence supporting the verdict. Implementation evidence or better, with consistent findings across multiple observation points.

### partial

Some evidence exists but it is incomplete or inconsistent. Evidence may be limited to a single module, a single code path, or a single execution scenario.

### missing

The assessor looked for evidence in the expected locations and did not find it. The absence is the finding.

### contradictory

Evidence from different sources or different parts of the system conflicts. For example: documentation claims a behavior exists, but implementation evidence shows it does not. Or: the mechanism exists in one module but is absent in another where it should also apply.

Contradictory evidence is more severe than missing evidence because it indicates either:
- Active divergence between intent and implementation
- Partial implementation that creates a false sense of security

---

## Evidence Collection Rules

### What to Document

For each principle assessed, record:
- What files/paths/traces were inspected
- What was found (or not found)
- Which evidence type each observation represents
- Whether the evidence is consistent across observation points

### When to Stop Collecting

See `stop-conditions.md` for detailed rules. In general:
- Stop when you have sufficient evidence for a confident verdict
- Stop when further reading would be repetitive
- Do not exhaustively read every file — sample representative paths

### How Evidence Maps to Verdicts

| Verdict | Required Evidence |
|---------|------------------|
| PASS | At least implementation evidence showing all required behaviors. Trace evidence preferred. |
| PARTIAL | Implementation evidence showing some but not all required behaviors. |
| GAP | Implementation evidence or trace evidence showing the required behavior is absent or violated. |
| UNASSESSABLE | Cannot collect sufficient evidence of any type — state what is needed. |

### Documentation Cannot Justify PASS

This rule is absolute. If the only evidence for a principle is documentation (specs, comments, READMEs), the maximum verdict is PARTIAL — and only if the documentation is specific and credible enough to suggest partial implementation.

In practice, documentation evidence serves two purposes:
1. It guides where to look for implementation evidence
2. It establishes contradictory evidence when implementation diverges from claims
