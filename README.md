# Agentic Architecture Assessment

A trace-based assessment skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that evaluates whether an agentic application has been implemented according to critical best practices and whether its architecture is correctly enforced in runtime behavior.

Assesses **44 behavioral principles** across **9 architectural domains** -- framework-agnostic, language-agnostic, behavior-first.

This is not a design review. The assessment follows execution traces through source code to evaluate whether critical implementation practices are actually in place and enforced -- not merely documented or intended. Documentation alone cannot justify a passing verdict.

---

## Quick Start

### Install into your project

Copy `SKILL.md` and `references/` into a Claude Code skill directory in your target project:

```bash
# From the root of your target project
mkdir -p .claude/skills/agentic-architecture-assessment

# Option A: clone and copy
git clone https://github.com/t3chm0nk3y/agentic-architecture-assessment.git /tmp/aaa
cp /tmp/aaa/SKILL.md .claude/skills/agentic-architecture-assessment/
cp -r /tmp/aaa/references .claude/skills/agentic-architecture-assessment/
rm -rf /tmp/aaa

# Option B: if you already have the repo locally
cp /path/to/agentic-architecture-assessment/SKILL.md .claude/skills/agentic-architecture-assessment/
cp -r /path/to/agentic-architecture-assessment/references .claude/skills/agentic-architecture-assessment/
```

Your target project should now have:

```
your-project/
├── .claude/
│   └── skills/
│       └── agentic-architecture-assessment/
│           ├── SKILL.md
│           └── references/
│               ├── index.md
│               ├── framework/    (9 methodology files)
│               ├── domains/      (9 domains, 44 principle files)
│               └── examples/     (2 calibration files)
├── src/
└── ...
```

### Run the assessment

In Claude Code, from your target project:

```
/agentic-architecture-assessment
```

The skill runs three phases automatically:

1. **Reconnaissance** -- reads docs and project structure, classifies the system type
2. **Evidence Collection** -- follows execution traces to collect evidence per domain
3. **Assessment** -- applies principles, produces the structured gap register report

### Output

Reports are written to:

```
docs/assessments/{project-name}-assessment-{YYYY-MM-DD}.md
```

The output directory is created automatically if it does not exist.

---

## Naming and Invocation

These identifiers are intentionally aligned to prevent naming drift:

| Aspect | Value |
|--------|-------|
| Repository | `agentic-architecture-assessment` |
| Skill folder | `.claude/skills/agentic-architecture-assessment/` |
| Slash command | `/agentic-architecture-assessment` |

No short alias is defined. All identifiers match the repository name.

---

## Critical Best Practices Assessed

The framework evaluates whether an agentic application correctly implements and enforces the practices that matter most for correctness, control, observability, safe autonomy, and extensibility.

Each practice below maps to one or more of the 44 principles assessed:

| Practice | Why It Matters |
|----------|---------------|
| **Centralized execution control** | A single execution authority prevents scattered, uncoordinated agent runs and makes the system auditable. |
| **Explicit orchestration model** | Deliberate orchestration with explicit state transitions prevents implicit, untraceable execution flows. |
| **Separation of control plane and execution** | Transport layers that delegate to runtime -- rather than owning business logic -- keep execution semantics in one place. |
| **Tool mediation and validation** | All external interactions through a single abstraction layer prevents uncontrolled side effects and enables instrumentation. |
| **Schema and contract discipline** | Canonical data contracts defined once and validated at boundaries prevent data corruption across module lines. |
| **LLM output validation** | Unvalidated LLM output flowing into downstream systems is a top failure mode -- validation must be enforced, not optional. |
| **End-to-end observability** | Structured logging, trace propagation, and lifecycle events across all layers make failures diagnosable. |
| **Error propagation with context** | Failures must propagate with classification, context, and trace identity -- silent swallowing is a critical gap. |
| **Configuration-driven behavior** | Externalized, validated configuration prevents hardcoded values and ensures startup safety. |
| **Bounded execution and termination control** | Autonomous agent loops must have explicit guardrails -- unbounded execution is a safety and cost risk. |
| **Safe autonomy and guardrails** | Tool access control, approval gates, and restriction enforcement under composition constrain agent authority. |
| **Truncation as completion, not failure** | Guardrail hits must produce controlled completion with truncation indicators, not crash the system. |
| **Extensibility without core modification** | New workflows, tools, and skills must be addable without modifying the runtime or execution engine. |
| **Identity and state propagation** | Every execution must carry a unique identifier propagated across all components for traceability. |
| **Behavioral verification capability** | Some practices cannot be fully assessed statically -- the system must support runtime verification of critical constraints. |

---

## Assessment Domains

| Domain | Principles | Governs |
|--------|-----------|---------|
| Execution Boundaries | P1.1--P1.4 | Who can initiate, drive, and complete runs |
| Configuration | P2.1--P2.4 | How runtime behavior is parameterized |
| Observability | P3.1--P3.6 | How execution is traced, logged, and monitored |
| Integration and Tool Model | P4.1--P4.5 | How external systems are accessed |
| Schema and Contract Discipline | P5.1--P5.5 | How data shapes are defined and enforced |
| Error Handling | P6.1--P6.5 | How failures are classified, propagated, and surfaced |
| Orchestration Model | P7.1--P7.5 | How execution is sequenced and bounded |
| Scalability and Extensibility | P8.1--P8.5 | How new capabilities are added |
| Autonomous Agent Safety | P9.1--P9.5 | How agent authority is constrained |

See [references/index.md](references/index.md) for the full principle index with links to each principle file.

## What It Does Not Assess

This framework does not evaluate:

- Code style or aesthetic preferences
- Test coverage metrics
- Performance benchmarks
- Security vulnerabilities (beyond agent safety principles)
- Business logic correctness
- Framework or language preference
- UI/UX design quality
- Design intent without implementation evidence

---

## Key Design Properties

- **Behavior-first** -- assesses behavioral invariants, not implementation patterns. A function registry satisfies P4.2 as well as an MCP server.
- **Framework-agnostic** -- works with any agent framework, language, or runtime.
- **Trace-based** -- follows execution paths rather than scanning for file patterns.
- **Honest verdicts** -- PASS requires strong implementation evidence; documentation alone is insufficient.
- **Explicit applicability** -- a system classification step determines which principles are relevant before assessment begins.

## Verdicts

| Verdict | Meaning |
|---------|---------|
| **PASS** | Clearly satisfied by observed implementation evidence |
| **GAP** | Clearly violated by observed evidence |
| **PARTIAL** | Partially satisfied -- something in place but incomplete |
| **UNASSESSABLE** | Relevant but insufficient evidence to evaluate |
| **N/A** | Architecturally irrelevant for this system type |

### Behavioral verification

Some principles cannot be fully assessed through static code analysis alone. When a principle requires runtime confirmation (e.g., guardrail enforcement under real execution), the assessment flags it in the **Behavioral Verification Register** with a suggested test case. Three principles are pre-identified as requiring behavioral verification: P7.3, P9.2, and P9.5.

---

## Repo Structure

```
.
├── SKILL.md                     # The Claude Code skill (copy this to target projects)
├── references/                  # Framework reference library (copy this too)
│   ├── index.md                 # Master index of all principles
│   ├── framework/               # Assessment methodology
│   │   ├── principle-template.md
│   │   ├── report-schema.md
│   │   ├── evidence-model.md
│   │   ├── verdicts.md
│   │   ├── system-classification.md
│   │   ├── sampling-rules.md
│   │   ├── stop-conditions.md
│   │   ├── severity-confidence-model.md
│   │   └── glossary.md
│   ├── domains/                 # 9 assessment domains
│   │   ├── execution-boundaries/     (overview + 4 principles)
│   │   ├── configuration/            (overview + 4 principles)
│   │   ├── observability/            (overview + 6 principles)
│   │   ├── integration-tool-model/   (overview + 5 principles)
│   │   ├── schema-contract-discipline/ (overview + 5 principles)
│   │   ├── error-handling/           (overview + 5 principles)
│   │   ├── orchestration-model/      (overview + 5 principles)
│   │   ├── scalability-extensibility/ (overview + 5 principles)
│   │   └── autonomous-agent-safety/  (overview + 5 principles)
│   └── examples/                # Calibration examples
│       ├── sample-gap-register.md
│       └── example-assessment-snippets.md
├── docs/                        # Development documentation
│   ├── source-synthesis.md
│   ├── framework-consistency-review.md
│   ├── repository-consistency-review.md
│   └── repository-improvements-summary.md
└── README.md
```

---

## Extending the Framework

### Add a principle to an existing domain

1. Create a new file in `references/domains/{domain}/` following `references/framework/principle-template.md`
2. Update the domain's `overview.md` to include the new principle
3. Update `references/index.md`

### Add a new domain

1. Create a new directory under `references/domains/`
2. Add an `overview.md` and principle files
3. Update `SKILL.md` to reference the new domain in the Assessment Domains table
4. Update `references/index.md`

---

## Upgrading in Target Projects

To update a target project to the latest framework version:

```bash
# Replace the entire skill folder with the newer version
rm -rf .claude/skills/agentic-architecture-assessment/
mkdir -p .claude/skills/agentic-architecture-assessment
cp /path/to/agentic-architecture-assessment/SKILL.md .claude/skills/agentic-architecture-assessment/
cp -r /path/to/agentic-architecture-assessment/references .claude/skills/agentic-architecture-assessment/
```

- Replace the entire skill folder -- do not merge files manually unless intentionally customizing.
- Assessment reports in the target project's `docs/assessments/` are independent and not affected by upgrades.
- Principle IDs (P1.1, P2.1, etc.) and the report format are stable across versions.

---

## Versioning

This framework does not yet use semantic versioning. Principle IDs and the report schema are treated as stable. Breaking changes to principle definitions or report structure will be documented in release notes when formal versioning is adopted.
