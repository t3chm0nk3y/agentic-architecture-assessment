# Verdict Definitions

This document defines the five verdicts used across all principle assessments. Verdict definitions are stable and must be applied consistently.

---

## Verdicts

### PASS

The principle is clearly satisfied by observed evidence.

- Requires strong evidence: runtime traces, end-to-end code paths, or confirmed implementation patterns
- Documentation or design claims alone are insufficient for PASS
- All required behaviors listed in the principle must be satisfied
- If behavioral verification is flagged but not performed, the assessor may still award PASS if static evidence is conclusive — but must note the verification gap

### PARTIAL

The principle is partially satisfied — something is in place but incomplete.

- At least one required behavior is satisfied, but not all
- OR the mechanism exists but has gaps in coverage (e.g., observability in some modules but not others)
- OR the mechanism exists but is not consistently enforced
- The assessor must describe what IS in place before noting what is missing
- PARTIAL is not a hedge for weak evidence — that is UNASSESSABLE

### GAP

The principle is clearly violated by observed evidence.

- Required behaviors are absent or contradicted
- OR evidence demonstrates the opposite of the required behavior
- Must be grounded in specific observed evidence, not absence of a preferred pattern
- Absence of a familiar pattern is only a GAP if the required behavior is genuinely missing — not if it is satisfied through an unfamiliar approach
- Contradictory evidence (mechanism exists but demonstrably does not work) is more severe than simple absence

### UNASSESSABLE

The principle is relevant to this system, but the available evidence is insufficient to determine a verdict.

- The principle applies (it is not N/A) but cannot be evaluated with the available evidence
- Must specify exactly what evidence would resolve the assessment
- Common causes: stubbed implementations, untestable without runtime access, insufficient code coverage of the relevant path
- UNASSESSABLE is a valid terminal verdict — do not stretch thin evidence into a finding
- UNASSESSABLE is never a substitute for N/A

### N/A (Not Applicable)

The principle is architecturally irrelevant for this system type.

- The system's classification makes this principle meaningless
- Examples: P1.2 (transport/runtime separation) is N/A for a CLI tool with no HTTP layer; P3.4 (event delivery reliability) is N/A for a system with no streaming interface
- N/A must be justified with a specific system characteristic, not assessor preference
- N/A principles are excluded from conformance counts

---

## Decision Tree

When assessing a principle, follow this sequence:

```
1. Is the principle applicable to this system type?
   └─ No  → N/A (justify with system classification)
   └─ Yes → continue

2. Is there sufficient evidence to evaluate the principle?
   └─ No  → UNASSESSABLE (specify what evidence is needed)
   └─ Yes → continue

3. Are ALL required behaviors satisfied by strong evidence?
   └─ Yes → PASS
   └─ No  → continue

4. Are SOME required behaviors satisfied?
   └─ Yes → PARTIAL (describe what is and is not in place)
   └─ No  → GAP (cite specific evidence of violation or absence)
```

---

## Compound Verdict Rules

- When a PARTIAL in one principle amplifies a GAP in another, note the interaction in both findings
- When two related principles are both GAP, consider whether the compound risk warrants severity escalation in the report
- When a principle is PASS but behavioral verification was not performed, note the verification gap in the evidence summary — it does not change the verdict but informs remediation priority

---

## Evidence Strength Requirements by Verdict

| Verdict | Minimum Evidence |
|---------|-----------------|
| PASS | Implementation evidence showing all required behaviors, confirmed by trace or runtime evidence |
| PARTIAL | Implementation evidence showing some required behaviors |
| GAP | Implementation evidence or trace evidence showing absence or violation |
| UNASSESSABLE | Identification of what evidence is needed |
| N/A | System classification characteristic that makes the principle irrelevant |

See `evidence-model.md` for the full evidence hierarchy.
