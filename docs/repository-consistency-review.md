# Repository Consistency Review

**Date:** 2026-03-26
**Purpose:** Document the naming alignment performed across the repository and establish the canonical identity to prevent future drift.

---

## Naming Drift Before This Update

The repository name and the runtime skill identity were misaligned:

| Aspect | Before | Problem |
|--------|--------|---------|
| Repository | `agentic-architecture-assessment` | -- |
| Skill name (frontmatter) | `agentic-audit` | Did not match repo name |
| Install folder | `.claude/skills/agentic-audit/` | Did not match repo name |
| Slash command | `/agentic-audit` | Did not match repo name |

The mismatch meant users encountered three different names for the same artifact: the repository name when cloning, the folder name when installing, and the command name when invoking.

---

## Canonical Identity Adopted

| Aspect | Value |
|--------|-------|
| Repository | `agentic-architecture-assessment` |
| Skill name (SKILL.md frontmatter) | `agentic-architecture-assessment` |
| Install folder | `.claude/skills/agentic-architecture-assessment/` |
| Slash command | `/agentic-architecture-assessment` |

No short alias is defined. All identifiers are aligned to the repository name.

---

## Files Updated

| File | What Changed |
|------|-------------|
| `SKILL.md` | Frontmatter `name` field, heading, description, and positioning language |
| `README.md` | Full rewrite: installation paths, invocation command, positioning language, added Naming and Invocation section, added Versioning section, added behavioral verification note |
| `docs/framework-consistency-review.md` | Two references to `agentic-audit-skill` updated |
| `docs/repository-consistency-review.md` | Created (this document) |

### Files Reviewed With No Changes Needed

| File | Reason |
|------|--------|
| `references/index.md` | No skill naming references |
| `references/examples/sample-gap-register.md` | No skill naming references |
| `references/examples/example-assessment-snippets.md` | No skill naming references |
| `references/framework/report-schema.md` | Uses "Agentic Architecture Audit" as report title (correct) |
| `docs/source-synthesis.md` | References `.claude/skills/audit/SKILL.md` in source document table (historical reference to the original monolithic skill, not to this repository's identity) |
| All 44 principle files | No skill naming references |
| All 9 domain overview files | No skill naming references |
| All 9 framework methodology files | No skill naming references |

---

## Positioning Strengthened

The README and SKILL.md descriptions were updated to make explicit that the framework assesses:

- Architecture (module boundaries, separation of concerns)
- Implementation enforcement (whether constraints are checked and violations rejected)
- Runtime behavior (execution paths, state transitions)
- Control-plane integrity (configuration, startup safety, orchestration bounds)
- Traceability (execution and failure path reconstruction)

The prior wording could be read as a design-only review. The updated wording makes clear that documentation alone cannot justify a passing verdict.

---

## Future Drift Monitoring

Watch these locations when making changes:

| Location | What to check |
|----------|--------------|
| `SKILL.md` frontmatter `name` field | Must match repository name |
| `SKILL.md` heading | Must use `/agentic-architecture-assessment` |
| `README.md` installation snippets | Folder must be `.claude/skills/agentic-architecture-assessment/` |
| `README.md` invocation example | Command must be `/agentic-architecture-assessment` |
| `README.md` upgrade section | Paths must match installation section |
| `README.md` Naming and Invocation table | Must stay current if any identity changes |

If a short alias is introduced in the future, it must be documented in:
1. The Naming and Invocation section of README.md
2. This document
3. SKILL.md frontmatter (if Claude Code supports alias declarations)
