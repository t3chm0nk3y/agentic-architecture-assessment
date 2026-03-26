# Agentic Architecture Assessment Framework

A portable, trace-based assessment framework for auditing agentic applications. Packaged as a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that can be installed into any project.

Assesses **44 behavioral principles** across **9 architectural domains** — framework-agnostic, language-agnostic, behavior-first.

---

## Quick Start

### Install into your project

Copy `SKILL.md` and `references/` into a Claude Code skill directory in your target project:

```bash
# From the root of your target project
mkdir -p .claude/skills/agentic-audit

# Option A: clone and copy
git clone https://github.com/t3chm0nk3y/agentic-architecture-assessment.git /tmp/aaa
cp /tmp/aaa/SKILL.md .claude/skills/agentic-audit/
cp -r /tmp/aaa/references .claude/skills/agentic-audit/
rm -rf /tmp/aaa

# Option B: if you already have the repo locally
cp /path/to/agentic-architecture-assessment/SKILL.md .claude/skills/agentic-audit/
cp -r /path/to/agentic-architecture-assessment/references .claude/skills/agentic-audit/
```

Your target project should now have:

```
your-project/
├── .claude/
│   └── skills/
│       └── agentic-audit/
│           ├── SKILL.md
│           └── references/
│               ├── index.md
│               ├── framework/    (9 methodology files)
│               ├── domains/      (9 domains, 44 principle files)
│               └── examples/     (2 calibration files)
├── src/
└── ...
```

### Run the audit

In Claude Code, from your target project:

```
/agentic-audit
```

The skill runs three phases automatically:

1. **Reconnaissance** — reads docs and project structure, classifies the system type
2. **Evidence Collection** — follows execution traces to collect evidence per domain
3. **Assessment** — applies principles, produces the structured gap register report

Output is written to `docs/assessments/{project-name}-audit-{YYYY-MM-DD}.md` in the target project.

---

## What It Assesses

| Domain | Principles | Governs |
|--------|-----------|---------|
| Execution Boundaries | P1.1 -- P1.4 | Who can initiate, drive, and complete runs |
| Configuration | P2.1 -- P2.4 | How runtime behavior is parameterized |
| Observability | P3.1 -- P3.6 | How execution is traced, logged, and monitored |
| Integration and Tool Model | P4.1 -- P4.5 | How external systems are accessed |
| Schema and Contract Discipline | P5.1 -- P5.5 | How data shapes are defined and enforced |
| Error Handling | P6.1 -- P6.5 | How failures are classified, propagated, and surfaced |
| Orchestration Model | P7.1 -- P7.5 | How execution is sequenced and bounded |
| Scalability and Extensibility | P8.1 -- P8.5 | How new capabilities are added |
| Autonomous Agent Safety | P9.1 -- P9.5 | How agent authority is constrained |

See [references/index.md](references/index.md) for the full principle index with links to each principle file.

## What It Does Not Assess

- Code quality or style
- Test coverage
- Performance benchmarks
- Security vulnerabilities (beyond agent safety principles)
- Business logic correctness

---

## Key Design Properties

- **Behavior-first** -- assesses behavioral invariants, not implementation patterns. A function registry satisfies P4.2 as well as an MCP server.
- **Framework-agnostic** -- works with any agent framework, language, or runtime.
- **Trace-based** -- follows execution paths rather than scanning for file patterns.
- **Honest verdicts** -- PASS requires strong evidence; documentation alone is insufficient.
- **Explicit applicability** -- a system classification step determines which principles are relevant before assessment begins.

## Verdicts

| Verdict | Meaning |
|---------|---------|
| **PASS** | Clearly satisfied by observed evidence |
| **GAP** | Clearly violated by observed evidence |
| **PARTIAL** | Partially satisfied -- something in place but incomplete |
| **UNASSESSABLE** | Relevant but insufficient evidence to evaluate |
| **N/A** | Architecturally irrelevant for this system type |

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
│   ├── source-synthesis.md      # Design rationale and source material analysis
│   └── framework-consistency-review.md
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
# Remove old version and copy new
rm -rf .claude/skills/agentic-audit/
mkdir -p .claude/skills/agentic-audit
cp /path/to/agentic-architecture-assessment/SKILL.md .claude/skills/agentic-audit/
cp -r /path/to/agentic-architecture-assessment/references .claude/skills/agentic-audit/
```

Principle IDs (P1.1, P2.1, etc.) and the report format are stable across versions. Existing audit reports remain valid references after upgrading.
