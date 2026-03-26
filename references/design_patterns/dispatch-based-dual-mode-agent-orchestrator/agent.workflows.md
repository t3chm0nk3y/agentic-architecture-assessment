# agent.workflows — Workflow and Skill Definition Specification

**Version:** 1.0.0  
**Status:** Authoritative — Behavioral Specification  
**Scope:** Workflow Definitions, Skill Definitions, Registry, Definition Schemas  
**Parent:** technical-design-specification.md

---

## 1. Purpose

agent.workflows is the definition layer of the platform. It owns what can be executed — not how.

Every workflow and skill available to the runtime is defined, validated, and registered by agent.workflows. No other module may define workflows or skills. agent.core consumes these definitions at startup and at execution time but never modifies them. agent.api exposes them for discovery but never interprets them.

agent.workflows is the single source of truth for:

- what workflows exist and what steps they contain
- what skills exist and what behavioral instructions they carry
- how workflow and skill definitions are structured
- how definitions are versioned
- how the registry is populated and queried

---

## 2. What agent.workflows Owns vs Delegates

**Owns**

- Workflow definition schema and validation
- Skill definition schema and validation
- Workflow and skill registry (registration, discovery, version resolution)
- Definition file loading from disk (paths from agent.yaml)
- Registry persistence to MongoDB
- Step definition schema (structure of individual steps within a workflow)
- Input mapping schema (how steps declare their inputs from RunContext)

**Does Not Own**

- Workflow execution → agent.core
- Step execution logic → agent.core
- Skill application at runtime (prompt composition, tool filtering) → agent.core
- Run context read/write semantics → agent.core
- Canonical contracts (Run, Step, Event, etc.) → agent.core
- MCP tool definitions → agent.adapters (discovered at startup by agent.core)
- HTTP endpoints for discovery → agent.api
- Storage and cache adapters → agent.adapters

---

## 3. Internal Component Map

```
agent.workflows
├── definitions/
│   ├── workflow.py            # Workflow definition Pydantic model
│   ├── skill.py               # Skill definition Pydantic model
│   └── step.py                # Step definition Pydantic model (within workflows)
├── registry/
│   ├── registry.py            # Unified registry: register, resolve, query
│   └── loader.py              # Load YAML files from disk, validate, register
└── schemas/
    ├── input_mapping.py       # Input mapping resolution schema
    └── output_format.py       # Output format constraint schema
```

---

## 4. Workflow Definitions

A workflow is a predefined, ordered sequence of steps. It is deterministic in structure — the steps are known before execution begins. Workflows are defined as YAML files in the directory specified by `agent.yaml → registries.workflows_dir`.

### 4.1 Workflow Definition Schema

```python
class WorkflowDefinition(BaseModel):
    workflow_id: str
    name: str
    version: str                          # Semantic version (e.g., "1.0.0")
    description: str
    tags: list[str] = []
    input_schema: dict                    # JSON Schema for validated input
    output_schema: dict                   # JSON Schema for validated output
    steps: list[StepDefinition]           # Ordered; execution follows this sequence
    active_skills: list[str] = []         # Default run-level skills (overridable at request time)
```

### 4.2 Validation Rules

- `workflow_id` must be unique within a version; the combination of `workflow_id` + `version` is the primary key
- `steps` must contain at least one entry
- All `skill_ids` referenced in step definitions must exist in the skill registry at startup
- `input_schema` and `output_schema` must be valid JSON Schema
- `version` must follow semantic versioning (MAJOR.MINOR.PATCH)

### 4.3 Example Workflow YAML

```yaml
workflow_id: classify-ttp
name: Classify Adversary TTP
version: "1.0.0"
description: >
  Maps an adversary activity description to MITRE ATT&CK tactics,
  techniques, and subtechniques.
tags:
  - mitre
  - detection-engineering

input_schema:
  type: object
  properties:
    description:
      type: string
      description: Free-text description of adversary behavior
  required:
    - description

output_schema:
  type: object
  properties:
    classifications:
      type: array
      items:
        type: object
        properties:
          tactic: { type: string }
          technique_id: { type: string }
          technique_name: { type: string }
          confidence: { type: number }

steps:
  - step_id: normalize-input
    name: Normalize Input
    step_type: AGENT
    skill_ids:
      - behavior-normalizer
    input_mapping:
      raw_description: inputs.description
    output_schema:
      type: object
      properties:
        normalized_behaviors:
          type: array
          items: { type: string }

  - step_id: classify
    name: ATT&CK Classification
    step_type: AGENT
    skill_ids:
      - attck-analyst
    input_mapping:
      behaviors: steps.normalize-input.output.normalized_behaviors
    output_schema:
      type: object
      properties:
        classifications:
          type: array

  - step_id: review
    name: Analyst Review
    step_type: HUMAN_GATE
    approval_context_mapping:
      classifications: steps.classify.output.classifications
      original_input: inputs.description
```

---

## 5. Step Definitions

A step definition describes a single unit of work within a workflow. It is a structural declaration — it says what should happen, not how. Execution semantics are owned by agent.core.

### 5.1 Step Definition Schema

```python
class StepDefinition(BaseModel):
    step_id: str
    name: str
    step_type: StepType                   # AUTOMATED | AGENT | HUMAN_GATE
    skill_ids: list[str] = []             # AGENT steps only; overrides run-level skills
    input_mapping: dict[str, str] = {}    # Declarative mapping from RunContext
    output_schema: dict | None = None     # JSON Schema; required for AGENT steps
    tool_ids: list[str] = []              # AUTOMATED steps: explicit tools to invoke
    approval_context_mapping: dict[str, str] = {}  # HUMAN_GATE steps: data shown to reviewer
    max_tool_calls: int | None = None     # Override agent.yaml default
    max_tokens: int | None = None         # Override agent.yaml default
    timeout_seconds: int | None = None    # Override agent.yaml default
    continue_on_failure: bool = False     # If true, failure is recorded but execution continues
```

### 5.2 Step Type Constraints

| Field | AUTOMATED | AGENT | HUMAN_GATE |
|-------|-----------|-------|------------|
| skill_ids | Ignored | Required (≥1) | Ignored |
| tool_ids | Required (≥1) | Ignored (agent selects freely) | Ignored |
| output_schema | Optional | Required | Ignored |
| approval_context_mapping | Ignored | Ignored | Required |
| max_tool_calls | Applies | Applies | Ignored |
| max_tokens | Ignored | Applies | Ignored |

### 5.3 Input Mapping

Input mappings are dot-path expressions that resolve values from the RunContext at execution time. The runtime (agent.core) performs the resolution; agent.workflows only defines the schema.

```yaml
input_mapping:
  # From original run input
  policy_id: inputs.policy_id

  # From a prior step's output
  asset_list: steps.fetch-assets.output.assets

  # From run metadata
  requester: metadata.initiated_by
```

If no `input_mapping` is specified, the full RunContext is passed to the step.

### 5.4 Approval Context Mapping

For HUMAN_GATE steps, `approval_context_mapping` defines what data is presented to the reviewer. It uses the same dot-path syntax as input mappings.

```yaml
approval_context_mapping:
  classifications: steps.classify.output.classifications
  original_input: inputs.description
```

The resolved data is written to the `context` field of the Approval canonical entity.

---

## 6. Skill Definitions

A skill is a reusable behavioral unit that shapes how the agent reasons. Skills are defined as YAML files in the directory specified by `agent.yaml → registries.skills_dir`.

### 6.1 Skill Definition Schema

```python
class SkillDefinition(BaseModel):
    skill_id: str
    name: str
    version: str                          # Semantic version
    description: str                      # When and why to activate this skill
    system_prompt: str                    # The behavioral instruction set
    output_format: OutputFormat | None = None  # Constraints on response structure
    tool_restrictions: ToolRestrictions | None = None  # Optional tool filtering
```

### 6.2 Output Format

```python
class OutputFormat(BaseModel):
    type: str                             # "json_schema" | "text" | "markdown"
    schema: dict | None = None            # JSON Schema (when type is "json_schema")
    instructions: str | None = None       # Freeform formatting instructions
```

When a skill specifies an output_format, agent.core applies it as a constraint on the agent step's output. If the step definition also has an `output_schema`, the step's schema takes precedence — the skill's output_format acts as a default.

### 6.3 Tool Restrictions

```python
class ToolRestrictions(BaseModel):
    mode: str                             # "allowlist" | "denylist"
    tool_ids: list[str]                   # tool_id values from the tool catalog
```

When a skill specifies tool_restrictions, agent.core filters the tool catalog for any agent step using that skill. In allowlist mode, only listed tools are available. In denylist mode, listed tools are excluded.

If multiple skills are active on a step with conflicting restrictions, the intersection of allowed tools is used (most restrictive wins).

### 6.4 Validation Rules

- `skill_id` must be unique within a version; `skill_id` + `version` is the primary key
- `system_prompt` must not be empty
- `tool_restrictions.tool_ids` are validated against the tool catalog at startup — warnings are logged for unrecognized tool IDs but validation does not fail (tools may be on optional servers)
- `version` must follow semantic versioning

### 6.5 Example Skill YAML

```yaml
skill_id: attck-analyst
name: ATT&CK Analyst
version: "1.0.0"
description: >
  Classifies adversary behaviors against the MITRE ATT&CK framework.
  Activate when mapping threat activity to tactics, techniques,
  and subtechniques.

system_prompt: |
  You are an expert MITRE ATT&CK analyst. Given a description of
  adversary behavior, identify the most relevant ATT&CK tactics,
  techniques, and subtechniques.

  For each classification:
  - Provide the tactic name and ID
  - Provide the technique name and ID (include subtechnique if applicable)
  - Assign a confidence score between 0.0 and 1.0
  - Provide a brief rationale

  Prefer specificity — map to subtechniques where the behavior is
  sufficiently detailed. If the behavior spans multiple techniques,
  return all applicable mappings.

output_format:
  type: json_schema
  schema:
    type: object
    properties:
      classifications:
        type: array
        items:
          type: object
          properties:
            tactic: { type: string }
            technique_id: { type: string }
            technique_name: { type: string }
            subtechnique_id: { type: string, nullable: true }
            confidence: { type: number, minimum: 0, maximum: 1 }
            rationale: { type: string }
          required:
            - tactic
            - technique_id
            - technique_name
            - confidence
            - rationale

tool_restrictions:
  mode: denylist
  tool_ids:
    - servicenow.create_ticket    # No side effects during classification
```

---

## 7. Registry

The registry is the runtime index of all available workflows and skills. It is populated at startup and queryable throughout the application lifecycle.

### 7.1 Architecture

```
YAML files on disk (authoring-time source of truth)
    → loader.py reads and validates at startup
    → registry.py registers into MongoDB (runtime source of truth)
    → agent.core resolves definitions from registry at execution time
    → agent.api exposes registry for discovery via GET /workflows, GET /skills
```

### 7.2 Registry Operations

```python
class Registry:
    async def register_workflow(self, definition: WorkflowDefinition) -> None:
        """Register or update a workflow definition."""

    async def register_skill(self, definition: SkillDefinition) -> None:
        """Register or update a skill definition."""

    async def resolve_workflow(
        self, workflow_id: str, version: str | None = None
    ) -> WorkflowDefinition:
        """Resolve a workflow by ID. If version is None, return highest version."""

    async def resolve_skill(
        self, skill_id: str, version: str | None = None
    ) -> SkillDefinition:
        """Resolve a skill by ID. If version is None, return highest version."""

    async def list_workflows(
        self, tags: list[str] | None = None
    ) -> list[WorkflowSummary]:
        """List all registered workflows, optionally filtered by tags."""

    async def list_skills(self) -> list[SkillSummary]:
        """List all registered skills."""

    async def validate_references(self, tool_catalog: list[str]) -> list[str]:
        """
        Validate all skill and workflow cross-references.
        Returns a list of warnings (e.g., unrecognized tool_ids).
        Called by agent.core at startup after the tool catalog is built.
        """
```

### 7.3 Loader

The loader reads YAML files from the directories specified in `agent.yaml` and registers them into the registry.

```python
class DefinitionLoader:
    async def load_all(self, config: AgentConfig) -> LoadResult:
        """
        Load all workflow and skill definitions from disk.

        1. Read all .yaml files from config.registries.workflows_dir
        2. Validate each against WorkflowDefinition schema
        3. Read all .yaml files from config.registries.skills_dir
        4. Validate each against SkillDefinition schema
        5. Register all validated definitions into the registry
        6. Return LoadResult with counts and any validation errors

        Files that fail validation are logged and skipped — they do not
        block startup. However, workflows referencing missing skills will
        fail validation in step 5 of agent.core's startup sequence.
        """
```

```python
class LoadResult(BaseModel):
    workflows_loaded: int
    skills_loaded: int
    errors: list[LoadError]

class LoadError(BaseModel):
    file_path: str
    error: str
```

### 7.4 Version Resolution

- Multiple versions of a workflow or skill may coexist in the registry
- When `version` is not specified in a request, the highest registered version is resolved (using semantic version ordering)
- The resolved version is recorded on the Run entity (`workflow_version`) by agent.core at run creation time
- Registering a definition with an existing `id` + `version` replaces the previous entry (idempotent upsert)

### 7.5 MongoDB Collections

| Collection | Contents | Key |
|------------|----------|-----|
| workflow_definitions | WorkflowDefinition documents | workflow_id + version |
| skill_definitions | SkillDefinition documents | skill_id + version |

These collections are managed exclusively by agent.workflows. No other module writes to them.

---

## 8. Boundary Rules

agent.workflows enforces these rules:

- No execution logic — definitions describe structure, not behavior
- No import of agent.core runtime components (AgentRuntime, MCPClient, GraphState, etc.)
- No import of agent.api or agent.web
- No direct MCP tool calls — tool_ids in definitions are references, not invocations
- No canonical contract definitions — consumed from agent.core, never duplicated
- Definition schemas (WorkflowDefinition, SkillDefinition, StepDefinition) are owned here and are distinct from canonical contracts (Run, Step, ToolInvocation, etc.)

### 8.1 Definition Schemas vs Canonical Contracts

This distinction is critical:

| Concept | Owned by | Purpose |
|---------|----------|---------|
| WorkflowDefinition | agent.workflows | Declares what a workflow is (template) |
| StepDefinition | agent.workflows | Declares what a step should do (template) |
| SkillDefinition | agent.workflows | Declares how the agent should behave (template) |
| Run | agent.core | Records an actual execution instance |
| Step | agent.core | Records an actual step execution |
| ToolInvocation | agent.core | Records an actual tool call |

Definitions are templates. Canonical contracts are runtime records. They are related but not interchangeable.

---

## 9. Extension Points

**To add a new workflow:**

1. Create a YAML file in the workflows directory
2. Define steps using existing step types (AUTOMATED, AGENT, HUMAN_GATE)
3. Reference existing skills by skill_id
4. Restart the application (or implement hot-reload — out of scope for v1)

**To add a new skill:**

1. Create a YAML file in the skills directory
2. Define the system_prompt, output_format, and tool_restrictions
3. Restart the application

**To add a new step type:**

This requires changes in agent.core (new graph node) and agent.workflows (new StepType enum value, new validation rules in StepDefinition). See agent.core Section 21 for the full procedure.

---

## 10. Implementation Notes for Claude Code

- Load definitions after agent.core has built the tool catalog — skill tool_restrictions validation requires the catalog
- Use Pydantic's JSON Schema validation for input_schema and output_schema fields
- YAML parsing uses `yaml.safe_load` — no custom constructors
- File discovery is non-recursive: only top-level .yaml files in the configured directories are loaded
- Definition validation errors are collected and logged, not raised — partial loading is acceptable, but agent.core's startup validation (step 5) will catch any missing references
- The registry must support concurrent reads from multiple agent.core execution contexts
- Registry writes only happen at startup (from the loader) — the registry is effectively read-only during normal operation
- `WorkflowSummary` and `SkillSummary` are lightweight projections for list endpoints — they exclude the full step list and system_prompt respectively

---

## 11. Example Execution Trace

This trace shows how agent.workflows participates in a workflow run.

```
1. Startup
   └── DefinitionLoader.load_all()
       ├── Reads classify-ttp.yaml from workflows_dir
       │   └── Validates → WorkflowDefinition (workflow_id: classify-ttp, version: 1.0.0)
       ├── Reads attck-analyst.yaml from skills_dir
       │   └── Validates → SkillDefinition (skill_id: attck-analyst, version: 1.0.0)
       ├── Reads behavior-normalizer.yaml from skills_dir
       │   └── Validates → SkillDefinition (skill_id: behavior-normalizer, version: 1.0.0)
       └── Registers all into MongoDB via Registry

2. agent.core startup validation
   └── Registry.validate_references(tool_catalog)
       ├── classify-ttp step "normalize-input" references skill "behavior-normalizer" → found
       ├── classify-ttp step "classify" references skill "attck-analyst" → found
       ├── attck-analyst tool_restriction "servicenow.create_ticket" → found in catalog
       └── Returns: no warnings

3. Execution request arrives: POST /runs { workflow_id: "classify-ttp", input: { description: "..." } }
   └── agent.core DispatchEngine
       └── Registry.resolve_workflow("classify-ttp")
           └── Returns WorkflowDefinition (version: 1.0.0)
       └── agent.core records workflow_version: "1.0.0" on Run entity
       └── agent.core constructs GraphState with steps from the definition
       └── Execution proceeds per agent.core spec
```