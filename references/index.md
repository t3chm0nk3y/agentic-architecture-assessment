# References Index

Master index of all framework documents, domain overviews, and principles.

These references support the assessment of critical best practices for agentic applications. They define the behavioral principles that an effective agentic system must implement and enforce, the methodology for evaluating implementation quality through trace-based evidence collection, and the structured reporting format for findings. The references are organized into assessment methodology (framework), domain-specific principles with verdict guidance (domains), and calibration material (examples).

---

## Framework

| Document | Purpose |
|----------|---------|
| [principle-template.md](framework/principle-template.md) | Canonical 19-field template for all principle files |
| [report-schema.md](framework/report-schema.md) | Gap register report structure and field definitions |
| [evidence-model.md](framework/evidence-model.md) | Evidence hierarchy and evidence status values |
| [verdicts.md](framework/verdicts.md) | PASS/PARTIAL/GAP/UNASSESSABLE/N/A definitions and decision tree |
| [system-classification.md](framework/system-classification.md) | System type taxonomy driving principle applicability |
| [sampling-rules.md](framework/sampling-rules.md) | Bounded evidence collection rules |
| [stop-conditions.md](framework/stop-conditions.md) | When to stop collecting evidence |
| [severity-confidence-model.md](framework/severity-confidence-model.md) | Severity and confidence level definitions |
| [glossary.md](framework/glossary.md) | Normalized terminology (~30 terms) |

---

## Domains and Principles

### Domain 1: Execution Boundaries

[Overview](domains/execution-boundaries/overview.md)

| ID | Title | Severity | File |
|----|-------|----------|------|
| P1.1 | Execution Centralization | HIGH | [P1.1](domains/execution-boundaries/P1.1-execution-centralization.md) |
| P1.2 | Transport / Runtime Separation | HIGH | [P1.2](domains/execution-boundaries/P1.2-transport-runtime-separation.md) |
| P1.3 | UI as Consumer | MEDIUM | [P1.3](domains/execution-boundaries/P1.3-ui-as-consumer.md) |
| P1.4 | Execution Errors Separate from Transport Errors | MEDIUM | [P1.4](domains/execution-boundaries/P1.4-execution-errors-separate.md) |

### Domain 2: Configuration

[Overview](domains/configuration/overview.md)

| ID | Title | Severity | File |
|----|-------|----------|------|
| P2.1 | Externalized Configuration | HIGH | [P2.1](domains/configuration/P2.1-externalized-configuration.md) |
| P2.2 | Configuration Validation at Startup | HIGH | [P2.2](domains/configuration/P2.2-configuration-validation.md) |
| P2.3 | Startup as a Safety Gate | HIGH | [P2.3](domains/configuration/P2.3-startup-safety-gate.md) |
| P2.4 | Configuration as Single Source of Truth | MEDIUM | [P2.4](domains/configuration/P2.4-configuration-single-source.md) |

### Domain 3: Observability

[Overview](domains/observability/overview.md)

| ID | Title | Severity | File |
|----|-------|----------|------|
| P3.1 | Run Traceability | CRITICAL | [P3.1](domains/observability/P3.1-run-traceability.md) |
| P3.2 | Structured Logging | HIGH | [P3.2](domains/observability/P3.2-structured-logging.md) |
| P3.3 | Structured Events | HIGH | [P3.3](domains/observability/P3.3-structured-events.md) |
| P3.4 | Event Delivery Reliability | MEDIUM | [P3.4](domains/observability/P3.4-event-delivery-reliability.md) |
| P3.5 | Failure Contextualization | HIGH | [P3.5](domains/observability/P3.5-failure-contextualization.md) |
| P3.6 | Observability Coverage | MEDIUM | [P3.6](domains/observability/P3.6-observability-coverage.md) |

### Domain 4: Integration and Tool Model

[Overview](domains/integration-tool-model/overview.md)

| ID | Title | Severity | File |
|----|-------|----------|------|
| P4.1 | Single Integration Surface | HIGH | [P4.1](domains/integration-tool-model/P4.1-single-integration-surface.md) |
| P4.2 | Tool Abstraction and Discoverability | HIGH | [P4.2](domains/integration-tool-model/P4.2-tool-abstraction-and-discoverability.md) |
| P4.3 | Normalization at the Adapter Boundary | HIGH | [P4.3](domains/integration-tool-model/P4.3-normalization-at-the-adapter-boundary.md) |
| P4.4 | Tool Observability | MEDIUM | [P4.4](domains/integration-tool-model/P4.4-tool-observability.md) |
| P4.5 | Adapter Self-Contained Authentication | MEDIUM | [P4.5](domains/integration-tool-model/P4.5-adapter-self-contained-authentication.md) |

### Domain 5: Schema and Contract Discipline

[Overview](domains/schema-contract-discipline/overview.md)

| ID | Title | Severity | File |
|----|-------|----------|------|
| P5.1 | Canonical Contract Centralization | HIGH | [P5.1](domains/schema-contract-discipline/P5.1-canonical-contract-centralization.md) |
| P5.2 | Definition Schemas vs Runtime Records | HIGH | [P5.2](domains/schema-contract-discipline/P5.2-definition-schemas-vs-runtime-records.md) |
| P5.3 | DTO Isolation | MEDIUM | [P5.3](domains/schema-contract-discipline/P5.3-dto-isolation.md) |
| P5.4 | Input Validation at All Entry Points | HIGH | [P5.4](domains/schema-contract-discipline/P5.4-input-validation-at-all-entry-points.md) |
| P5.5 | LLM Output Validation | HIGH | [P5.5](domains/schema-contract-discipline/P5.5-llm-output-validation.md) |

### Domain 6: Error Handling

[Overview](domains/error-handling/overview.md)

| ID | Title | Severity | File |
|----|-------|----------|------|
| P6.1 | Structured Error Model | HIGH | [P6.1](domains/error-handling/P6.1-structured-error-model.md) |
| P6.2 | Error Classification | MEDIUM | [P6.2](domains/error-handling/P6.2-error-classification.md) |
| P6.3 | Failure Propagation | CRITICAL | [P6.3](domains/error-handling/P6.3-failure-propagation.md) |
| P6.4 | Infrastructure Failure Resilience | HIGH | [P6.4](domains/error-handling/P6.4-infrastructure-failure-resilience.md) |
| P6.5 | Retry Safety | MEDIUM | [P6.5](domains/error-handling/P6.5-retry-safety.md) |

### Domain 7: Orchestration Model

[Overview](domains/orchestration-model/overview.md)

| ID | Title | Severity | File |
|----|-------|----------|------|
| P7.1 | Deliberate Orchestration Framework | MEDIUM | [P7.1](domains/orchestration-model/P7.1-deliberate-orchestration-framework.md) |
| P7.2 | Explicit State Transitions | HIGH | [P7.2](domains/orchestration-model/P7.2-explicit-state-transitions.md) |
| P7.3 | Bounded Autonomous Execution | CRITICAL | [P7.3](domains/orchestration-model/P7.3-bounded-autonomous-execution.md) |
| P7.4 | Deterministic Workflow Structure | HIGH | [P7.4](domains/orchestration-model/P7.4-deterministic-workflow-structure.md) |
| P7.5 | Run State Auditability | HIGH | [P7.5](domains/orchestration-model/P7.5-run-state-auditability.md) |

### Domain 8: Scalability and Extensibility

[Overview](domains/scalability-extensibility/overview.md)

| ID | Title | Severity | File |
|----|-------|----------|------|
| P8.1 | Workflow Extensibility | HIGH | [P8.1](domains/scalability-extensibility/P8.1-workflow-extensibility.md) |
| P8.2 | Tool Extensibility | HIGH | [P8.2](domains/scalability-extensibility/P8.2-tool-extensibility.md) |
| P8.3 | Skill Extensibility | HIGH | [P8.3](domains/scalability-extensibility/P8.3-skill-extensibility.md) |
| P8.4 | Configuration-Driven Behavior | MEDIUM | [P8.4](domains/scalability-extensibility/P8.4-configuration-driven-behavior.md) |
| P8.5 | Separation of Definition from Execution | HIGH | [P8.5](domains/scalability-extensibility/P8.5-separation-of-definition-from-execution.md) |

### Domain 9: Autonomous Agent Safety

[Overview](domains/autonomous-agent-safety/overview.md)

| ID | Title | Severity | File |
|----|-------|----------|------|
| P9.1 | Tool Access Control | HIGH | [P9.1](domains/autonomous-agent-safety/P9.1-tool-access-control.md) |
| P9.2 | Tool Restriction Enforcement Under Composition | HIGH | [P9.2](domains/autonomous-agent-safety/P9.2-tool-restriction-enforcement-under-composition.md) |
| P9.3 | Human-in-the-Loop Capability | HIGH | [P9.3](domains/autonomous-agent-safety/P9.3-human-in-the-loop-capability.md) |
| P9.4 | Prompt / Behavior Governance | MEDIUM | [P9.4](domains/autonomous-agent-safety/P9.4-prompt-behavior-governance.md) |
| P9.5 | Truncation Not Failure | MEDIUM | [P9.5](domains/autonomous-agent-safety/P9.5-truncation-not-failure.md) |

---

## Examples

| Document | Purpose |
|----------|---------|
| [sample-gap-register.md](examples/sample-gap-register.md) | Complete example assessment report for calibration |
| [example-assessment-snippets.md](examples/example-assessment-snippets.md) | Good/weak verdict justification examples |

---

## Summary

- **9 domains**, **44 principles**
- **3 CRITICAL** severity principles: P3.1, P6.3, P7.3
- **27 HIGH** severity principles
- **14 MEDIUM** severity principles
- **3 principles** requiring behavioral verification: P7.3, P9.2, P9.5
