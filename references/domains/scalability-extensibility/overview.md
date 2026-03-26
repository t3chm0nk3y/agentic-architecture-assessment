# Domain: Scalability and Extensibility

## Domain Name

Scalability and Extensibility

## What This Domain Governs

How the system accommodates growth in capability — new workflows, new tools, new skills, new behavioral modes — without requiring modifications to the core execution engine. This domain assesses whether the system is architected for extension through addition rather than modification, and whether behavioral variation is driven by configuration and declarative definitions rather than code changes.

## Why It Matters

An agentic system that requires runtime modification to add a new tool, workflow, or skill is a system that cannot scale safely. Every modification to the execution engine risks introducing regressions in existing behavior. When tool addition requires changes to the agent loop, when new workflows require modifying the runtime, or when behavioral variation requires code changes, the system's growth trajectory is limited by its core architecture's fragility.

Extensibility gaps compound. A system that requires code changes to add tools will eventually have tool-specific branching scattered through its runtime. A system that cannot separate definitions from execution will eventually have workflow logic hard-coded into the engine. Each missing extension point creates pressure to modify the core, and each core modification makes future extension harder.

## Principles in This Domain

| ID | Title | Severity |
|----|-------|----------|
| P8.1 | Workflow Extensibility | HIGH |
| P8.2 | Tool Extensibility | HIGH |
| P8.3 | Skill Extensibility | HIGH |
| P8.4 | Configuration-Driven Behavior | MEDIUM |
| P8.5 | Separation of Definition from Execution | HIGH |

## Common Failure Modes

- **Hard-coded workflows** — new workflow types require modifying the runtime's main loop or adding conditional branches to the execution engine
- **Tool registration coupled to the agent loop** — adding a new tool requires modifying the agent loop, tool dispatch, or LLM prompt construction code
- **Monolithic skill definitions** — skills or personas are defined inline in runtime code rather than as external, loadable definitions
- **Behavior controlled by code branches** — varying agent behavior (different models, different prompt strategies, different guardrails) requires code changes rather than configuration changes
- **Definitions entangled with execution** — workflow graphs, tool schemas, and skill definitions are constructed inside the execution engine rather than loaded from external sources

## Typical Repository Indicators

Look for these when forming the system hypothesis:

- Plugin or registry patterns (tool registries, workflow registries, skill catalogs)
- Configuration files that control behavioral variation (YAML, TOML, JSON config)
- Loader or discovery mechanisms (dynamic imports, directory scanning, plugin interfaces)
- Separation between definition files and runtime modules
- Factory or builder patterns that construct behavior from configuration

## Common Trace Types to Inspect

- **Tool addition trace** — simulate adding a new tool and determine which files must be modified: only a definition, or also the runtime?
- **Workflow addition trace** — simulate adding a new workflow type and determine whether the runtime requires modification
- **Configuration variation trace** — identify a behavioral parameter (model selection, retry count, guardrail threshold) and check whether changing it requires code modification or configuration change
- **Definition loading trace** — follow how workflow, tool, and skill definitions are loaded into the runtime: are they read from external sources or constructed in code?

## Relationship to Adjacent Domains

- **Integration and Tool Model (Domain 4):** P4.1 (Typed Tool Interface) and P4.2 (Tool Registration and Discovery) are prerequisites for P8.2 (Tool Extensibility). A system cannot have extensible tooling without a consistent interface and registration mechanism.
- **Orchestration Model (Domain 7):** P7.1 (Agent Loop Isolation) enables P8.1 (Workflow Extensibility). If the agent loop is not isolated, workflows cannot be added without modifying it.
- **Configuration (Domain 5):** P5.1-P5.3 provide the configuration infrastructure that P8.4 (Configuration-Driven Behavior) depends on. Without layered, validated configuration, configuration-driven behavior is unreliable.
- **Autonomous Agent Safety (Domain 9):** P9.1 (Tool Access Control) constrains P8.2 (Tool Extensibility). When new tools are added dynamically, access control must still apply.

## Common False Positives in Assessment

- **A plugin interface does not mean extensibility works** — check that at least one workflow, tool, or skill is actually loaded through the extension mechanism, not just that the interface exists
- **Configuration files do not mean configuration-driven behavior** — check that the configuration actually controls behavioral variation, not just static deployment settings
- **Separate definition files do not mean separation from execution** — check that definitions are loaded by the runtime, not imported and reconstructed inline
- **A tool registry does not mean tools are dynamically extensible** — check whether new tools can be added to the registry without modifying the code that reads from it
