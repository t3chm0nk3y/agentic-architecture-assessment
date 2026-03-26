# Stop Conditions

This document defines when evidence collection should stop for a given principle or domain. The framework values efficient, targeted assessment over exhaustive reading.

---

## Per-Principle Stop Conditions

Stop collecting evidence for a principle when any of these conditions is met:

### 1. Confident Verdict Reached

The collected evidence is sufficient for a high-confidence verdict. Additional reading would not change the verdict.

- PASS: All required behaviors confirmed through implementation or trace evidence
- GAP: Clear absence or violation confirmed — further reading would be redundant
- N/A: System classification makes the principle irrelevant

### 2. Representative Path Traced End to End

The enforcement point has been found, and downstream propagation has been inspected. The path is representative of the system's typical behavior.

### 3. Stronger Evidence Unavailable

The assessor has looked in the expected locations and the best available evidence has been found. Stronger evidence (e.g., runtime traces) is not available during this assessment.

### 4. Repetitive Evidence

Further reading is producing the same evidence pattern. For example: if three modules all handle errors the same way, inspecting a fourth is unlikely to change the verdict.

---

## Per-Domain Stop Conditions

Stop evidence collection for a domain when:

- All principles in the domain have a verdict (PASS, GAP, PARTIAL, UNASSESSABLE, or N/A)
- The domain overview accurately reflects the findings
- Cross-domain interactions have been noted

---

## Assessment-Level Stop Conditions

The overall assessment is complete when:

- All applicable domains have been assessed
- All applicable principles have a verdict
- The gap register is populated
- The conformance overview is accurate
- The behavioral verification register identifies principles needing runtime verification
- Compound interactions between findings have been noted

---

## When NOT to Stop

Do not stop early when:

- A finding is borderline and one more code path could resolve it
- An unexpected finding suggests a systemic issue that may affect adjacent principles
- The system hypothesis from Phase 1 identified high-risk areas that have not yet been assessed
- Contradictory evidence has been found — this requires investigation, not early termination

---

## Time-Bounded Assessment

If the assessment is time-constrained:

1. Assess domains in priority order (use sampling-rules.md priority list)
2. For each domain, assess principles in severity order (CRITICAL first)
3. Use UNASSESSABLE for principles where time ran out before evidence could be collected
4. Note the time constraint in the report header
5. Never compromise verdict accuracy for speed — UNASSESSABLE is always preferred over a guessed verdict
