# Glossary

Normalized terminology used throughout the assessment framework. All principle files, framework documents, and reports must use these terms consistently.

---

## Execution Concepts

### Run

A single top-level execution instance. A run begins when the system receives a request to execute and ends when a terminal state is reached (completed, failed, or cancelled). A run may contain multiple steps.

### Step

A unit of execution within a run. Steps are the atomic building blocks of workflows. Each step has a defined type (automated, agent-driven, approval gate, etc.), inputs, outputs, and a lifecycle.

### Workflow

A predefined, ordered sequence of steps. A workflow is deterministic in structure — the steps are known before execution begins. Contrast with autonomous execution, where the agent determines its own action sequence.

### Agent Loop

The iterative reasoning cycle in which an autonomous agent selects actions, invokes tools, observes results, and decides the next action. May be bounded by guardrails.

### Dispatch

The routing decision that determines which execution path a request follows. In well-structured systems, dispatch is a single, identifiable decision point — not distributed logic.

---

## Capability Concepts

### Skill

A reusable behavioral unit that guides agent reasoning. A skill typically includes a system prompt, output constraints, and optionally tool restrictions. Skills are composable — multiple skills may be active simultaneously.

### Tool

An external capability that the agent can invoke during execution. Tools are registered in a catalog and discovered at startup or runtime. Tools may be provided by external services, APIs, databases, or other systems. The protocol used (MCP, function calling, REST, etc.) is an implementation detail.

### Tool Catalog

A registry of all tools available to the agent, typically built at startup from tool providers. The catalog enables tool discovery, schema inspection, and access control.

### Orchestration

The mechanism by which execution is sequenced, bounded, and controlled. Orchestration may be driven by a state machine, graph framework, simple loop, or other mechanism. The invariant is that orchestration is deliberate and explicit.

---

## Architecture Concepts

### Control Plane

The layer of the system that makes execution decisions: dispatch, routing, state transitions, access control, and enforcement. Contrast with the data plane, which carries payloads and results.

### Data Plane

The layer of the system that carries actual data: tool inputs/outputs, run context, event payloads, API request/response bodies.

### Contract

A defined data shape with explicit structure and validation rules. Contracts are authoritative — they define what data looks like across boundaries. Synonymous with "schema" or "model" in some ecosystems.

### Schema

A structural definition for data validation. Used interchangeably with "contract" when referring to data shapes. When referring specifically to validation rules (JSON Schema, Pydantic model, Zod schema), "schema" is preferred.

### Enforcement Point

A location in the system where a rule is checked and violations are rejected. Enforcement points are where invariants become real — a rule that exists in documentation but is not enforced at runtime is not an enforcement point.

### Propagation

How a value (identifier, error, state change) flows across boundaries within the system. Propagation can be explicit (passed as a parameter) or implicit (context-local, middleware-injected).

---

## Assessment Concepts

### Principle

A behavioral invariant that the framework assesses. Each principle describes a required behavior, not a specific implementation. Principles are grouped into domains.

### Domain

A category of related principles that govern a specific architectural concern (e.g., execution boundaries, observability, error handling).

### Verdict

The assessment outcome for a single principle: PASS, PARTIAL, GAP, UNASSESSABLE, or N/A. See `verdicts.md` for definitions.

### Evidence

Observable information that supports or contradicts a verdict. Evidence has types (runtime, trace, implementation, declarative) with different weights. See `evidence-model.md`.

### Evidence Status

The quality of evidence for a finding: confirmed, partial, missing, or contradictory. See `evidence-model.md`.

### Applicability

Whether a principle is relevant to the system under assessment. Determined by system classification, not assessor preference. See `system-classification.md`.

### Behavioral Verification

Confirmation of a principle's invariant through runtime testing rather than static analysis. Required for principles where code reading alone cannot confirm enforcement.

### Gap Register

The structured report of findings produced by the assessment. Contains per-principle findings organized by severity. See `report-schema.md`.

---

## Boundary Terms

### Normalization Boundary

The point where external vendor-specific data is transformed into internal canonical shapes. This should happen exactly once, at the integration/adapter layer.

### Transport Boundary

The point where internal domain models are projected to external API shapes (DTOs, response bodies). Transport shapes should be distinct from domain models.

### Module Boundary

The interface between two modules in the system. Dependency direction, contract ownership, and enforcement rules apply at module boundaries.
