# Domain: Schema and Contract Discipline

## Domain Name

Schema and Contract Discipline

## What This Domain Governs

How the system defines, validates, and enforces data structures across its boundaries. This domain assesses whether data contracts are centralized, whether definition-time and runtime data are properly separated, whether transport shapes are isolated from domain models, and whether all inputs — including LLM outputs — are validated before use.

## Why It Matters

Agentic systems pass structured data across many boundaries: between API layers and runtime, between runtime and tools, between LLM providers and orchestration, between configuration files and execution engines. At each boundary, assumptions about data shape create implicit contracts. When those contracts are implicit — existing only in the minds of developers or scattered across multiple module implementations — any change to a data structure risks cascading breakage in modules that assumed a shape that no longer holds.

Schema discipline makes these contracts explicit. Canonical definitions ensure there is one source of truth. Validation at boundaries ensures malformed data is rejected early rather than propagated until it causes a confusing downstream failure. LLM output validation is especially critical: LLMs produce probabilistic output that may deviate from expected schemas in subtle ways that pass naive checks but break downstream processing.

## Principles in This Domain

| ID | Title | Severity |
|----|-------|----------|
| P5.1 | Canonical Contract Centralization | HIGH |
| P5.2 | Definition Schemas vs Runtime Records | HIGH |
| P5.3 | DTO Isolation | MEDIUM |
| P5.4 | Input Validation at All Entry Points | HIGH |
| P5.5 | LLM Output Validation | HIGH |

## Common Failure Modes

- **Duplicate definitions** — the same entity defined in multiple modules with slightly different field sets, leading to silent data loss or corruption during transfers
- **Conflated definition and runtime data** — a workflow template that also contains runtime state, making it impossible to distinguish the reusable definition from the specific execution
- **Leaky DTOs** — API response shapes used directly as internal domain models, coupling the internal architecture to the transport layer
- **Missing validation** — external inputs accepted and processed without schema validation, allowing malformed data to propagate
- **Trusted LLM output** — LLM responses parsed and used directly without validation, with failures surfacing far from the point of ingestion

## Typical Repository Indicators

Look for these when forming the system hypothesis:

- Schema definition files (JSON Schema, Pydantic models, TypeScript interfaces, protobuf definitions, Zod schemas)
- Validation code at API boundaries (request validation middleware, input parsers)
- Separate model directories for API models vs domain models
- Template or definition storage separate from runtime record storage
- LLM response parsing code with or without validation

## Common Trace Types to Inspect

- **Contract duplication trace** — search for a core entity name across the codebase and check whether it is defined in more than one location
- **Definition/runtime separation trace** — follow a workflow or skill definition from its storage through to execution, checking whether the definition record and runtime record are distinct types
- **DTO isolation trace** — follow data from an API endpoint through to the domain layer, checking whether the API response shape is used internally or mapped to a separate domain model
- **Input validation trace** — follow an external input from the entry point to the first point of use, checking whether validation occurs between receipt and use
- **LLM output trace** — follow an LLM response from the provider adapter through parsing to downstream use, checking for schema validation

## Relationship to Adjacent Domains

- **Integration and Tool Model (Domain 4):** P4.3 (Normalization at the Adapter Boundary) is the complement of P5.1 — P4.3 normalizes external data at the adapter boundary, P5.1 ensures the canonical types exist for normalization targets.
- **Execution Boundaries (Domain 1):** P5.2 (Definition vs Runtime) is related to P1.3 (State Isolation) — both address the separation of static structure from dynamic execution state, at different levels of abstraction.
- **Error Handling (Domain 6):** P5.4 and P5.5 (Input and LLM Output Validation) interact with error handling — validation failures need to produce structured, actionable errors per Domain 6 principles.
- **Configuration (Domain 8):** Configuration files and definition files may overlap. P5.4 requires validation of configuration inputs; P5.2 requires that definition schemas are separate from runtime records.

## Common False Positives in Assessment

- **Presence of a models directory does not mean canonical centralization** — check that other modules do not redefine the same entities independently
- **Type annotations do not mean validation** — a function signature with typed parameters does not validate incoming data at runtime unless a validation library enforces the types
- **Separate API and domain model files do not mean DTO isolation** — check that the API models are actually mapped to domain models, not that domain models are simply re-exported as API models
- **LLM response parsing does not mean LLM output validation** — extracting fields from a JSON response is not the same as validating the response against an expected schema
- **Schema files in the repository do not mean schemas are enforced** — check that the schemas are actually used for validation at runtime, not just documentation
