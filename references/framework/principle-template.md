# Principle Template

Every principle file in the framework must follow this structure exactly. Fields marked **(required)** must be present in every principle file. Fields marked **(conditional)** may be omitted with justification.

---

## Frontmatter

```yaml
---
id: P{domain}.{number}          # e.g., P3.1
domain: {domain-name}           # e.g., observability
title: {short title}            # e.g., Run Traceability
severity: CRITICAL | HIGH | MEDIUM | LOW
last_updated: YYYY-MM-DD
---
```

---

## Field Definitions

### 1. Principle ID **(required)**

Format: `P{domain_number}.{principle_number}` (e.g., P3.1). Stable across versions. Never renumbered.

### 2. Domain **(required)**

The domain this principle belongs to. Must match a directory name in `references/domains/`.

### 3. Title **(required)**

A short, descriptive title. Imperative or declarative form. Must be understandable without reading the full principle.

### 4. Intent **(required)**

A single paragraph stating the behavioral invariant this principle requires. Must use "must" language. Must describe what is required, not how to implement it.

Rules:
- State the invariant, not a preference
- Use technology-neutral language
- Do not name specific frameworks, libraries, or tools
- Do not prescribe folder structures or file names

### 5. Why It Matters **(required)**

A short paragraph explaining the production impact of violating this principle. Focus on what breaks, degrades, or becomes unsafe — not on aesthetic or theoretical concerns.

### 6. Applicability **(required)**

Which system classifications (from `system-classification.md`) this principle applies to. Use the classification dimensions explicitly.

Format:
```
Applies to: [list of system types or "all system types"]
Requires: [minimum architectural features needed, e.g., "HTTP API layer", "persistent run state"]
```

### 7. N/A Conditions **(required)**

Explicit conditions under which this principle is not applicable. Must reference system classification dimensions. If no N/A conditions exist, state "None — this principle applies to all agentic systems."

Rules:
- N/A means architecturally irrelevant, not "hard to assess"
- N/A is never a substitute for UNASSESSABLE
- Each condition must be objectively determinable from the system under assessment

### 8. Required Behaviors **(required)**

A numbered list of testable behavioral invariants that the system must satisfy for this principle. Each behavior must be:
- Observable (can be confirmed by reading code, traces, or runtime output)
- Implementation-neutral (multiple valid approaches can satisfy it)
- Binary or near-binary (either the behavior exists or it does not)

### 9. Acceptable Implementation Patterns **(required)**

Technology-neutral examples of how conforming systems may satisfy the required behaviors. Include at least 2 distinct patterns. Purpose: prevent assessors from penalizing unfamiliar but valid approaches.

Rules:
- Never present a single pattern as the only valid approach
- Include patterns from different technology ecosystems
- Do not name this repository's specific implementations

### 10. Evidence to Seek **(required)**

Structured list of what the assessor should look for, organized by evidence type (from `evidence-model.md`):
- **Runtime evidence:** What to check in running systems, logs, traces
- **Trace evidence:** What execution paths to follow end-to-end
- **Implementation evidence:** What code patterns to inspect
- **Declarative evidence:** What documentation or specs claim (lowest weight)

### 11. Trace Method **(required)**

Step-by-step investigation procedure for an assessor working in an unfamiliar codebase. Must be trace-first: follow execution paths, not scan for file patterns.

Format: Numbered steps, each describing one investigation action. Steps should be ordered from highest-value evidence to lowest.

### 12. Verdict Conditions **(required)**

Explicit rubric for each verdict. Must be unambiguous enough that two assessors would reach the same verdict given the same evidence.

```
**PASS when:** [specific conditions that must all be true]
**PARTIAL when:** [specific conditions — something is in place but incomplete]
**GAP when:** [specific conditions — the principle is clearly violated]
**UNASSESSABLE when:** [specific conditions — relevant but insufficient evidence]
```

Rules:
- PASS requires strong evidence, not just documentation claims
- PARTIAL must describe what IS in place, not just what is missing
- GAP must be grounded in observed evidence, not absence of familiar patterns
- UNASSESSABLE must specify what evidence would resolve the assessment

### 13. Behavioral Verification Required? **(required)**

Yes or No. If Yes, describe what can only be verified by runtime testing and suggest a specific test case.

Format:
```
**Required:** Yes | No
**What cannot be verified statically:** [description]
**Suggested test case:** [specific test scenario]
```

### 14. Suggested Runtime Verification **(conditional)**

Present only when Behavioral Verification Required is Yes. Describes a concrete test scenario that would confirm or deny the principle at runtime.

### 15. Related Principles **(required)**

Cross-references to other principles with a brief explanation of the relationship. Use principle IDs and relative file paths.

Format:
```
- [P{x}.{y}](relative-path.md) — [relationship description]
```

Relationships to document:
- Amplifies: this principle's gap makes another principle's gap worse
- Depends on: this principle cannot be satisfied without the other
- Overlaps with: there is a shared concern — document the boundary

### 16. Boundary Notes **(conditional)**

Edge cases, judgment calls, common assessment mistakes, and clarifications about where this principle's scope ends and another begins. Include only when non-obvious.

---

## Formatting Rules

- Use Markdown throughout
- Frontmatter is YAML between `---` delimiters
- Section headers use `##` (h2)
- Lists use `-` for unordered, numbers for ordered
- Code references use backticks
- File references use relative paths from the principle file's location
- No emojis
- No HTML
- Line length: soft wrap at 120 characters
