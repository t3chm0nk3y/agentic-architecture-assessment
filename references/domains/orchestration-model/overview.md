# Domain: Orchestration Model

## Domain Name

Orchestration Model

## What This Domain Governs

How agent execution is structured, controlled, and bounded. This domain assesses whether the system has a deliberate orchestration approach, whether state transitions are explicit and recorded, whether autonomous execution has guardrails, whether workflows are deterministic, and whether enough state is persisted to audit any run after the fact.

## Why It Matters

An agentic system without deliberate orchestration is a system that cannot be reasoned about. When execution flow is ad-hoc, state transitions are implicit, and autonomous behavior is unbounded, the system becomes unpredictable — operators cannot determine what happened, why it happened, or what will happen next. In the worst case, unbounded autonomous execution consumes unlimited resources, produces cascading failures, or takes actions that cannot be undone.

Orchestration governs the contract between the system and its operators. Explicit state transitions mean the system's behavior is auditable. Bounded execution means the system cannot run indefinitely. Deterministic workflow structure means operators can predict what steps will execute before they execute. Run state auditability means any execution can be reconstructed after the fact.

## Principles in This Domain

| ID | Title | Severity |
|----|-------|----------|
| P7.1 | Deliberate Orchestration Framework | MEDIUM |
| P7.2 | Explicit State Transitions | HIGH |
| P7.3 | Bounded Autonomous Execution | CRITICAL |
| P7.4 | Deterministic Workflow Structure | HIGH |
| P7.5 | Run State Auditability | HIGH |

## Common Failure Modes

- **Ad-hoc orchestration** — execution flow determined by scattered if/else chains without a coherent control structure, making behavior unpredictable and unauditable
- **Implicit state transitions** — run/step states change as side effects of other operations rather than through explicit, recorded transitions
- **Unbounded autonomous execution** — agent loops that iterate without maximum limits on iterations, tool calls, token spend, or wall-clock time
- **Non-deterministic workflows** — step execution order varies unpredictably based on runtime conditions without clear rules, making it impossible to reason about what a run will do
- **Ephemeral state** — run execution state exists only in memory during execution and cannot be inspected or reconstructed after the fact

## Typical Repository Indicators

Look for these when forming the system hypothesis:

- Orchestration framework or engine code (state machines, workflow engines, step runners, agent loops)
- State model definitions (run states, step states, transition rules)
- Execution bounds configuration (max iterations, max tool calls, max tokens, timeouts)
- Workflow definitions (step sequences, DAGs, pipeline configurations)
- State persistence (run records, step records, state snapshots, execution logs)
- Agent loop implementations (while loops with LLM calls, tool dispatch, and termination conditions)

## Common Trace Types to Inspect

- **Orchestration flow trace** — follow an execution from initiation through step dispatch to termination, mapping the control flow
- **State transition trace** — follow a run or step through its lifecycle states, checking that each transition is explicit and recorded
- **Execution bound trace** — find the agent loop and verify that iteration limits, tool call limits, and token limits are enforced and tested
- **Audit reconstruction trace** — given a completed run, determine whether enough information was persisted to reconstruct the execution sequence

## Relationship to Adjacent Domains

- **Error Handling (Domain 6):** P6.3 (Failure Propagation) feeds into P7.2 (Explicit State Transitions) — failures must propagate to trigger error state transitions. P6.5 (Retry Safety) interacts with P7.3 (Bounded Execution) — unbounded retries can violate execution bounds.
- **Observability (Domain 3):** P3.1 (Run Traceability) provides the correlation identifier that P7.5 (Run State Auditability) depends on. P3.3 (Structured Events) provides the event stream that makes state transitions observable.
- **Execution Boundaries (Domain 1):** P1.x execution boundary principles define what the system is; P7.x orchestration principles define how the system controls its execution.

## Common False Positives in Assessment

- **Presence of a state enum does not mean explicit state transitions** — check that transitions are recorded and enforced, not just that states are defined
- **A while loop is not necessarily an orchestration framework** — check whether the loop has deliberate structure, bounds, and state management
- **Configuration for max_iterations does not mean bounds are enforced** — check that the configuration is read and enforced in the execution path, not just defined
- **Database tables for runs do not mean auditability** — check what is actually persisted (just a final status? or full execution state including step sequence and intermediate results?)
- **Step functions or named functions do not mean deterministic workflows** — check whether the execution order is known before execution or determined dynamically
