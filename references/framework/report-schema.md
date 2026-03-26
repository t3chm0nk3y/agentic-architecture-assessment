# Report Schema

This document defines the structure of the gap register report produced by the assessment framework.

---

## Report Sections

The report must contain these sections in this order:

1. Header
2. Executive Summary
3. System Classification
4. Conformance Overview
5. Gap Register
6. Passed Principles
7. Not Applicable
8. Behavioral Verification Register
9. Recommended Next Steps

---

## 1. Header

```markdown
# Agentic Architecture Assessment — {Project Name}
**Date:** {YYYY-MM-DD}
**Auditor:** {auditor identity}
**Evidence Sources:** {list of key files and docs reviewed}
**System Type:** {classification from system-classification.md}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| Project Name | Yes | Name of the system under assessment |
| Date | Yes | Assessment date (YYYY-MM-DD) |
| Auditor | Yes | Who performed the assessment |
| Evidence Sources | Yes | Key files, docs, and paths reviewed — not exhaustive, but representative |
| System Type | Yes | Primary classification from the system classification taxonomy |

---

## 2. Executive Summary

2-4 sentences covering:
- What kind of system this is
- Overall conformance picture (X of Y principles pass)
- The 2-3 most critical findings by severity

---

## 3. System Classification

Formal classification against the dimensions in `system-classification.md`. This section drives applicability for all principles.

```markdown
## System Classification

| Dimension | Classification |
|-----------|---------------|
| Interaction model | conversational / workflow-driven / hybrid |
| Tool capability | tool-enabled / non-tool |
| Approval model | approval-gated / ungated |
| Autonomy level | autonomous / constrained / hybrid |
| Agent cardinality | single-agent / multi-agent |
| Execution model | synchronous / asynchronous / both |
| Interface type | API-only / UI-backed / both / CLI / notebook |
| Streaming | streaming / non-streaming |
```

---

## 4. Conformance Overview

A summary table with counts per domain.

```markdown
| Domain | Assessed | Pass | Gap | Partial | N/A | Unassessable |
|--------|----------|------|-----|---------|-----|--------------|
```

- **Assessed** = total principles evaluated (excludes N/A)
- Severity columns (Critical, High, Medium, Low) count gaps and partials only
- Include a **Total** row

---

## 5. Gap Register

Organized by severity: CRITICAL, HIGH, MEDIUM, LOW, then UNASSESSABLE.

### Per-Finding Schema

Each finding must include:

| Field | Required | Description |
|-------|----------|-------------|
| `principle_id` | Yes | e.g., P2.1 |
| `title` | Yes | Principle title |
| `domain` | Yes | Domain name |
| `applicability` | Yes | Why this principle applies to this system |
| `verdict` | Yes | GAP, PARTIAL, or UNASSESSABLE |
| `evidence_status` | Yes | confirmed, partial, missing, or contradictory |
| `confidence` | Yes | high, medium, or low |
| `severity` | Yes | CRITICAL, HIGH, MEDIUM, or LOW |
| `finding` | Yes | Specific finding grounded in evidence |
| `why_it_matters` | Yes | Production impact if not addressed |
| `evidence_summary` | Yes | What was observed and where |
| `affected_components` | Yes | Modules, files, or paths affected |
| `recommended_remediation` | Yes | Actionable fix guidance |
| `verification_method` | Yes | How to confirm the fix works |
| `behavioral_verification_required` | Yes | Whether runtime testing is needed to fully assess |

### Markdown Format

```markdown
### {SEVERITY}

#### {GAP-ID}: {Principle ID} — {Title}
**Verdict:** {GAP | PARTIAL}
**Evidence Status:** {confirmed | partial | missing | contradictory}
**Confidence:** {high | medium | low}
**Finding:** {Specific finding grounded in evidence}
**Evidence:** {File paths, code patterns, or absences observed}
**Risk:** {Production impact}
**Affected Components:** {List}
**Remediation:** {Actionable guidance}
**Verification:** {How to confirm the fix} | Behavioral verification needed: {Yes/No}
```

---

## 6. Passed Principles

Brief list — one line per principle with ID and one-line confirmation of what was observed.

```markdown
- **P1.1** — Execution centralized in {component}; no other module initiates runs.
```

---

## 7. Not Applicable

List of N/A principle IDs with one-line justification referencing system classification.

```markdown
- **P1.2** — N/A: system has no HTTP layer (CLI tool).
```

---

## 8. Behavioral Verification Register

Aggregated list of principles where behavioral verification was flagged but not performed during assessment.

```markdown
| Principle | What Needs Verification | Suggested Test Case |
|-----------|------------------------|-------------------|
| P7.3 | Guardrail enforcement at runtime | Exhaust max_iterations and verify COMPLETED with truncation |
```

---

## 9. Recommended Next Steps

Ordered list of top 5 actions by severity and effort. Format:

```markdown
1. **Address {GAP-ID}: {what to do and why it matters most}**
```

---

## Compact Mode

For small projects (single file or under ~500 lines, no infrastructure, no multi-module structure):

- State at the top: `[Compact Mode — small project]`
- Consolidate passed principles to one sentence per domain
- Limit each gap entry to two lines (finding + evidence)
- Skip the severity columns in the conformance table; use a single gap count
- Omit the behavioral verification register
- Omit the system classification table (state type in header only)

---

## Report Output Location

Reports are written to:

```
docs/assessments/{project-name}-assessment-{YYYY-MM-DD}.md
```

Create the directory if it does not exist.
