# 02 | Reference architecture for Artifact-based Shared State + Structured Handoff

[English](./02-reference-architecture.md) | [繁體中文](./02-reference-architecture-zh-TW.md)

## 1. Clarify the division of responsibility

Artifact-based Shared State and Structured Handoff solve different problems:

| Mechanism | Core question | Architecture role |
|---|---|---|
| Artifact-based Shared State | Which facts does the system currently trust? | Data Plane |
| Structured Handoff | Who should do the work, what should they read, and what should they output? | Control Plane |

With only Shared State, an agent may know "what exists now" but not "who owns the next step."

With only Handoff, an agent may know "who receives the work" but lack reliable input and version grounding.

The complete design is:

> Artifact-based Shared State with Structured Handoff

## 2. Do not mix four kinds of data

Before implementation, separate four concepts that are easy to confuse.

## 2.1 Durable Knowledge

Long-lived engineering knowledge that future maintainers should understand, such as:

- Requirements.
- Design.
- Acceptance Criteria.
- Task Plan.
- ADR / Decision.
- Final Execution Summary.

It usually goes into Git and follows the project lifecycle.

## 2.2 Runtime State

The current snapshot of this run, such as:

- Current phase.
- Current owner.
- Attempt.
- Gate status.
- Latest artifact refs.
- Next action.

It is operational state, not permanent documentation.

## 2.3 Artifact

A concrete deliverable produced by an agent at a stage, such as:

- Coordinator Result.
- Implementation Result.
- Review Result.
- Test Evidence.
- Diff Patch.
- Readiness Result.

The A2A Protocol also separates Message from Artifact: a Message is communication, while an Artifact is a concrete deliverable created by task execution.

## 2.4 Trace

Complete execution information used for debugging, audit, and observability, such as:

- Model request and response metadata.
- Token, latency, retry.
- Tool calls.
- stdout / stderr.
- Detailed event sequence.

Trace should not become normal agent input for every round, and it should not all be committed to Git.

## 3. Reference role model

The role names below stay model-neutral and tool-neutral.

| Role | Main responsibility | Write permission |
|---|---|---|
| Coordinator / Planner | Create and arbitrate specs; decide whether the flow can enter the next stage | Specs and coordination result |
| Implementer | Change code based on approved specs and produce validation evidence | Code and Implementation Result |
| Reviewer | Read-only review of specs, diff, evidence, and risk | Review Result only |
| Orchestrator | Validate contracts, update state, apply transitions, pause / resume | Runtime State |
| Human Approver | Approve irreversible or high-risk operations | Approval Decision |

In a first version without an orchestrator, a human can still start each role manually while using the same contracts.

## 4. Current State: shared state snapshot

Current State represents "how the system should be understood now." It does not store full history.

Example:

```json
{
  "schemaVersion": "1.0",
  "changeId": "feature-example",
  "runId": "run-20260630-001",
  "phase": "READY_FOR_REVIEW",
  "currentOwner": "reviewer",
  "attempt": 1,
  "latestArtifacts": {
    "plan": "repo://specs/feature-example/design.md",
    "implementation": "runtime://artifacts/implementation-result.json",
    "diff": "runtime://evidence/git-diff.patch",
    "validation": "runtime://evidence/validation-summary.json"
  },
  "gates": {
    "planApproved": true,
    "implementationCompleted": true,
    "validationPassed": true,
    "reviewApproved": false,
    "humanCommitApproved": false
  },
  "blockers": [],
  "nextAction": "REQUEST_REVIEW"
}
```

### What Current State should store

- Facts that must persist across nodes.
- References to the latest accepted artifacts.
- Gates needed for legal routing.
- Identifiers needed for retry and recovery.

### What Current State should not store

- Full chat history.
- Private agent reasoning.
- Full text of every historical version.
- Long formatted descriptions that can be derived from artifacts.
- Secrets or credentials.

LangGraph gives similar guidance: put data that must persist across steps into State, and prefer raw reusable information over one-off formatted output.

## 5. Agent Result Envelope: unified delivery contract

Different roles can have different payloads, but their shared fields should be consistent.

```json
{
  "schemaVersion": "1.0",
  "artifactId": "artifact-implementation-001",
  "changeId": "feature-example",
  "runId": "run-20260630-001",
  "producer": "implementer",
  "stage": "apply-change",
  "kind": "implementation_result",
  "status": "completed",
  "summary": "Implemented approved scope and executed validation.",
  "inputRefs": [],
  "outputRefs": [],
  "verification": [],
  "risks": [],
  "nextHandoff": {}
}
```

Use a shared envelope plus a discriminated union:

```text
kind = coordinator_result
kind = implementation_result
kind = review_result
kind = readiness_result
kind = archive_result
```

This gives:

- A shared parsing path.
- Clear role-specific fields.
- Less schema duplication.
- Easier versioning and role extension.

## 6. Structured Handoff: the next-step contract

Handoff passes references and a work contract. It does not copy full artifacts.

```json
{
  "schemaVersion": "1.0",
  "handoffId": "handoff-003",
  "changeId": "feature-example",
  "runId": "run-20260630-001",
  "from": "implementer",
  "to": "reviewer",
  "stage": "review-result",
  "reason": "implementation_completed",
  "requiredInputRefs": [
    "repo://specs/feature-example/design.md",
    "runtime://artifacts/implementation-result.json",
    "runtime://evidence/git-diff.patch",
    "runtime://evidence/validation-summary.json"
  ],
  "expectedOutputKind": "review_result",
  "acceptanceCriteriaRefs": [
    "repo://specs/feature-example/requirements.md#acceptance"
  ],
  "onSuccess": "READY_FOR_READINESS_CHECK",
  "onFailure": "CHANGES_REQUESTED",
  "onConflict": "NEEDS_COORDINATOR_ARBITRATION",
  "requiresHumanApproval": false
}
```

### Minimum responsibilities of Handoff

1. Identify the sender and receiver.
2. Describe why the handoff exists.
3. List required inputs.
4. Define the expected output kind.
5. Provide success, failure, and conflict routes.
6. Mark whether human approval is required.

### Handoff should not

- Store the full diff.
- Store the full spec.
- Store all review findings.
- Replace Current State.
- Directly authorize irreversible operations.

## 7. Artifact Reference: pass references, not full text

References can use URI-like formats:

```text
repo://specs/feature/design.md
runtime://artifacts/review-result.json
runtime://evidence/git-diff.patch
ci://runs/12345/test-report
object://agent-artifacts/run-001/diff.patch
```

Each reference should ideally include:

- `artifactId`
- `version`
- `mediaType`
- `sha256` or another content hash
- `createdAt`
- `producer`
- `scope`

This follows the same spirit as Claim Check: large content stays in a dedicated store, while workflow messages carry controlled references.

## 8. State Machine: who can move forward under which conditions

The first version should use explicit states instead of letting the LLM invent the next phase name.

```text
PLAN_DRAFT
    ↓
PLAN_REVIEW
    ├─ approved ───────────────→ PLAN_APPROVED
    └─ changes requested ──────→ PLAN_DRAFT

PLAN_APPROVED
    ↓
READY_FOR_IMPLEMENTATION
    ↓
IMPLEMENTING
    ├─ completed ──────────────→ READY_FOR_REVIEW
    ├─ design conflict ────────→ NEEDS_COORDINATOR_ARBITRATION
    └─ failed ─────────────────→ FAILED

READY_FOR_REVIEW
    ↓
REVIEWING
    ├─ approved ───────────────→ READY_FOR_READINESS_CHECK
    ├─ changes requested ──────→ CHANGES_REQUESTED
    ├─ incomplete evidence ────→ INCOMPLETE
    └─ design conflict ────────→ NEEDS_COORDINATOR_ARBITRATION

CHANGES_REQUESTED
    ↓
IMPLEMENTING

READY_FOR_READINESS_CHECK
    ├─ ready ──────────────────→ READY_FOR_ARCHIVE
    └─ not ready ──────────────→ IMPLEMENTING / REVIEWING

READY_FOR_ARCHIVE
    ↓
ARCHIVED_AWAITING_HUMAN_COMMIT
    ├─ human approved ─────────→ COMPLETED
    └─ human rejected ─────────→ READY_FOR_ARCHIVE
```

### Invariants that should be explicit

- Failed tests cannot enter READY_FOR_ARCHIVE.
- Unresolved Blocker / Major findings cannot be marked Review Approved.
- The Reviewer cannot modify requirements.
- The Implementer cannot treat scope deviation as approved.
- Invalid schema cannot advance state.
- Commit, push, deploy, and other irreversible operations cannot be triggered directly by a normal agent verdict.

## 9. Who can update Current State?

This ownership issue is easy to miss.

### First version without an orchestrator

The Coordinator or human host can update state:

```text
Agent outputs Result
-> Host validates Result
-> Host saves Artifact
-> Host updates Current State
```

Even when the Reviewer outputs `APPROVE`, it does not directly modify the state file.

### Version with an orchestrator

Only the Orchestrator can:

- Validate Result Schema.
- Save Artifact.
- Apply Transition.
- Update Current State.

Agents can only propose:

```text
recommendedTransition
nextHandoff
```

This avoids making the agent both the decision maker and the state database manager.

## 10. Read-only Reviewer design

Reviewer reliability comes from complete, traceable evidence, not from broader write permissions.

The Reviewer should read:

- Approved spec.
- Implementation Result.
- Changed Files.
- Git Diff.
- Validation Summary.
- Skipped items.
- Directly relevant source code.

The Reviewer outputs:

- Verdict.
- Findings.
- Residual Risks.
- Evidence Assessment.
- Suggested Transition.

The Reviewer should not:

- Fix code.
- Check off tasks directly.
- Approve commits directly.
- Run unauthorized commands to fill missing evidence.

If evidence is insufficient, the Reviewer should output `INCOMPLETE`, not guess `APPROVE`.

## 11. Relationship with SDD, OpenSpec, and ADR

Specification documents are already important Shared Knowledge, but they usually cover only the Planning Layer.

```text
proposal / requirements
= why to do it, what to achieve

design
= how it is expected to be done

tasks
= how construction is split

current-state
= where the workflow is now

implementation-result
= what was actually done

review-result
= what review found

execution-summary
= what long-term facts remain
```

Runtime State should not replace SDD. It should reference stable specifications.

## 12. Tradeoff between code-led and LLM-led orchestration

OpenAI Agents SDK documentation distinguishes LLM-led and code-led multi-agent orchestration. Coding workflows usually benefit from a hybrid approach.

### LLMs are good at

- Specification analysis.
- Code implementation.
- Semantic review.
- Finding classification.
- Risk explanation.

### Code is good at

- Schema validation.
- State transitions.
- Permission checks.
- Retry limits.
- Artifact persistence.
- Gate decisions.
- HITL pause / resume.

The reason is direct:

> Professional judgment needs model flexibility; workflow safety needs determinism.

## 13. Minimum reference flow

```text
1. Coordinator produces coordinator_result
2. Host validates and saves Artifact
3. Host updates current-state
4. Host creates Coordinator -> Implementer Handoff
5. Implementer reads the referenced minimum context
6. Implementer produces implementation_result + evidence refs
7. Host updates State and creates Implementer -> Reviewer Handoff
8. Reviewer produces review_result
9. Host routes based on Verdict:
   - APPROVE -> Readiness
   - REQUEST_CHANGES -> Implementer
   - DESIGN_CONFLICT -> Coordinator
   - INCOMPLETE -> add Evidence / manual handling
10. After Readiness passes, pause at Human Commit Gate
```

Even if a human starts every step manually, this flow already removes most context transfer.

## References

- LangGraph: State and Persistence  
  <https://docs.langchain.com/oss/javascript/langgraph/thinking-in-langgraph>  
  <https://docs.langchain.com/oss/javascript/langgraph/persistence>
- OpenAI Agents SDK: Handoffs and Agent Orchestration  
  <https://openai.github.io/openai-agents-js/guides/handoffs/>  
  <https://openai.github.io/openai-agents-js/guides/multi-agent/>
- A2A Protocol: Artifacts and Task Lifecycle  
  <https://a2a-protocol.org/latest/topics/key-concepts/>  
  <https://a2a-protocol.org/latest/topics/life-of-a-task/>
- Azure Architecture Center: Claim Check  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check>
