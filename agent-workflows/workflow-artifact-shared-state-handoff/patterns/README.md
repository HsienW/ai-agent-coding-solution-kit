# Pattern mapping: multi-agent shared state and handoff

Artifact-based Shared State and Structured Handoff are not a new category of architecture problem. They combine several established patterns to handle shared working state, ownership transfer, long-running routing, failure loops, persistence, artifact references, history, snapshots, query efficiency, human approval, and irreversible operations.

The practical question is not "which single GoF pattern is this?" A better question is what each pattern is responsible for inside the workflow.

## Summary table

| Multi-agent component | Pattern | Problem addressed |
|---|---|---|
| Shared State | Blackboard | Multiple specialists work from the same problem space |
| State Store Interface | Repository | Isolate the domain from JSON, databases, or checkpointers |
| Handoff Envelope | Command Message / Envelope Wrapper | Package work transfer into an explicit message |
| Artifact Reference | Claim Check | Pass references instead of large payloads |
| Workflow Routing | State Machine | Define legal phases and transitions |
| Planner -> Coder -> Reviewer | Pipes and Filters | Each stage consumes artifacts and produces new artifacts |
| Orchestrator | Process Manager | Choose the next step from intermediate results |
| Complete History | Event Sourcing | Preserve the full history of state changes |
| Current State | Snapshot / Materialized View | Provide a fast current operational view |
| Retry-safe Consumer | Idempotent Receiver | Avoid duplicate side effects during retry |
| Human Approval | Approval Gate / Interrupt-Resume | Pause before high-risk actions |

## Blackboard

Multiple knowledge sources do not need to know each other directly. They read the current state from a shared blackboard and write new facts back through controlled paths.

```text
Planner ───────┐
Implementer ───┼── Shared Blackboard
Reviewer ──────┘
```

In a coding workflow:

- The Planner writes approved scope and acceptance criteria.
- The Implementer writes implementation results and evidence.
- The Reviewer writes findings and verdicts.
- The Coordinator uses the shared state for arbitration and readiness checks.

The value is lower coupling between agents, replaceable roles, and shared state that both people and code can inspect. The risk is an unstructured shared dump, so Blackboard needs schema, ownership, a state machine, and structured handoff.

## Repository

A Repository interface keeps workflow logic away from storage details:

```ts
interface WorkflowStateRepository {
  getCurrentState(changeId: string): Promise<CurrentState>;
  saveArtifact(artifact: AgentArtifact): Promise<ArtifactRef>;
  updateState(expectedVersion: number, next: CurrentState): Promise<void>;
}
```

The first implementation can use local JSON files. Later implementations can move to SQLite, PostgreSQL, Redis, object storage metadata, or a LangGraph checkpointer without forcing agent and routing logic to learn each backend.

The main caution is consistency. Different stores do not provide the same transaction, locking, and recovery guarantees.

## Command Message and Envelope Wrapper

Structured Handoff is close to a Command Message:

> Ask a specific role to perform a specific stage, using specific inputs, and produce a specific result.

The envelope carries routing metadata, correlation ID, schema version, security context, required references, and expected output. The actual work payload stays in referenced artifacts.

This makes handoff validation possible and keeps adapters simpler. The envelope should not become another copy of Current State, and it should not embed full artifacts.

## Claim Check

Claim Check stores large or sensitive payloads in a dedicated store and passes only controlled references through workflow messages.

```text
Handoff
  └─ diffRef: runtime://evidence/git-diff.patch
```

This is better than embedding a large patch directly into the handoff message. It reduces token cost, allows different permissions and retention policies for artifacts, and makes checksum validation easier.

The risks are broken references, path traversal, mismatched versions, and wrong artifact permissions.

## State Machine

State Machine gives Structured Handoff its operating rules.

```text
READY_FOR_REVIEW
    ↓ review.requested
REVIEWING
    ├─ approve -> READY_FOR_READINESS_CHECK
    ├─ changes -> CHANGES_REQUESTED
    └─ conflict -> NEEDS_COORDINATOR_ARBITRATION
```

It rejects illegal transitions, makes the flow testable, and keeps routing out of free-form model output. Keep the state set small. Only make a state explicit when it affects routing, permissions, or recovery.

## Pipes and Filters

Planning, implementation, and review can be treated as filters:

```text
Requirement
-> Plan Filter
-> Implementation Filter
-> Review Filter
-> Readiness Filter
```

Each filter accepts explicit input, performs a specialist transformation, and produces an artifact. Real coding workflows are rarely purely linear, so Pipes and Filters should be paired with a State Machine and Process Manager.

## Process Manager

The Orchestrator acts as a Process Manager. It maintains Current State, calls the next agent, routes based on results, manages retries, and pauses at human gates.

This supports non-linear workflows and centralized safety controls. It can also become a bottleneck or a God Object. Keep domain judgment inside agents and keep the Orchestrator focused on flow control, validation, storage, policy, and routing.

## Event Sourcing

Event Sourcing stores every state change as an append-only event:

```text
PlanApproved
ImplementationCompleted
ReviewRequested
ChangesRequested
ReviewApproved
HumanCommitApproved
```

It gives strong auditability and allows state reconstruction, but it adds event versioning, replay cost, ordering concerns, idempotency, snapshots, and migration work. Most first versions can start with Current State, latest artifacts, and important approval records.

## Snapshot / Materialized View

Current State is the operational snapshot of the full history:

```text
Full Events / Artifacts
          ↓ projection
Current State
```

Agents normally read the snapshot, not the entire history. This reduces read cost, identifies the latest accepted artifact, and improves recovery speed. Use state versions, atomic updates, checksums, and reconstruction paths to reduce stale state risk.

## Idempotent Receiver

Agents and CLI tools may be retried after timeouts. Processing the same result twice must not duplicate state updates, artifacts, commits, or deploys.

Useful keys include:

- `runId`
- `handoffId`
- `artifactId`
- `idempotencyKey`
- Expected State Version

## Pattern combination

The full architecture can be described as:

```text
                         ┌──────────────────┐
                         │ Durable Knowledge│
                         └────────┬─────────┘
                                  │
                                  ▼
┌───────────┐   Command/Handoff  ┌────────────────┐
│   Agent   │ ◄────────────────► │ Process Manager│
└─────┬─────┘                    └───────┬────────┘
      │                                  │
      │ Artifact                         │ State Transition
      ▼                                  ▼
┌────────────────────────────────────────────┐
│ Blackboard / Shared State                  │
│ Current Snapshot + Artifact References     │
└───────────────────┬────────────────────────┘
                    │ Repository
                    ▼
       ┌────────────────────────────┐
       │ Runtime / Artifact Storage │
       └────────────────────────────┘
```

The minimum useful set is:

```text
Blackboard-like Current State
+ Structured Command/Handoff
+ Explicit State Machine
+ Local Repository Adapter
+ Human Approval Gate
```

Add Process Manager automation, Event Sourcing, queues, distributed locks, or a full trace platform only when the workflow actually needs them.

## References

- Martin Fowler: Repository  
  <https://martinfowler.com/eaaCatalog/repository.html>
- Enterprise Integration Patterns: Process Manager  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html>
- Enterprise Integration Patterns: Message / Pipes and Filters / Claim Check  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/Message.html>  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/PipesAndFilters.html>  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/StoreInLibrary.html>
- Azure Architecture Center: Event Sourcing / Materialized View / Claim Check  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing>  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view>  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check>
