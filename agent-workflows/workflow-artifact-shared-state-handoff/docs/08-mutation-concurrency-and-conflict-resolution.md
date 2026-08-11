# 08 | Mutation, concurrency, and conflict resolution

[English](./08-mutation-concurrency-and-conflict-resolution.md) | [繁體中文](./08-mutation-concurrency-and-conflict-resolution-zh-TW.md)

This document defines how Coordinators, Orchestrators, Agent Adapters, Runtime Stores, and controlled Executors submit Mutations, handle concurrent writes, and recover from conflicts in State, Source, Artifacts, or external side effects. Every Mutation MUST bind an Actor, Capability, Target, Base Identity, and Idempotency Identity. The system accepts a write only while its prerequisite version, Gate, and resource state still match the request.

This document uses the following normative terms:

- "MUST" defines a requirement that the workflow cannot omit.
- "SHOULD" defines the default practice; a deviation requires a recorded reason.
- "MAY" defines an optional capability that a team can adopt according to its scale and risk.

## 1. Scope

[`06-context-authority-and-retrieval-policy.md`](./06-context-authority-and-retrieval-policy.md) pins the State Revision, Candidate, and Accepted Artifact that an Agent reads. [`07-role-capability-and-scope-control.md`](./07-role-capability-and-scope-control.md) determines whether an Actor may perform an Action on a specified Resource. This document covers the writes that follow:

- How should the system reject a stale write when another Writer updates Current State after it was read?
- In what order should the workflow publish an Artifact, Handoff, and State?
- How should multiple Implementers isolate Source Mutations?
- Does a clean Git merge still require fresh Validation and Review?
- Which races do Locks, Leases, CAS, and Fencing Tokens address?
- How can a Retry avoid duplicating a Commit, Push, or another external side effect?
- Which Conflicts can automation resolve, and which ones require a Coordinator or Human?

This document covers State CAS, Atomic Replace, Crash Consistency, Artifact Immutability, Worktree Integration, Idempotency, Locks and Leases, Conflict Classification, and Recovery Routing. Other documents cover these subjects:

- For the base structures of Current State, Agent Result, Handoff, and Artifact Reference, see [`02-reference-architecture.md`](./02-reference-architecture.md).
- For Runtime Storage, Retention, and the criteria for adopting Event Sourcing, see [`03-runtime-storage-and-retention.md`](./03-runtime-storage-and-retention.md).
- For Concurrency Races, Artifact Tampering, and Single Writer security risks, see [`05-security-and-maintainability.md`](./05-security-and-maintainability.md).
- For Context Version, Checksum, and Candidate versus Accepted selection, see [`06-context-authority-and-retrieval-policy.md`](./06-context-authority-and-retrieval-policy.md).
- For Role, Capability, Path, and Tool Scope, see [`07-role-capability-and-scope-control.md`](./07-role-capability-and-scope-control.md).

This document does not define Production Deployment, Release Promotion, Database Transactions, or business compensation workflows. It does not change the existing JSON Schemas.

## 2. Mutation types and write ownership

A Mutation is any operation that changes a Repository, Runtime Store, Artifact Registry, or external System of Record. Each Target requires an appropriate isolation and commit method.

| Mutation type | Target | Default Writer | Primary controls |
|---|---|---|---|
| Specification Mutation | Requirement, Proposal, Design, Tasks, ADR | Coordinator / controlled Human | Change Scope, Single Writer, Git Identity |
| Source Mutation | Source, Test, Config, Generated File | Implementer | Worktree, Base Commit, Path Ownership |
| Artifact Publication | Agent Result, Evidence, Handoff | Producer / Orchestrator | Unique URI, Create-only, Checksum, Immutable |
| Runtime State Mutation | Current State, Gate, Accepted Pointer | Orchestrator / controlled Automation | Expected Revision, CAS, Atomic Replace |
| External Side Effect | Commit, Push, Issue Update, Deploy, Delete | controlled Executor | Human Gate, Idempotency, Before / After Reference |

Reviewers, Readiness workers, and Archivists MAY create their own Stage Outputs. They MUST NOT modify Source, upstream Artifacts, Gates, or Current State. A Producer creates a Candidate. Only the Orchestrator may update the Accepted Pointer after validation.

### 2.1 Mutation Identity

A Mutation conceptually contains at least:

```text
mutationId
idempotencyKey
actorId
changeId
runId
stage
capability
resource
baseRevision / baseCommit
payloadHash
```

These fields establish who intends to change which resource, the version on which the change depends, and whether the system has already executed it. The current Schemas do not define a complete Mutation Envelope. In the first local implementation, Orchestrator Invocation Metadata and Trace MUST retain this information. Writers MUST NOT insert it into Current State, Agent Result, or Handoff because those Schemas use `additionalProperties: false`.

### 2.2 Prepare, Publish, and Accept

The workflow MUST keep these actions separate:

| Action | Description | Actor |
|---|---|---|
| Prepare | Create a Source Change, Candidate Artifact, or next State version in isolation | Agent / Orchestrator |
| Publish | Place a complete, minimally validated Candidate at a unique URI | Producer Host / Orchestrator |
| Accept | Update the Current State Pointer after required Gates pass | Orchestrator |

Publication does not grant Accepted status. Before Accept, the Orchestrator MUST recheck State Revision, Gates, Artifact Binding, and Capability.

## 3. Concurrency invariants

Every implementation MUST preserve these invariants:

1. Each Change has only one Current State Writer at a time.
2. A State Lock is not held during an Agent Invocation, test, Build, or Review.
3. A Lock covers only the short Critical Section used to load the latest State, validate, write, and replace it.
4. Current State does not allow partial in-place modification.
5. Every Artifact referenced by State has been published completely and has passed Schema and Integrity checks.
6. Every Artifact uses a unique URI and remains Immutable after publication.
7. A Producer cannot update the Accepted Pointer for its own output.
8. A failed State CAS cannot overwrite a newer Revision.
9. `revision`, `attempt`, and Artifact Version each have one defined purpose.
10. A merged result becomes a new Candidate. It cannot inherit Validation or Review without rerunning them.
11. Human Approval binds a fixed Diff or Payload. A content change requires reevaluation.
12. When the outcome of an external side effect is unknown, the Executor queries the System of Record before retrying.

Current State, Gates, Accepted Pointers, Permissions, and Approvals MUST NOT use last-write-wins. Trace, Metrics, or rebuildable temporary caches MAY use another strategy when their data policy allows it.

## 4. Current State Revision and CAS

### 4.1 Meaning of `revision`

[`current-state.schema.json`](../schemas/current-state.schema.json) already requires an integer `revision`. This document defines it as the monotonically increasing version of a Current State Snapshot:

- The first valid State has a `revision` of at least 1.
- Each successful logical State Mutation increments `revision` by 1.
- A CAS rejection, Schema validation failure, or Atomic Replace failure does not increment it.
- A Writer cannot reserve future Revisions by skipping numbers.
- Restore and manual repair create a new Revision. They cannot move the value backward.

`attempt` counts Agent work attempts. Retrying an Agent MAY increment `attempt`, while only a successful Current State update increments `revision`.

### 4.2 Expected Revision

A Writer MUST submit the Revision it read:

```text
expectedRevision = 5
proposedRevision = 6
```

If Current State has reached Revision 6 when the write arrives, the Store MUST return `STATE_REVISION_CONFLICT` and preserve the current content. The Writer then reloads State, Handoff, and required Artifacts before deciding whether the original Mutation still applies.

### 4.3 CAS flow

The conceptual flow is:

```text
applyTransition(changeId, expectedRevision, proposedTransition):
  acquireShortStateLock(changeId)

  current = loadAndValidateState(changeId)

  if current.revision != expectedRevision:
    releaseStateLock()
    return STATE_REVISION_CONFLICT

  validateTransition(current, proposedTransition)
  validateGates(current, proposedTransition)
  validateArtifactRefs(proposedTransition)

  next = buildNextState(current, proposedTransition)
  next.revision = current.revision + 1

  atomicReplace(current-state.json, next)
  releaseStateLock()

  return committed(next.revision)
```

The implementation MUST release the Lock through `finally` or an equivalent mechanism. A timeout, Process Crash, or Validation Error cannot leave the Lock permanently occupied.

### 4.4 Conditions for recomputation

After a CAS failure, the Orchestrator MAY perform a bounded retry only when every condition remains true:

- The same Change still owns the new State.
- Phase, Owner, and required Gates preserve the premise of the original Mutation.
- Target Artifact, Diff, and Approval Binding remain identical.
- The Retry cannot duplicate a completed external side effect.
- The retry count remains below the Policy limit.

If any semantic premise has changed, the workflow stops and creates a Conflict Result. The Writer MUST NOT replace `expectedRevision` with the latest number and resubmit without reevaluating the Mutation.

## 5. Local Atomic Write and Crash Consistency

### 5.1 State write

Local Current State SHOULD use complete Snapshot replacement:

```text
1. Read and validate current-state.json
2. Create a temporary file on the same Filesystem
3. Write the complete next State
4. Validate JSON and the Current State Schema
5. Flush required data
6. Atomically replace current-state.json
7. Record the Mutation Receipt / Trace
```

The temporary file MUST reside on the same Filesystem or Volume as the target. Otherwise, Rename may degrade to Copy across devices. If the platform cannot guarantee Atomic Replace, use a Durable Store with Transaction or CAS support.

### 5.2 Artifact and State commit order

The workflow MUST publish the Artifact before updating State:

```text
write temporary artifact
→ validate schema / checksum / target binding
→ publish immutable artifact URI
→ publish immutable handoff
→ CAS current-state.json
→ record mutation receipt
```

If CAS fails, the published Artifact remains an unreferenced Candidate that a Retention Job can remove. Current State remains valid.

The following order creates a Dangling Reference and MUST be prohibited:

```text
update current-state.json
→ write referenced artifact
```

If the Process stops between these steps, a downstream Agent reads an absent or incomplete Artifact.

### 5.3 Partial Write Recovery

At startup or recovery, the Orchestrator SHOULD check:

- Whether `current-state.json` passes its Schema.
- Whether a temporary State file remains.
- Whether every Artifact referenced by State exists and matches its Checksum.
- Whether a published Candidate is unreferenced by any State or Handoff.
- Whether the Mutation Receipt shows that the previous commit completed.

If the Orchestrator cannot prove that State is complete, the workflow SHOULD enter `FAILED` or retain a safe Phase with a `MUTATION_COMMIT_INCOMPLETE` Blocker. Recovery MUST NOT guess which temporary file is newer.

## 6. Immutable Candidates and the Accepted Pointer

### 6.1 Candidates use unique URIs

A Producer SHOULD assign every output a unique Artifact ID and URI, for example:

```text
artifact://feature-example/run-003/implementation-result-attempt-02.json
evidence://feature-example/run-003/validation-attempt-02.json
```

If the same URI already exists:

- When both Checksum and Idempotency Key match, the Host MAY return the existing Mutation Receipt.
- When the Checksum differs, the Host returns `ARTIFACT_ID_COLLISION`.
- The Producer MUST NOT overwrite, delete, or modify an existing Artifact to pass a Gate.

### 6.2 Compatible meaning of `latestArtifacts`

Current State uses `latestArtifacts`. Until the Schema adds a separate `acceptedArtifacts` field, the Orchestrator MUST interpret each Pointer as the latest available Artifact that has been accepted.

```text
accepted implementation = v1

candidate v2
→ validation failed
→ remains candidate

latestArtifacts.implementationResult
→ remains v1
```

A newer filename, creation time, or Attempt does not update the Pointer.

### 6.3 Accepted Pointer CAS

The Pointer update, Phase, Gate, Owner, and Handoff MUST form one State Mutation and use the same `expectedRevision`. If two Candidates request acceptance concurrently:

```text
v2 reads revision 8
v3 reads revision 8

v2 commits revision 9
v3 attempts expectedRevision 8
→ STATE_REVISION_CONFLICT
→ accepted pointer remains v2
```

v3 MAY remain a Candidate while the Coordinator compares the outputs or creates another Review Handoff. The Producer cannot write v3 into `latestArtifacts`.

### 6.4 A merge creates a new Candidate

Merging two validated Candidates changes the Diff, Target Binding, and risk. The integrated result MUST receive a new Artifact ID, Diff Hash, Validation Evidence, and Review Result:

```text
candidate-A + candidate-B
          ↓
integrated-candidate-C
          ↓
combined validation
          ↓
review
          ↓
accepted pointer CAS
```

The Review Verdicts for A and B are integration inputs. They cannot approve C.

## 7. Source Mutation and Worktree Isolation

### 7.1 Single Writer default

The first local implementation SHOULD retain these boundaries:

- One Implementer modifies one worktree.
- A Reviewer has read-only access to Source.
- The Coordinator manages Scope and Path Ownership.
- The Orchestrator serializes Runtime State Mutations.
- A Human controls the Git Gate.

Even with one Writer, the workflow MUST retain a Base Commit or Source Snapshot Identity so that it can detect manual changes to the same worktree during an Agent run.

### 7.2 Parallel Implementers

When work runs in parallel, every Implementer MUST:

- Use a separate Worktree or Branch.
- Pin `baseCommit` or an equivalent Source Identity.
- Receive explicit Path Ownership.
- Modify only the approved Scope.
- Produce a separate Candidate, Changed Files list, Diff Hash, and Evidence.
- Avoid merging directly into the Integration Branch.

The Coordinator SHOULD assign Shared Generated Files, a Lockfile, Migration, Schema, and shared Config to one Integration Owner. These files can be affected by several Candidates even when they reside outside overlapping Feature Paths.

### 7.3 Base Divergence

When a Candidate is submitted, the Integration Host MUST compare:

```text
candidate.baseCommit
integration.currentBase
```

If the values differ, the Host returns `SOURCE_BASE_DIVERGED`. The workflow MAY create a controlled Rebase or Integration Handoff, but it cannot silently apply a stale Diff. After Rebase, the Candidate MUST receive a new Diff Hash and rerun relevant Validation.

## 8. Controlled Integration

### 8.1 Integration Owner

The Integration Owner MAY be controlled Automation, a designated Implementer, or a Human. It is responsible for:

- Validating the Base Identity of every Candidate.
- Checking Changed Files against Path Ownership.
- Applying candidate changes in an isolated Integration Worktree.
- Classifying textual, structural, and semantic conflicts.
- Producing an Integrated Candidate and Combined Evidence.

The Integration Owner cannot bypass the applicable Role Policy, Review Gate, or Human Gate.

### 8.2 Minimum conditions for automatic integration

The workflow MAY attempt mechanical integration only when every condition holds:

- Candidates use the same Base or explicitly compatible Bases.
- Changed Files have no unresolved Ownership overlap.
- Shared Generated Files have one Owner.
- No Requirement, Design, Schema, or API Contract Conflict exists.
- Integration Policy permits automatic handling for the file type.
- The integrated result will rerun Validation and Review.

A clean Git merge proves only that the text applied. Combined Behavior can still fail.

### 8.3 Source protection after a conflict

When Integration encounters a Conflict:

- Do not modify the original Candidate Artifact.
- Do not perform repeated, unbounded fixes in an Implementer's original Worktree.
- Preserve the Base, Candidate, Conflict Path, and Tool Output Reference.
- Create a new Integration or Coordinator Handoff.
- Keep the incomplete integrated result out of the Accepted Pointer.

When Conflict Resolution creates new content, it MUST create a new Candidate. It cannot rewrite a reviewed version in place.

## 9. Locks, Leases, and Fencing

### 9.1 Select controls for the execution environment

| Execution environment | Recommended controls | Boundary |
|---|---|---|
| Single local Process | In-process Mutex + Revision Check + Atomic Replace | One Orchestrator Process |
| Multiple Processes on one host | OS File Lock + Revision Check + Atomic Replace | Multiple Processes sharing a local Runtime |
| Shared Runtime across hosts | Durable Store CAS + Lease + Fencing Token | Network Store or distributed Workers |
| Parallel Source work | Worktree / Branch + Base Identity + Integration Owner | Multiple Implementers |

Locks, Leases, and Worktrees solve different problems. A State Lock serializes a short State Commit. A Worktree isolates a long-running Source Mutation. A Lease manages temporary ownership across Processes or hosts.

### 9.2 Lock scope

A State Lock may cover only:

```text
load current state
→ compare revision
→ validate proposed transition
→ atomic replace
→ release lock
```

The following work MUST NOT execute while holding the State Lock:

- LLM Invocation.
- Tool Retrieval.
- Test, Lint, or Build.
- Review.
- Network Call.
- Waiting for Human Approval.

A long-held Lock turns one Agent timeout into a blocked Change and makes Crash Recovery harder.

### 9.3 Lease Record

A Lease used across Processes or hosts conceptually contains at least:

```text
leaseId
ownerId
resource
acquiredAt
expiresAt
fencingToken
```

The Lease Holder MUST renew before expiration. If renewal fails, it stops writing immediately. A recovered old Process cannot reuse the previous Lease.

### 9.4 Fencing Token

Checking only Lease time can allow an old Writer to resume and write after a pause. The Store SHOULD issue a monotonically increasing Fencing Token whenever it grants a Lease:

```text
writer A gets token 41
writer A pauses
lease expires

writer B gets token 42
writer B commits

writer A resumes with token 41
→ FENCING_TOKEN_REJECTED
```

A clock-based Lock File cannot provide this guarantee by itself. A local implementation that uses only a Lock File MUST remain within one controlled host and combine the Lock with Revision Check. A multi-host deployment must use a Store that can validate the Token.

## 10. Idempotency and safe retry

### 10.1 Idempotency Key

Every Mutation that may be redelivered SHOULD have an Idempotency Key. The Key SHOULD bind:

```text
changeId
runId
stage
action
resource
baseRevision / baseCommit
payloadHash
```

The Key SHOULD NOT consist only of an Agent name or Timestamp. Two semantically different Mutations cannot share one Key.

### 10.2 Mutation Receipt

The Mutation Host SHOULD retain a minimum Receipt:

```text
mutationId
idempotencyKey
requestHash
decision
target
beforeRef
afterRef
committedRevision
sideEffectRef
createdAt
traceId
```

When the same Idempotency Key appears again:

- If the Request Hash matches and the previous Mutation succeeded, return the original Receipt.
- If the previous Mutation is still running, return its current status without starting another operation.
- If the Request Hash differs, return `IDEMPOTENCY_KEY_REUSED`.
- If the previous outcome is unknown, run Reconciliation first.

The Runtime Trace or Host Journal can hold the Receipt initially. The current Schemas have no matching field, so Writers MUST NOT embed a Receipt arbitrarily in Agent Result or Current State.

### 10.3 Retry classification

| Failure type | Automatic Retry | Conditions |
|---|---:|---|
| Temporary Lock Contention | MAY | Bounded attempts, Backoff, and no entry into the Critical Section |
| State Revision Conflict | Conditional | Semantic premises still match after reload |
| Network Read Timeout | MAY | The Read Operation is repeatable and has a Timeout |
| Validation Failure | MUST NOT | Create a correction Handoff first |
| Design / Semantic Conflict | MUST NOT | Return to the Coordinator |
| Approval Changed / Expired | MUST NOT | Create a new Approval Request |
| External Write Outcome Unknown | MUST NOT resend directly | Reconcile with the System of Record first |

Retry Policy MUST define a maximum attempt count, Backoff, and termination conditions. Unlimited retries waste resources and conceal State or Design Conflicts.

### 10.4 Unknown side-effect outcome

A Commit, Push, Issue Update, or external API Write may succeed remotely before the Client times out. The Executor MUST:

1. Query the System of Record with the Idempotency Key, Commit Hash, Request ID, or Target Version.
2. If the action succeeded, preserve the After Reference and return the original result.
3. Retry only after confirming that the action did not execute.
4. If the outcome remains unknown, return `SIDE_EFFECT_OUTCOME_UNKNOWN` and stop automation.

An Agent cannot assume that a side effect failed because it did not receive a success response.

## 11. Conflict classification and fixed codes

### 11.1 Conflict Matrix

| Code | Condition | Default handling |
|---|---|---|
| `STATE_REVISION_CONFLICT` | Current State no longer matches the Revision that the Writer read | Reload and rebuild the Mutation |
| `STATE_LOCK_UNAVAILABLE` | The Writer cannot acquire the short State Lock | Retry within a limit, then return `incomplete` |
| `LEASE_EXPIRED` | The Lease has expired | Stop writing and obtain a new Lease |
| `FENCING_TOKEN_REJECTED` | An old Writer submits an expired Token | Reject the write and preserve Trace |
| `MUTATION_DUPLICATE` | The same Mutation already succeeded | Return the existing Receipt |
| `IDEMPOTENCY_KEY_REUSED` | One Key maps to a different Request Hash | Return `blocked` for Host or Human inspection |
| `ARTIFACT_ID_COLLISION` | The same URI already contains different content | Isolate the new output and prohibit overwrite |
| `MUTATION_COMMIT_INCOMPLETE` | Crash, Partial Write, or State Integrity remains uncertain | Preserve the original State and start Recovery |
| `SOURCE_BASE_DIVERGED` | Candidate Base differs from Integration Base | Create a controlled Rebase or Integration Handoff |
| `PATH_OWNERSHIP_CONFLICT` | Multiple Writers modify the same Ownership scope | Coordinator reassigns work |
| `SOURCE_MERGE_CONFLICT` | Git cannot complete a textual merge | Integration Owner resolves it |
| `SEMANTIC_CONFLICT` | Text merges, but Requirements, Schema, API, or behavior conflict | `NEEDS_COORDINATOR_ARBITRATION` |
| `ACCEPTED_POINTER_CONFLICT` | Multiple Candidates request the Accepted Pointer concurrently | Keep the Pointer unchanged and compare Candidates again |
| `GATE_CHANGED` | Review, Approval, or State Gate has changed | Rerun Gate checks |
| `SIDE_EFFECT_OUTCOME_UNKNOWN` | The external write cannot be confirmed after Timeout | Reconcile and prohibit blind Retry |

### 11.2 Conflict Result

A Conflict Result or Trace SHOULD retain at least:

```text
code
changeId
runId
stage
actorId
resource
expectedIdentity
actualIdentity
candidateRefs
safeReason
nextAllowedAction
traceId
```

This is a conceptual format. When a Role uses the existing Agent Result, it SHOULD select the `blocked`, `failed`, or `incomplete` Status and place the fixed Code in a verifiable Payload, Blocker, or Trace. It cannot add fields that the Schema does not define.

### 11.3 State Conflict and Design Conflict

A State Revision Conflict means that a write premise has expired. It does not always require Human arbitration. If the Transition remains valid after reload, the Orchestrator MAY create a new Mutation.

A Design Conflict means that two authoritative engineering decisions are incompatible. The Orchestrator MUST NOT choose one side or apply last-write-wins. The workflow enters `NEEDS_COORDINATOR_ARBITRATION`.

## 12. Automatic handling and Human arbitration

### 12.1 Automatic handling

The Orchestrator or controlled Automation MAY handle:

- Duplicate Delivery with the same Idempotency Key and Request Hash.
- Temporary State Lock Contention.
- A CAS failure after reload when every semantic premise remains unchanged.
- Cleanup of temporary files and Orphan Candidates that no State or Handoff references.
- Mechanical Source Integration explicitly allowed by Policy.
- Repeated Artifact Publication with an identical Checksum.

Automatic handling still requires a Receipt or Trace and remains subject to Retry limits and Retention Policy.

### 12.2 Conditions that require a stop

An Agent MUST NOT resolve these conditions on its own:

- Conflicts in Requirements, Design, Scope, Schema, or an API Contract.
- Different content within one Path when no Integration Owner is assigned.
- Accepted Pointer contention that requires comparison of several valid Candidates.
- A changed Diff that invalidates a previous Review or Approval.
- An external side effect whose outcome remains unknown.
- Failed State Integrity, Artifact Integrity, or Fencing Token validation.

The Coordinator arbitrates engineering semantics. A Human handles high-risk Approval, Permission, and irreversible side effects. The Orchestrator performs reproducible checks and routing only.

### 12.3 Version Conflict Resolution

After a Human or Agent resolves a Conflict, it MUST NOT modify a published Candidate in place. The workflow SHOULD:

```text
load conflict evidence
→ record resolution decision
→ create new candidate
→ recalculate diff / checksum
→ rerun required validation
→ rerun review
→ attempt accepted pointer CAS
```

If the Resolution Decision changes Requirements, Design, or Scope, the workflow MUST update Durable Knowledge first.

## 13. Handoff `onConflict` and Schema compatibility

### 13.1 Using the current Handoff

[`handoff-envelope.schema.json`](../schemas/handoff-envelope.schema.json) provides `onConflict`, but it has no fields for `expectedRevision`, `baseCommit`, `mutationId`, `leaseId`, or `fencingToken`. It also uses `additionalProperties: false`. Writers cannot add these custom fields to the current Handoff.

The first implementation can use:

- `requiredInputRefs` to pin Source, Candidates, Evidence, and a State Reference.
- The Artifact Reference `uri` and `sha256` to pin content.
- `description` to state the Base Identity, Candidate or Accepted status, and purpose.
- `onConflict` to select a valid Phase and Owner.
- Orchestrator Invocation Metadata to carry Expected Revision and Idempotency Key.
- Trace or Host Journal to retain Lock, Lease, Receipt, and Conflict Detail.

### 13.2 `onConflict` routing

Handoff `onConflict` is a predefined safe route. It does not turn every Conflict into design arbitration. The Orchestrator SHOULD classify the Conflict first:

| Conflict type | Route |
|---|---|
| Temporary Lock / Revision Contention | Retain a safe Phase, use a bounded Retry, or return `INCOMPLETE` |
| Missing Source / Evidence | Return `INCOMPLETE` and create a completion Handoff |
| Design / Scope / Semantic Conflict | `NEEDS_COORDINATOR_ARBITRATION` |
| Integrity / Partial Commit | `FAILED` or safe Phase with a Blocker |
| Human Approval Rejected / Changed | Return to a valid pre-Gate Phase |

`onConflict.phase` is a general string in the Handoff Schema. Before writing it to Current State, the Orchestrator MUST still validate it against the Current State Phase Enum and Transition Invariants.

### 13.3 Do not use `FAILED_RETRYABLE`

Failure Handling in [`05-security-and-maintainability.md`](./05-security-and-maintainability.md) mentions `FAILED_RETRYABLE`, but the current [`current-state.schema.json`](../schemas/current-state.schema.json) does not list that Phase. Implementations of this document MUST NOT write it into Current State.

For retryable work, use one of the existing options:

- Retain the current Phase and record a Blocker.
- `INCOMPLETE`.
- `FAILED`.
- `NEEDS_COORDINATOR_ARBITRATION`.

Adding a Phase later requires a Schema, Transition Table, Example Fixture, and Migration update first.

## 14. Conceptual Mutation Policy

The following YAML illustrates Mutation controls. It does not conform to the current `workflow-policy.template.yaml` and cannot be embedded in the current Handoff:

```yaml
schemaVersion: 0.1.0-draft
policyId: local-mutation-policy
policyVersion: 0.1.0

state:
  writerRoles: [automation]
  requireExpectedRevision: true
  incrementRevisionBy: 1
  atomicReplaceRequired: true
  lockScope: commit-critical-section
  maxCasRetries: 1

artifacts:
  publishMode: create-only
  overwrite: deny
  checksumRequired: true
  producerMayAccept: false

source:
  defaultMode: single-writer
  parallelIsolation: worktree
  requireBaseCommit: true
  requirePathOwnership: true
  sharedFilesRequireIntegrationOwner: true

integration:
  createNewCandidate: true
  requireCombinedValidation: true
  requireReviewAfterMerge: true
  automaticMergeRiskMax: low

idempotency:
  requiredForSideEffects: true
  sameKeyDifferentPayload: deny
  reconcileUnknownOutcome: true

conflicts:
  semantic: coordinator-arbitration
  acceptedPointer: coordinator-arbitration
  integrity: fail-closed
```

Before adopting this format, the repository needs at least:

```text
schemas/mutation-request.schema.json
schemas/mutation-receipt.schema.json
schemas/lease-record.schema.json
templates/mutation-policy.template.yaml
examples/state-revision-conflict/
examples/parallel-worktree-integration/
```

These assets belong to a later contract upgrade. Local adoption of this document does not depend on them.

## 15. Local parallel implementation example

### 15.1 Assign work

The Coordinator approves a Change and assigns two tasks to separate Implementers:

```text
baseCommit: abc123
currentStateRevision: 5

Implementer A
  worktree: worktrees/payment-service
  ownedPaths:
    - src/payment/**

Implementer B
  worktree: worktrees/payment-tests
  ownedPaths:
    - tests/payment/**
    - package-lock.json
```

`package-lock.json` is a Shared Generated File. The Coordinator assigns its regeneration to the Integration Owner, so Implementer B's modification remains Candidate Input only.

### 15.2 Produce separate Candidates

```text
candidate-A
  baseCommit: abc123
  changedFiles:
    - src/payment/service.ts
  diffHash: hash-a
  validation: passed

candidate-B
  baseCommit: abc123
  changedFiles:
    - tests/payment/service.test.ts
    - package-lock.json
  diffHash: hash-b
  validation: passed
```

A and B are Candidates. Neither has entered `latestArtifacts.implementationResult`.

### 15.3 Integration

The Integration Owner:

1. Verifies that both Candidates use `abc123`.
2. Applies `src/payment/service.ts` and the Test Change.
3. Regenerates `package-lock.json` against the integrated Dependency Graph.
4. Produces `integrated-candidate-C`, a new Diff Hash, and a new Changed Files list.
5. Runs the Combined Test, Lint, Build, and required Validation.
6. Creates a Review Handoff that pins C and its Evidence.

If the Lockfile cannot be regenerated, the workflow returns `PATH_OWNERSHIP_CONFLICT` or `SOURCE_MERGE_CONFLICT`. The original A and B Candidates remain unchanged.

### 15.4 Review and Pointer update

After the Reviewer approves C, the Orchestrator prepares an update from Revision 5:

```text
expectedRevision: 5
acceptedCandidate: integrated-candidate-C
nextPhase: READY_FOR_READINESS_CHECK
nextOwner: readiness
```

If another valid Transition has already moved State to Revision 6, C's commit returns `STATE_REVISION_CONFLICT`. The Orchestrator reloads Revision 6. It creates a new Pointer Mutation only if Review, Gates, Phase, Owner, and Target Binding remain compatible.

### 15.5 Human Git Gate

After Readiness and Archive complete, the Approval Record binds C's Diff Hash. If later Conflict Resolution creates Candidate D, C's Approval does not apply to D. A Human MUST inspect D's Evidence and create a new Approval.

## 16. Minimum local adoption

A local implementation can adopt most of these rules without changing the existing Schemas:

1. Allow only one Orchestrator or Host to update `current-state.json` for each Change.
2. Define `revision` to increment by 1 after each successful State Mutation.
3. Require the caller to provide `expectedRevision` to the State Update API.
4. Use a temporary file on the same Filesystem, Schema Validation, and Atomic Replace.
5. Do not hold a State Lock while an Agent runs.
6. Give each Artifact a unique URI, Create-only publication, and a Checksum. Prohibit overwrite.
7. Use `latestArtifacts` only for Accepted Artifacts.
8. Let Producers create Candidates. Let the Orchestrator update Pointers.
9. Pin one Implementer to a Base Commit. Use separate Worktrees for parallel work.
10. Record Path Ownership and Shared File Ownership in approved Change Scope.
11. Create an Integrated Candidate after Merge, then rerun Validation and Review.
12. Give side-effect Tools an Idempotency Key and Before / After References.
13. Place Conflict Codes in a Blocker or Trace and route them to an existing valid Phase.
14. After an external Write times out, Reconcile before deciding whether to Retry.

### 16.1 Optional local directory

The first implementation MAY add a Host-only area under the Git-ignored Runtime:

```text
.agent-runtime/<change-id>/
├─ current-state.json
├─ runs/
│  └─ <run-id>/
│     ├─ artifacts/
│     └─ evidence/
├─ locks/
│  └─ current-state.lock
└─ host-journal/
   └─ mutation-receipts/
```

`locks/` and `host-journal/` are Runtime Implementation Details, not Agent Context. Agents SHOULD NOT modify or depend on their content.

### 16.2 Local Commit Critical Section

A minimum Orchestrator SHOULD wrap these operations in one controlled function:

```text
commitStateMutation(
  changeId,
  expectedRevision,
  proposedTransition,
  acceptedArtifactRefs,
  idempotencyKey
)
```

The function handles Lock acquisition, Revision Compare, Schema Validation, Transition Validation, Atomic Replace, and Receipt creation. An Agent Adapter cannot assemble or overwrite `current-state.json` directly.

## 17. Criteria for adopting a Durable Store

Evaluate moving Current State and Mutation Receipts to a Durable Store with Transaction or CAS support when any condition applies:

- Multiple Orchestrator Processes may handle the same Change.
- Workers run on multiple hosts.
- Runtime resides on a Network Filesystem.
- The workflow needs Leases, Fencing Tokens, or failover across hosts.
- Mutation Receipt and State Update require transactional consistency.
- External side effects require a reliable Outbox or Inbox.
- Audit requirements must reconstruct every State Transition.
- The volume of Parallel Runs makes local Locks and Journals difficult to manage.

The upgraded Store MUST preserve the same semantics:

```text
expected revision
conditional write
immutable candidate
accepted pointer
idempotency receipt
conflict classification
```

Changing Storage cannot change how Agents interpret Candidates, Accepted Artifacts, Gates, and Role Boundaries.

### 17.1 Cases that do not require an upgrade

Local file mode is usually sufficient when:

- One Orchestrator Process runs.
- Each Change has one Source Writer.
- Runtime is not shared across hosts.
- Commit, Push, and Deploy retain a Human Gate.
- The Host can perform bounded Recovery after a Crash.

In this case, an In-process Mutex, Revision Check, Atomic Replace, and Immutable Artifacts provide clear boundaries.

## 18. Acceptance checklist

### State Mutation

- [ ] `revision` increments by exactly 1 after each successful State Mutation.
- [ ] `attempt` is not used as a CAS Token.
- [ ] A Writer submits the Expected Revision that it read.
- [ ] A CAS failure does not overwrite newer State.
- [ ] Current State uses a complete Snapshot and Atomic Replace.
- [ ] Agent Invocation, Validation, and Human Wait do not hold a State Lock.

### Artifacts and Pointers

- [ ] A Candidate uses a unique URI, Checksum, and Create-only Publication.
- [ ] A published Artifact does not allow in-place modification.
- [ ] State references only complete Artifacts that have passed Integrity checks.
- [ ] `latestArtifacts` retains Accepted Pointer semantics.
- [ ] A Producer cannot update an Accepted Pointer.
- [ ] Pointer, Gate, Phase, Owner, and Handoff use the same State CAS.

### Source and Integration

- [ ] Every Implementer has a pinned Base Identity and Path Ownership.
- [ ] Parallel Implementers use separate Worktrees or Branches.
- [ ] Each Shared Generated File has one Integration Owner.
- [ ] Base Divergence is never ignored silently.
- [ ] A Merge creates a new Integrated Candidate.
- [ ] The Integrated Candidate reruns Combined Validation and Review.

### Locks, Leases, and Retry

- [ ] A Lock covers only the short Commit Critical Section.
- [ ] A multi-host Lease uses Expiration and a Fencing Token.
- [ ] Every side-effect Mutation has an Idempotency Key.
- [ ] The system rejects the same Key with a different Payload.
- [ ] Retry has an attempt limit, Backoff, and termination conditions.
- [ ] An unknown side-effect outcome triggers a System of Record query first.

### Conflicts and Recovery

- [ ] State, Source, Semantic, Pointer, and side-effect Conflicts use separate handling.
- [ ] Every Conflict Result has a fixed Code and Trace ID.
- [ ] The Coordinator arbitrates Design and Scope Conflicts.
- [ ] A changed Diff cannot reuse an old Review or Approval.
- [ ] A Partial Write cannot leave a Dangling State Reference.
- [ ] Orphan Candidates have a Retention or Cleanup Policy.

### Schema compatibility

- [ ] The current Handoff contains no undefined Mutation fields.
- [ ] Orchestrator Metadata or Journal manages Expected Revision and Idempotency Key.
- [ ] An `onConflict` target passes Current State Phase and Transition validation.
- [ ] The current Current State never receives `FAILED_RETRYABLE`.
- [ ] Future Schema assets have Version and Migration rules before adoption.

## 19. References

- [Artifact-based Shared State + Structured Handoff reference architecture](./02-reference-architecture.md)
- [Repository Knowledge, Runtime State, Evidence, and Trace layers](./03-runtime-storage-and-retention.md)
- [Security, governance, and long-term maintenance](./05-security-and-maintainability.md)
- [Context Authority and Retrieval Policy](./06-context-authority-and-retrieval-policy.md)
- [Role capability and scope control](./07-role-capability-and-scope-control.md)
- [Agent Platform Operations](../../../context-engineering/docs/03-agent-platform-operations.md)
- [Tool Governance and Evaluation](../../../agent-design/tool-schema-routing/docs/03-tool-governance-and-evaluation.md)
- [Handoff Envelope Schema](../schemas/handoff-envelope.schema.json)
- [Current State Schema](../schemas/current-state.schema.json)
- [Agent Result Schema](../schemas/agent-result.schema.json)
- [Workflow Policy Template](../templates/workflow-policy.template.yaml)
