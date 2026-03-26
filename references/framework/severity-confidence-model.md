# Severity and Confidence Model

This document defines how severity and confidence are assigned to findings in the gap register.

---

## Severity Levels

Severity reflects the production impact of a gap or partial finding. Rate by what actually breaks or degrades — not by aesthetic preference.

### CRITICAL

The gap creates an immediate safety, correctness, or data integrity risk in production.

Examples:
- Failures silently swallowed, leaving runs in indeterminate states
- No trace identifier propagated — production debugging is impossible under concurrency
- Unbounded autonomous execution with no guardrails

### HIGH

The gap creates significant instability, operability, or architectural risk.

Examples:
- Hardcoded configuration values that break portability or require code changes to modify
- Execution logic scattered across modules — changes to execution semantics require coordinated edits
- Missing input validation at system boundaries — malformed input reaches the runtime
- Unvalidated LLM output flowing to downstream systems
- Missing error classification — all failures look the same to operators

### MEDIUM

The gap degrades operability, debuggability, or extensibility but does not create immediate instability.

Examples:
- Uneven observability coverage — some modules are traced, others are dark
- Transport-layer DTOs used as domain models — coupling that resists change
- Inline system prompts instead of centralized, versioned behavioral definitions
- No reconnection support for event streams
- Guardrail hits treated as failures instead of expected truncation

### LOW

The gap represents a minor concern that does not affect production stability or operability at current scale.

Examples:
- Retry logic absent but not causing instability at current scale
- Minor terminology inconsistency in error messages
- Optional dependencies not explicitly distinguished from required ones at startup

---

## Severity Considerations

When assigning severity, consider these dimensions:

| Dimension | Question |
|-----------|----------|
| Safety risk | Could this cause the agent to take unauthorized or harmful actions? |
| Instability risk | Could this cause crashes, data loss, or unpredictable behavior? |
| Operability degradation | Does this make production debugging or monitoring significantly harder? |
| Architectural drift | Does this create coupling or boundary violations that amplify future change cost? |
| Scale constraint | Does this create a ceiling that blocks scaling the system? |
| Change amplification | Does this mean a simple change requires coordinated edits across multiple modules? |

If a gap scores high on safety or instability, it is CRITICAL. If it scores high on operability or architectural drift, it is HIGH. If it scores on debuggability or extensibility, it is MEDIUM. If it has no current impact but may become relevant, it is LOW.

---

## Confidence Levels

Confidence reflects the certainty of the evidence supporting the verdict. It is independent of severity.

### High

The evidence is strong, consistent, and from multiple observation points. The assessor has high certainty in the verdict.

- Multiple code paths inspected and consistent
- Trace followed end to end across boundaries
- Runtime behavior confirmed (if applicable)

### Medium

The evidence supports the verdict but has gaps or is limited to a single observation point.

- Single code path inspected
- Mechanism found but coverage not fully verified
- Static analysis only (no runtime confirmation for principles that recommend it)

### Low

The evidence is thin or indirect. The verdict is the best assessment possible but could change with more information.

- Based primarily on documentation or architectural inference
- Key code paths could not be located
- Behavior inferred from structure rather than confirmed

---

## Severity-Confidence Interaction

| Severity | Confidence | Recommended Action |
|----------|------------|-------------------|
| CRITICAL | High | Immediate remediation — block further development |
| CRITICAL | Medium | Investigate urgently — confirm and remediate |
| CRITICAL | Low | Investigate to raise confidence before acting |
| HIGH | High | Remediate in current sprint/iteration |
| HIGH | Medium | Remediate soon; investigate to confirm scope |
| HIGH | Low | Investigate — may not be as severe as it appears |
| MEDIUM | High | Plan remediation — schedule in backlog |
| MEDIUM | Medium-Low | Track — revisit when adjacent work touches the area |
| LOW | Any | Note and deprioritize — address opportunistically |

---

## Compound Severity

When findings interact, the compound risk may exceed individual severities:

- Two HIGH findings in the same execution path may compound to CRITICAL risk
- A MEDIUM finding that amplifies a HIGH finding should be noted in both entries
- Document compound interactions in the gap register using Related Principles cross-references
