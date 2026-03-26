# Repository Consistency Review

**Date:** 2026-03-26
**Purpose:** Document the canonical identity and naming alignment across the repository. Track consistency to prevent future drift.

---

## Canonical Identity

| Aspect | Value |
|--------|-------|
| Repository | `agentic-architecture-assessment` |
| Skill name (SKILL.md frontmatter) | `agentic-architecture-assessment` |
| Install folder | `.claude/skills/agentic-architecture-assessment/` |
| Slash command | `/agentic-architecture-assessment` |
| Short alias | None defined |

All identifiers are aligned to the repository name.

---

## Prior Naming Drift (Resolved)

The original skill identity was `agentic-audit`, which diverged from the repository name `agentic-architecture-assessment`. This was resolved by adopting the repository name as the canonical identity everywhere.

Additionally, report headers and output paths used "audit" while the product name uses "assessment." This was normalized to "assessment" throughout.

---

## Current Consistency Status

| Check | Status |
|-------|--------|
| Skill frontmatter name matches repo name | Consistent |
| SKILL.md heading uses canonical slash command | Consistent |
| README install paths use canonical folder | Consistent |
| README invocation uses canonical command | Consistent |
| README upgrade paths match install paths | Consistent |
| Report schema header uses "Assessment" | Consistent |
| Sample gap register header uses "Assessment" | Consistent |
| Report output path uses "assessment" | Consistent |
| References index framing aligns with product positioning | Consistent |
| SKILL.md and README describe the same artifact and purpose | Consistent |

---

## Files That Carry Identity References

These files contain the canonical identity and must be updated together if the identity ever changes:

| File | What to check |
|------|--------------|
| `SKILL.md` | Frontmatter `name`, heading, description |
| `README.md` | Installation paths, invocation command, upgrade paths, Naming and Invocation table |
| `references/framework/report-schema.md` | Report header template, output path |
| `references/examples/sample-gap-register.md` | Example report header |
| `references/index.md` | Example descriptions |
| `docs/repository-consistency-review.md` | This document |

---

## Future Drift Monitoring

If any of the following changes are made, check all files listed above:

- Repository rename
- Skill name change
- Introduction of a short alias
- Report format versioning
- New example or calibration files added
