---
name: agentic-audit
description: Portable agentic architecture audit. Assesses any agentic application against behavioral principles across 9 domains using trace-based evidence collection. Produces a structured gap register report. Framework-agnostic, repository-agnostic, behavior-first.
disable-model-invocation: true
---

# /agentic-audit — Agentic Architecture Audit

Audit the current project against agentic architecture principles. Produce a structured gap register report that identifies architectural deviations, stability risks, and safety constraints. The report is the input to remediation planning.

This skill is portable — it assesses any agentic application regardless of framework, language, or stack. It reasons from behavioral invariants, not implementation patterns.

---

## Framework References

This skill is backed by a structured reference library. Load references on demand during assessment — do not read all reference files upfront.

```
references/
├── framework/          # Assessment methodology
│   ├── principle-template.md
│   ├── report-schema.md
│   ├── evidence-model.md
│   ├── verdicts.md
│   ├── system-classification.md
│   ├── sampling-rules.md
│   ├── stop-conditions.md
│   ├── severity-confidence-model.md
│   └── glossary.md
├── domains/            # 9 assessment domains with principles
│   ├── execution-boundaries/
│   ├── configuration/
│   ├── observability/
│   ├── integration-tool-model/
│   ├── schema-contract-discipline/
│   ├── error-handling/
│   ├── orchestration-model/
│   ├── scalability-extensibility/
│   └── autonomous-agent-safety/
└── examples/           # Calibration examples
    ├── sample-gap-register.md
    └── example-assessment-snippets.md
```

---

## Execution Instructions

When this skill is invoked, execute three phases without interruption.

### Phase 1 — Reconnaissance and Classification

**Read architecture docs and project structure first. Classify the system before reading source code.**

1. Read top-level documentation: `README.md`, `docs/`, any `*spec*`, `*design*`, `*architecture*` files
2. Read the directory tree and module structure (entry points, package manifests, config files)
3. Read configuration files at the project root
4. Read the entry point(s): `main.py`, `app.py`, `server.py`, `index.ts`, or equivalent

After these reads, produce a **system classification** using the dimensions from `references/framework/system-classification.md`:

- Interaction model: conversational / workflow-driven / hybrid
- Tool capability: tool-enabled / non-tool
- Approval model: approval-gated / ungated
- Autonomy level: autonomous / constrained / hybrid
- Agent cardinality: single-agent / multi-agent
- Execution model: synchronous / asynchronous / both
- Interface type: API-only / UI-backed / CLI / notebook / library / both
- Streaming: streaming / non-streaming

Then identify:
- Which domains are applicable, partially applicable, or N/A
- The 3-5 highest-risk areas to investigate
- Approximate system scale

**Do not begin domain assessment until classification is complete.**

If the project is small (single file or under ~500 lines, no infrastructure), apply **compact mode** automatically.

### Phase 2 — Targeted Evidence Collection

Based on the classification and risk hypothesis, collect evidence for each applicable domain. Use trace-based investigation:

| Priority | Evidence Target |
|----------|----------------|
| 1 | Central execution paths — agent loop, runtime entry, dispatch |
| 2 | Error handling paths — what happens on failure |
| 3 | Schema boundaries — where data crosses module lines |
| 4 | Tool invocation paths — how external systems are accessed |
| 5 | Configuration and startup — how the system initializes |
| 6 | Observability infrastructure — logging, tracing, events |
| 7 | Extension mechanisms — how new capabilities are added |
| 8 | Safety mechanisms — tool control, guardrails, approval gates |

For each domain, read the `overview.md` from the relevant `references/domains/` directory to guide evidence collection. Read individual principle files as needed for detailed verdict guidance.

**Evidence rules:**
- Follow execution traces, not file patterns
- One representative success path and one failure path per domain
- Stop when evidence is sufficient for a confident verdict (see `references/framework/stop-conditions.md`)
- Documentation is intent evidence only — it cannot justify PASS

### Phase 3 — Assessment and Report

Assess each applicable principle using the verdict definitions from `references/framework/verdicts.md`:

| Verdict | Meaning |
|---------|---------|
| **PASS** | Clearly satisfied by observed evidence |
| **GAP** | Clearly violated by observed evidence |
| **PARTIAL** | Partially satisfied — something in place but incomplete |
| **UNASSESSABLE** | Relevant but insufficient evidence to evaluate |
| **N/A** | Architecturally irrelevant for this system type |

Write the report following the structure in `references/framework/report-schema.md`.

**Report output:** `docs/assessments/{project-name}-audit-{YYYY-MM-DD}.md`

---

## Assessment Domains

| Domain | Principles | Governs |
|--------|-----------|---------|
| Execution Boundaries | P1.1–P1.4 | Who can initiate, drive, and complete runs |
| Configuration | P2.1–P2.4 | How runtime behavior is parameterized |
| Observability | P3.1–P3.6 | How execution is traced, logged, and monitored |
| Integration and Tool Model | P4.1–P4.5 | How external systems are accessed |
| Schema and Contract Discipline | P5.1–P5.5 | How data shapes are defined and enforced |
| Error Handling | P6.1–P6.5 | How failures are classified, propagated, surfaced |
| Orchestration Model | P7.1–P7.5 | How execution is sequenced and bounded |
| Scalability and Extensibility | P8.1–P8.5 | How new capabilities are added |
| Autonomous Agent Safety | P9.1–P9.5 | How agent authority is constrained |

---

## Reasoning Rules

- **Evidence-grounded only.** Every finding must cite specific evidence. Do not infer gaps from the absence of familiar patterns without noting the inference explicitly.
- **Principles over implementation.** Assess conformance to the behavioral invariant, not to a specific technology. A function registry satisfies P4.2 as well as an MCP server if it meets the underlying principle. Recognize all equivalent tool patterns.
- **N/A is not UNASSESSABLE.** N/A means architecturally irrelevant. UNASSESSABLE means relevant but evidence is insufficient. State the reason for each.
- **UNASSESSABLE is a valid verdict.** Do not stretch thin evidence into a finding.
- **Partial credit is real.** Acknowledge what IS in place before noting what is missing.
- **Severity is about production impact.** Rate by what actually breaks, not by aesthetic preference.
- **Flag behavioral verification needs.** Some principles cannot be fully assessed statically. Note this explicitly and suggest a test case.
- **Compound interactions matter.** Note when a MEDIUM gap amplifies a HIGH gap.
- **Compact mode for small projects.** Single file or under ~500 lines: collapsed format per `references/framework/report-schema.md`.
- **Do not pad the report.** If a principle passes cleanly, one line is sufficient.
- **Trace first, not file first.** Follow execution paths rather than scanning for file patterns.
- **Never PASS on documentation alone.** Documentation is intent evidence — weakest tier.
