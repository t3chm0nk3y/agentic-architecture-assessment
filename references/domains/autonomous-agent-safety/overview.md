# Domain: Autonomous Agent Safety

## Domain Name

Autonomous Agent Safety

## What This Domain Governs

How the system constrains autonomous agent behavior to prevent unintended or harmful actions. This domain assesses whether tool access is controlled, whether conflicting constraints compose safely, whether consequential actions require human approval, whether system prompts are governed as auditable artifacts, and whether guardrail enforcement produces graceful outcomes rather than failures.

## Why It Matters

An autonomous agent that can invoke any tool in any context, bypass access restrictions through composition, execute consequential actions without approval, or fail catastrophically when it hits a guardrail is an agent that cannot be safely deployed. As agents gain more capabilities and operate with less supervision, the safety constraints around their behavior become the primary mechanism for maintaining trust and preventing harm.

Safety gaps in agentic systems are qualitatively different from safety gaps in traditional software. A web application with an authorization bug exposes data. An agent with a tool access control gap can take actions — executing code, modifying files, calling external services — with compounding consequences that may not be immediately visible. The autonomy that makes agents powerful is the same autonomy that makes safety constraints essential.

## Principles in This Domain

| ID | Title | Severity |
|----|-------|----------|
| P9.1 | Tool Access Control | HIGH |
| P9.2 | Tool Restriction Enforcement Under Composition | HIGH |
| P9.3 | Human-in-the-Loop Capability | HIGH |
| P9.4 | Prompt / Behavior Governance | MEDIUM |
| P9.5 | Truncation Not Failure | MEDIUM |

## Common Failure Modes

- **Unrestricted tool access** — every execution context has access to every registered tool, with no mechanism to scope or restrict access per workflow, skill, or user
- **Permissive composition** — when multiple constraints are active, the system takes the union of allowed tools rather than the intersection, effectively granting the most permissive access
- **No approval mechanism** — consequential actions (file writes, external API calls, code execution) proceed without any checkpoint or human confirmation
- **Inline prompts** — system prompts are hard-coded strings scattered across source files, with no versioning, no central registry, and no audit trail
- **Guardrail crashes** — when a token limit, step limit, or budget limit is reached, the execution fails with an exception rather than completing with a truncation indicator

## Typical Repository Indicators

Look for these when forming the system hypothesis:

- Tool allowlist/denylist configuration or access control definitions
- Constraint composition or merging logic (intersection vs. union of tool sets)
- Confirmation or approval hooks (human-in-the-loop callbacks, approval queues)
- System prompt management (prompt files, prompt registries, prompt versioning)
- Guardrail limit handling (max_steps, max_tokens, budget limits, and their completion behavior)

## Common Trace Types to Inspect

- **Tool access control trace** — follow a tool invocation from request through dispatch, checking for access control checks before execution
- **Composition constraint trace** — set up or find a scenario where multiple constraints are active and check how conflicting tool restrictions are resolved
- **Approval mechanism trace** — find a consequential action path and check whether any approval or confirmation gate exists
- **Prompt governance trace** — locate all system prompts and check whether they are centralized, versioned, and loaded from governed sources
- **Guardrail enforcement trace** — find a guardrail limit check and follow the code path when the limit is exceeded: does it raise an exception or complete with truncation?

## Relationship to Adjacent Domains

- **Integration and Tool Model (Domain 4):** P4.1 (Typed Tool Interface) and P4.2 (Tool Registration and Discovery) provide the tool infrastructure that P9.1 (Tool Access Control) constrains. Access control requires a tool registry to control access to.
- **Scalability and Extensibility (Domain 8):** P8.2 (Tool Extensibility) must interact safely with P9.1. When new tools are dynamically added, access control must still be enforced.
- **Orchestration Model (Domain 7):** P7.3 (Guardrail Enforcement) is the mechanism that P9.5 (Truncation Not Failure) assesses for graceful behavior. A system with guardrails (P7.3) that crash on limit (P9.5 GAP) has a safety gap.
- **Configuration (Domain 5):** P5.1-P5.3 may govern the configuration of tool access policies, prompt versions, and guardrail thresholds. Poor configuration discipline (Domain 5 gaps) can undermine safety controls.

## Common False Positives in Assessment

- **Tool type definitions do not mean tool access control** — check for actual access checks at invocation time, not just that tools have categories or types
- **A confirmation prompt in the UI does not mean human-in-the-loop** — check whether the agent runtime itself has an approval mechanism, not just the client application
- **Prompt files do not mean prompt governance** — check that prompts are versioned and loaded from a governed source, not just that they exist as separate files
- **Guardrail configuration does not mean graceful enforcement** — check the code path when a limit is exceeded, not just that limits are configurable
- **An admin role does not mean tool access control** — user-level authorization is different from context-level tool access scoping within the agent runtime
