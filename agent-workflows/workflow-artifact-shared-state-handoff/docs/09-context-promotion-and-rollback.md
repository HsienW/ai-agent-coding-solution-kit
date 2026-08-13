# 09 | Context promotion and rollback

[English](./09-context-promotion-and-rollback.md) | [繁體中文](./09-context-promotion-and-rollback-zh-TW.md)

This document defines how Coordinators, Orchestrators, Agents, Archivists, Runtime Stores, and Human Approvers promote Candidate Context to Accepted Context, turn Runtime conclusions into Durable Knowledge, and perform traceable Rollback when accepted content becomes invalid. Promotion and Rollback both change the Context that downstream Agents trust. Each operation MUST bind a fixed version, Gate, State Revision, Capability, and Evidence.

This document uses the following normative terms:

- "MUST" defines a requirement that the workflow cannot omit.
- "SHOULD" defines the default practice; a deviation requires a recorded reason.
- "MAY" defines an optional capability that a team can adopt according to its scale and risk.

## 1. Scope

[`06-context-authority-and-retrieval-policy.md`](./06-context-authority-and-retrieval-policy.md) defines which Context an Agent should trust. [`07-role-capability-and-scope-control.md`](./07-role-capability-and-scope-control.md) limits who may propose or apply a change. [`08-mutation-concurrency-and-conflict-resolution.md`](./08-mutation-concurrency-and-conflict-resolution.md) governs CAS and concurrent writes to the Accepted Pointer. This document addresses the questions that follow:

- Which conditions must a Candidate satisfy before becoming Accepted Context?
- How should the Accepted Pointer, Gates, Phase, and Handoff change together?
- Which Runtime conclusions should move into OpenSpec or Git?
- Can the workflow point directly to an older version after finding a Regression in an Accepted Artifact?
- How does a Rollback Target prove that it still applies to the current Requirement and Source Base?
- Which Review, Readiness, and Approval records become invalid after Rollback?
- Which recovery path applies to an Active Run or a Terminal Run?
- Which Actions may automation perform, and which ones require a Coordinator or Human?

This document covers Accepted Context Promotion, Runtime-to-Durable Knowledge Promotion, Pointer Re-selection, Compensating Rollback, Gate Invalidation, Cross-run Recovery, and Rollback Evidence. Other documents cover these subjects:

- For the base structures of Current State, Agent Result, Handoff, and Artifact Reference, see [`02-reference-architecture.md`](./02-reference-architecture.md).
- For Runtime, Evidence, Trace, Retention, and Archive layers, see [`03-runtime-storage-and-retention.md`](./03-runtime-storage-and-retention.md).
- For Artifact Integrity, Human Gates, and State Machine Governance, see [`05-security-and-maintainability.md`](./05-security-and-maintainability.md).
- For Context Authority, Version Pinning, Freshness, and Binding, see [`06-context-authority-and-retrieval-policy.md`](./06-context-authority-and-retrieval-policy.md).
- For Role, Capability, Path, Tool, and Approval Scope, see [`07-role-capability-and-scope-control.md`](./07-role-capability-and-scope-control.md).
- For CAS, Atomic Replace, Idempotency, and Conflict Routing, see [`08-mutation-concurrency-and-conflict-resolution.md`](./08-mutation-concurrency-and-conflict-resolution.md).

This document does not define implementations for Production Deployment, Database Rollback, Feature Flags, Traffic Shifts, or business data compensation. It does not change the existing JSON Schemas.

## 2. Terms and operation boundaries

### 2.1 Promotion

Promotion is a controlled operation that raises the trust level or retention lifetime of a Context item. This document defines two types:

| Type | Source | Target | Primary result |
|---|---|---|---|
| Accepted Context Promotion | Candidate Artifact | Accepted Context | Current State updates the Accepted Pointer |
| Runtime-to-Durable Knowledge Promotion | Accepted Runtime Artifact and Evidence | OpenSpec / Git | Create a durable Execution Summary, ADR, or Spec correction |

Accepted Context Promotion changes the default input for downstream Stages in the current Run. Durable Knowledge Promotion changes engineering facts that later Changes and future maintainers must understand. Each type requires its own Gates. One "completed" Boolean cannot represent both.

### 2.2 Rollback, Revert, Restore, and Re-run

| Name | Meaning | Rule in this document |
|---|---|---|
| Rollback | Restore effective behavior or trusted Context to a known acceptable state | Complete through a new Decision and Revision |
| Revert | Create a reverse or compensating Diff against current Source | Treat as a new Source Mutation and Candidate |
| Restore | Recover damaged data from a backup or Durable Store | Never move Current State Revision backward |
| Re-run | Execute a Stage again with the same or corrected inputs | Produce a new Attempt, Artifact, or Run |
| Invalidate | Revoke the applicability of a Pointer, Gate, Handoff, or Approval | Preserve the original record and add an invalidation reason |
| Supersede | Give a new Artifact or Durable Decision current authority over an older version | Keep the older version Immutable and traceable |

Overwriting Current State with an old `current-state.json` is not a valid Rollback. It moves `revision` backward and may revive revoked Gates, Approvals, and Handoffs.

### 2.3 Promotion Target

A Promotion Target is the resource that becomes an authoritative downstream input or durable engineering fact after Promotion. The Target MUST have a defined Fact Type and version identity, for example:

```text
implementation fact → accepted implementation artifact
review verdict      → accepted review result
validation fact     → evidence bound to accepted implementation
execution outcome   → Git-tracked execution summary
design deviation    → approved OpenSpec / ADR update
```

A Promotion can assert only the Fact Type that it verified. Passing Implementation Review does not approve a Requirement change. Completing Archive does not authorize Commit or Push.

## 3. Roles and responsibilities

| Role | Promotion / Rollback responsibility | Fixed restrictions |
|---|---|---|
| Coordinator | Determine semantic applicability, resolve Scope or Design Conflicts, approve the Rollback Plan | Does not modify Current State directly or replace Reviewer or Human Gates |
| Implementer | Create Candidates, compensating Diffs, and Validation Evidence | Does not accept its own output, overwrite an old Candidate, or switch a Pointer directly |
| Reviewer | Review a pinned Candidate, Diff, and matching Evidence | Does not modify Source, Gates, the Accepted Pointer, or upstream Artifacts |
| Readiness | Determine whether the Accepted Artifact and Gates support Archive | Does not restore an old Pointer or perform Implementer work |
| Archive | Create an Archive Result and controlled Execution Summary | Does not Commit directly or promote unaccepted Runtime content into Git |
| Orchestrator | Validate versions, Gates, Capabilities, and CAS; apply a legal State Mutation | Does not arbitrate Requirements, issue a quality Verdict, or replace Human Approval |
| Human | Approve Git, Deploy, Delete, external Rollback, and other high-risk Actions | Approval MUST bind a fixed Target, Diff, Scope, Revision, and validity period |

An Artifact Producer MAY propose Promotion or Rollback in its Result. The proposal itself carries no Accept, Apply, or Approve permission.

## 4. Context lifecycle invariants

Every implementation MUST preserve these invariants:

1. A Candidate becomes Accepted Context only through explicit Gates.
2. A Producer cannot update the Accepted Pointer for its own output.
3. A Promotion Decision binds a fixed URI, Checksum, Change ID, Run ID, and Target Identity.
4. The Accepted Pointer, Gates, Phase, Owner, and Handoff use one State CAS.
5. A failed Promotion preserves the current Accepted Pointer.
6. A Superseded or rolled-back Artifact remains Immutable.
7. Rollback never moves Current State `revision` backward.
8. When Source has changed, Pointer Re-selection cannot replace Source Revert.
9. New Source content produced by Rollback becomes a new Candidate.
10. The new Candidate receives its own Diff, Validation, Review, and Approval Binding.
11. A Terminal Run does not reopen because of a later problem. Follow-up work creates a new Change or Run.
12. A Durable Knowledge correction preserves Git history and does not erase a published Decision in place.
13. The workflow reevaluates Handoff, Review, Readiness, and Approval applicability whenever Target Binding changes.
14. When an external side-effect outcome is unknown, the Executor queries the System of Record first.

Promotion Decisions, Accepted Pointers, Gates, Approvals, and Rollback Target selection MUST NOT use last-write-wins.

## 5. Promotion types and Gates

Each Artifact Kind has its own Promotion conditions:

| Promotion Target | Required conditions | Decision Owner | Writer |
|---|---|---|---|
| Plan / Design Candidate | Requirement Binding, Plan Review, Scope Approval | Coordinator / Human Policy | Orchestrator |
| Implementation Candidate | Schema, Diff Binding, Validation, Review Verdict | Reviewer provides Verdict; Policy evaluates Gate | Orchestrator |
| Review Result | Review target matches the Accepted Candidate; Result is valid | Orchestrator validates | Orchestrator |
| Readiness Result | Accepted Implementation, Review, Evidence, and Task status agree | Readiness provides Verdict | Orchestrator |
| Archive Result | Readiness Passed, Summary complete, Human Gate state explicit | Archive proposes; Orchestrator validates | Orchestrator |
| Execution Summary | Contains only Accepted Outcome, Deviations, Risks, and References | Archive / Coordinator | controlled Git Executor |
| ADR / Spec correction | Durable Decision approved and Scope explicit | Coordinator / Human | controlled Spec Writer / Git Executor |

Reviewer Verdict, Gate Transition, Pointer Update, Human Approval, and external Action are separate events. The presence of one cannot prove that another has occurred.

## 6. Accepted Context Promotion

### 6.1 Promotion Eligibility

Before accepting a Candidate, the Orchestrator MUST verify:

- The Candidate passes the applicable Agent Result Schema.
- `changeId`, `runId`, `producer`, `stage`, and `kind` match the Handoff.
- The Candidate URI and Checksum are pinned, and its content is complete and Immutable.
- The Candidate Source Base and Diff Hash match the Target under review.
- Validation Evidence binds the same Candidate and Diff.
- The Review Result explicitly reviews the same Candidate.
- Requirement, Acceptance Criteria, and Scope did not change during Review.
- Role, Capability, Path, and Environment remain inside Effective Scope.
- Required Gates passed, and Policy has handled every Blocker and Major Finding.
- Current State still matches the Expected Revision that the Orchestrator read.

Promotion MUST stop when any condition cannot be proven. A newer filename, larger Attempt number, or later creation time cannot replace a missing Gate.

### 6.2 Promotion Decision

A Promotion Decision conceptually contains at least:

```text
promotionId
promotionType
changeId
runId
actorId
candidateRef
targetFactType
sourceAuthority
targetAuthority
evidenceRefs
gateRefs
expectedRevision
policyVersion
decision
reasonCode
decidedAt
traceId
```

The current Schemas have no Promotion Decision field. In the first local implementation, Trace, a Host Journal, or an independent Artifact SHOULD retain the Decision. Writers MUST NOT insert it into Current State, Agent Result, or Handoff.

### 6.3 Promotion commit order

Accepted Context Promotion SHOULD follow this order:

```text
load Current State at expectedRevision
→ resolve Candidate and required Evidence
→ validate Schema / checksum / target binding
→ evaluate Review and Promotion Gates
→ create Promotion Decision
→ build next State Snapshot
→ CAS Accepted Pointer + Gate + Phase + Owner + Handoff
→ record Mutation Receipt and Trace
```

The Orchestrator MUST validate the Promotion Decision before CAS. The Receipt records the actual commit outcome. If CAS fails, the Decision remains an unapplied proposal. The Orchestrator reloads State and evaluates it again against the new Revision.

### 6.4 Context after Promotion

After a successful Promotion:

- `latestArtifacts` points to the new Accepted Artifact.
- Downstream Handoffs pin the new URI and Checksum.
- The previous Accepted Artifact becomes Superseded Context and remains available for audit and Rollback analysis.
- Other unaccepted Candidates retain Candidate status.
- An old Handoff becomes invalid if it binds a different Target.
- A downstream Agent cannot select a newer file from the directory to replace the Pointer.

The current Schema has no `supersededBy` field. Trace, a Host Journal, Artifact `description`, or an independent index retains the Supersede relationship.

## 7. Runtime-to-Durable Knowledge Promotion

### 7.1 Content eligible for Promotion

When a Change completes or enters Archive, the Archivist SHOULD produce a compact Durable Output from Accepted Artifacts and Evidence:

- The completed Scope and main changed files.
- Final Validation and Review results.
- Deviations from the original Proposal or Design.
- Accepted Risks, remaining work, and follow-up items.
- Decisions that affect later maintenance.
- Accepted Artifact, Commit, CI, or Release References.

When a Requirement, Design, or Architecture Decision changes permanently, the workflow SHOULD update the corresponding OpenSpec or ADR. An Execution Summary cannot replace the Durable Contract that requires correction.

### 7.2 Content excluded from Promotion

The following content remains in Runtime, Evidence, or Trace by default:

- Full conversations and private reasoning.
- Raw responses from every Retry.
- Complete stdout, Debug Logs, and Token details.
- Full content of Candidates that failed Validation.
- Temporary summaries that a later version Superseded.
- Secrets, Credentials, personal data, or unapproved external content.
- Diagnostic data unrelated to the final Decision.

If audit requirements retain any of these items, Retention Policy SHOULD name the controlled Store, access rights, and duration. The workflow SHOULD NOT commit the content directly into Git.

### 7.3 Durable Promotion Gate

Before an Archivist or Git Executor performs Durable Promotion, it MUST verify:

- The Readiness Result points to the current Accepted Artifact.
- The Execution Summary contains only pinned, resolvable References.
- Validation, Review, and Risk statements match their source Artifacts.
- Durable Contract corrections passed the applicable Coordinator or Human Gate.
- Commit Scope contains no Runtime Trace, Secret, or unaccepted Candidate.
- Human Approval binds the actual Diff, Target Branch, Action, and validity period.

The Archive Agent MAY create an `archive_result` and an Execution Summary Candidate. Only a controlled Executor with the applicable Capability can write to Git. Commit and Push remain separate Actions.

### 7.4 Durable Promotion completion

Promotion completes only after the result is visible in the System of Record:

```text
prepare Execution Summary / Spec update
→ validate content and references
→ obtain Human Approval when required
→ commit through controlled Executor
→ verify Commit Identity in Git
→ record Commit Reference in Archive Evidence / Trace
```

After a Client Timeout or lost tool response, the Executor MUST query Git, CI, or the applicable System of Record. If it cannot confirm the result, it returns `DURABLE_PROMOTION_OUTCOME_UNKNOWN` and MUST NOT resend Commit or Push blindly.

## 8. Promotion CAS and concurrent decisions

When two Candidates request Promotion concurrently, each submits the Revision it read:

```text
candidate v2 reads revision 8
candidate v3 reads revision 8

v2 passes gates and commits revision 9
v3 submits expectedRevision 8
→ STATE_REVISION_CONFLICT
→ accepted pointer remains v2
```

v3 remains a Candidate. The Orchestrator cannot change `expectedRevision` to 9 and resubmit without reevaluation. It MUST check whether v3 still satisfies the current Scope, Gates, Owner, Review Target, and Promotion Policy.

If both v2 and v3 are valid but represent incompatible engineering Decisions, the workflow returns `PROMOTION_AUTHORITY_CONFLICT` and enters `NEEDS_COORDINATOR_ARBITRATION`. Creation time cannot arbitrate the conflict.

## 9. Rollback types

### 9.1 Accepted Pointer Re-selection

Pointer Re-selection directs downstream Context to a previously Accepted Artifact. The workflow MAY use it only when every condition holds:

- Actual Source, Config, and execution baseline still match the older Artifact description.
- The older Artifact passes Integrity, Retention, and readability checks.
- Requirement, Schema, Policy, and Environment have not invalidated the older version.
- Required Validation remains valid under Freshness Policy.
- A new State Mutation recalculates Gates, Owner, and Handoff.

Once Source contains a later version, Pointer Re-selection cannot declare the system restored. The workflow MUST create a Compensating Candidate.

### 9.2 Compensating Source Rollback

A Compensating Rollback creates new corrective content against the current Source Base to restore acceptable behavior:

```text
current source with v2
+ accepted behavior from v1
+ current requirement / schema / dependency constraints
                    ↓
           rollback candidate v3
```

v3 is a new Candidate. It may use a Revert Commit, reverse Patch, manual correction, or replacement implementation. Every approach MUST produce a new Diff Hash, Changed Files list, Validation Evidence, and Review Result.

### 9.3 Durable Knowledge Correction

When a committed Execution Summary, ADR, Spec, or Task status is wrong, the Coordinator SHOULD create a new OpenSpec or Git Change to correct it. The correction MUST:

- Identify the corrected Commit, Document Version, or Decision Reference.
- State which facts remain valid and which ones are Superseded.
- Provide the new Requirement, Design, or Execution Evidence.
- Pass the Review and Human Git Gate required by the document type.

The workflow SHOULD NOT rewrite a historical Commit in place because its content became stale. Repository Governance handles exceptional cases that require rewriting public history. Agent automation has no authority to perform them.

### 9.4 External Environment Rollback

Separate operational policies govern Deploy, Migration, Feature Flag, Remote Config, and business data Rollback. This document requires only that:

- The Handoff pins Environment, Target Version, Action, and Evidence.
- The Executor has an explicit Capability and valid Human Approval.
- The Action uses an Idempotency Key or an equivalent platform mechanism.
- The Executor reads the System of Record before and after execution.
- Trace or Archive Evidence records the result through an External Reference.

Agent Result, Reviewer Verdict, and `requiresHumanApproval: true` are not Approval Proof for an external Rollback.

## 10. Rollback Target selection

### 10.1 Target Eligibility

When selecting a Rollback Target, the Coordinator and Orchestrator MUST check:

| Check | Question |
|---|---|
| Identity | Can the workflow verify the Target URI, Checksum, Commit, Run, and Producer? |
| Prior Acceptance | Did the Target pass the required Validation and Review when it was accepted? |
| Current Requirement | Does the Target behavior still satisfy the currently approved Requirement? |
| Current Base | Are Source, Dependencies, Schema, and Environment still compatible? |
| Security | Does the Target contain a known Vulnerability, revoked Credential, or prohibited setting? |
| Retention | Are the required Artifact, Diff, and Evidence still available? |
| Scope | Which Paths, Components, Data, and external systems will Rollback affect? |
| Approval | Does the current Action require new Human Approval? |

Past success proves that the Target worked under earlier conditions. It does not prove current applicability.

### 10.2 Rollback Basis and Rollback Candidate

The Rollback Basis is an older Artifact, Commit, or Decision used to restore behavior. The Rollback Candidate is the new result produced against the current baseline. The workflow MUST record them separately:

```text
rollbackBasisRef: accepted-v1
currentBaseRef: source-at-v2
rollbackCandidateRef: candidate-v3
```

The Reviewer reviews v3 and cannot apply v1's old Verdict directly. Evidence for v1 MAY provide a comparison baseline, but it cannot replace Validation in the current environment.

### 10.3 Unusable Targets

The workflow MUST NOT perform an automatic Rollback when any condition applies:

- The Target Artifact is missing or its Checksum does not match.
- Requirement, Schema, Dependencies, or data format are incompatible.
- The older version contains a known Security Issue.
- Rollback requires data or permissions outside approved Scope.
- The workflow cannot confirm current Source or Environment Identity.
- Several Rollback Targets are valid but have incompatible behavior.

The Coordinator SHOULD create a new correction Plan instead. A Human decides when the conflict involves high-risk external state.

## 11. Compensating Rollback flow

### 11.1 Prepare

The Coordinator creates a Rollback Plan that pins at least:

```text
incident / regression reference
current accepted artifact
current source base
rollback basis
affected scope
expected restored behavior
required validation
required review
human-gated actions
```

If the Regression shows that the original Requirement or Design is wrong, the Coordinator MUST update the Durable Contract before handing work to the Implementer.

### 11.2 Implement

In a separate Worktree or controlled worktree, the Implementer:

1. Verifies that Current Base matches the version pinned by the Handoff.
2. Reads the Rollback Basis, current Requirement, and required Source.
3. Creates a compensating Diff without modifying an older Artifact.
4. Runs specified Validation and produces new Evidence.
5. Publishes a new Implementation Candidate.
6. Proposes a Review Handoff without updating the Pointer.

For Base Divergence, use `SOURCE_BASE_DIVERGED` from document 08. The workflow cannot apply an old Patch silently.

### 11.3 Review and Readiness

The Reviewer MUST check:

- Whether the compensating Diff stays within approved Scope.
- Whether the Candidate restores expected behavior.
- Whether it accidentally removes a later Requirement, Schema change, or Security Fix.
- Whether Validation covers the original Regression and current integrated behavior.
- Whether a Coordinator or Human must accept any Residual Risk.

After Review passes, Readiness still evaluates Gates against the Rollback Candidate. A Readiness Result for an older Candidate does not apply.

### 11.4 Accept

The Orchestrator submits a new Promotion Decision with the Expected Revision:

```text
accepted implementation: candidate-v3
review result: review-v3
readiness result: readiness-v3
next phase: policy-defined legal phase
next owner: policy-defined owner
```

The Accepted Pointer, related Gates, Phase, Owner, and Handoff MUST change together. If CAS fails, the workflow preserves current State. The Orchestrator reloads it before deciding whether v3 remains acceptable.

### 11.5 Durable Record

After Rollback completes, the Execution Summary or follow-up Change SHOULD retain:

- The Regression or Incident Reference.
- The replaced Accepted Artifact.
- The Rollback Basis and new Candidate.
- Actual Source or Environment changes.
- Validation, Review, and Human Approval References.
- Items that remain unrestored and follow-up work.

The workflow does not delete the original Promotion Decision or Artifact. When Retention expires, it retains enough Metadata to reconstruct the Decision relationship.

## 12. Gate, Handoff, and Approval invalidation

### 12.1 Gate Invalidation Matrix

After a Rollback or Promotion Target changes, the Orchestrator SHOULD recalculate Gates according to the impact:

| Change | Items invalidated by default |
|---|---|
| Implementation URI / Checksum changes | `implementationComplete`, `reviewPassed`, `readinessPassed`, `humanApproved` |
| Diff Hash / Changed Files change | Review, Readiness, Commit / Deploy Approval |
| Requirement / Scope changes | Every downstream Gate after Plan Approval |
| Validation Target changes | Review and Readiness |
| Review Result changes | Readiness and later Approval |
| Target Branch / Environment changes | Applicable Git / Deploy Approval |
| Policy Version or Capability Scope changes | Authorization Decision and unexecuted Approval |

A versioned Transition Policy defines the actual invalidation set. The Orchestrator cannot preserve an unbound `true` value merely to keep the workflow moving.

### 12.2 Handoff Invalidation

An existing Handoff becomes invalid when:

- Current State Revision changed, and the Handoff no longer matches the current Owner or Phase.
- A Required Input Ref points to a rolled-back or Superseded Target.
- Checksum, Run ID, Change ID, or Source Base does not match.
- Acceptance Criteria, Scope, Policy, or Permission changed.
- `expiresAt` has passed.

After invalidation, the Orchestrator SHOULD create a new Handoff from the latest State. The Agent cannot replace References on its own and continue execution.

### 12.3 Approval Invalidation

Human Approval MUST bind a fixed Action and content. Each of these changes requires new Approval:

- Diff, Payload, Commit, Target Branch, or Environment changes.
- Rollback expands from local Source to external Deploy or Data Mutation.
- State Revision or Policy Version changes and invalidates an original condition.
- Approval expires or is revoked.
- Executor, Tool, or Capability Scope changes.

A Reviewer `approve` Verdict cannot replace Human Approval. Previous Human Approval cannot replace Review of a new Candidate.

## 13. Active Runs, Terminal Runs, and Cross-run work

### 13.1 Active Run

When a problem appears after Promotion in a non-terminal Run, the Orchestrator SHOULD route it through a legal Transition:

- For an implementation correction, enter `CHANGES_REQUESTED` and create `fix-from-review` or another implementation Handoff allowed by Policy.
- For a Requirement, Design, or Scope Conflict, enter `NEEDS_COORDINATOR_ARBITRATION`.
- For missing Context or Evidence, use `INCOMPLETE` or retain a safe Phase with a Blocker.
- When Integrity cannot be proven, use `FAILED` or retain a safe Phase with a Blocker.

The current Phase Enum has no `ROLLING_BACK`. Implementations cannot add that Phase without a Schema change. Handoff Reason, Blocker, Trace, and Artifact References express Rollback intent.

### 13.2 Terminal Run

A Run with `terminal: true` remains closed. A later Regression, Requirement correction, or Rollback creates a new Change or Run that references the original Run's Accepted Artifact, Execution Summary, Commit, and Incident Reference.

The new Run MAY use the original Artifact as Diagnostic Context or a Rollback Basis. It cannot modify the original Run's Results, Gates, Handoffs, or Revision History.

### 13.3 Cross-run Binding

Cross-run References require explicit permission and verification of at least:

```text
source changeId / runId
target changeId / runId
fact type and purpose
artifact URI / checksum
source base / commit
authority level
retention availability
```

An Accepted Artifact from an older Run is Diagnostic Context or a Rollback Basis by default in the new Run. A new Candidate becomes current Accepted Context only after the new Run completes its required Gates.

## 14. Boundary between State Recovery and Rollback

When the Runtime Store is damaged, the Host MAY reconstruct Current State from Durable Knowledge, Git, Artifacts, and Mutation Receipts. The reconstructed State still receives a newer `revision` and a Recovery Trace.

The implementation MUST prohibit:

- Overwriting the current file with an old backup of `current-state.json`.
- Guessing which State to accept from file modification time.
- Retaining an Accepted Pointer to a missing Artifact.
- Copying old Gates and Human Approval into reconstructed State without reevaluation.
- Deleting an unexplained Blocker or History Reference merely to pass Schema validation.

Recovery addresses data integrity. Rollback addresses engineering behavior or trusted Context. Both may occur during one incident, but each requires a distinct Reason Code and Evidence.

## 15. Fixed failure codes and routing

### 15.1 Promotion Failure

| Code | Condition | Default handling |
|---|---|---|
| `PROMOTION_PRECONDITION_FAILED` | A required Gate, Evidence item, or Policy condition is unmet | Preserve the current Pointer and complete prerequisites |
| `PROMOTION_TARGET_STALE` | Candidate, Validation, or Source Base is stale | Create a new Handoff or rerun Validation |
| `PROMOTION_BINDING_MISMATCH` | Candidate, Diff, Validation, and Review point to different Targets | Return `INCOMPLETE` and reject Promotion |
| `PROMOTION_GATE_INVALID` | Gate content does not match Target or Policy | Clear the invalid Gate and return to a legal Stage |
| `PROMOTION_AUTHORITY_CONFLICT` | Several valid Candidates represent incompatible Decisions | Enter `NEEDS_COORDINATOR_ARBITRATION` |
| `DURABLE_PROMOTION_INCOMPLETE` | Summary or Spec correction lacks required References | Stop Archive or Git Action |
| `DURABLE_PROMOTION_OUTCOME_UNKNOWN` | Commit, Push, or external write cannot be confirmed | Query the System of Record and stop resubmission |

### 15.2 Rollback Failure

| Code | Condition | Default handling |
|---|---|---|
| `ROLLBACK_TARGET_MISSING` | Rollback Basis is missing or unavailable under Retention | Stop and obtain an alternative Plan from the Coordinator |
| `ROLLBACK_TARGET_INTEGRITY_FAILED` | Checksum, Schema, or source identity does not match | Return `FAILED` and isolate the Target |
| `ROLLBACK_BASE_DIVERGED` | Target is incompatible with current Source or Environment | Create a compensating Plan and do not apply directly |
| `ROLLBACK_REVALIDATION_REQUIRED` | Old Evidence does not cover the current Candidate | Create a Validation Handoff |
| `ROLLBACK_SCOPE_EXPANDED` | Actual impact exceeds approved Scope | Pause and request Coordinator or Human review |
| `ROLLBACK_GATE_INVALIDATED` | A Target change invalidates existing Gates | Clear applicable Gates and rerun Review or Readiness |
| `TERMINAL_RUN_IMMUTABLE` | A request attempts to modify a closed Run | Create a new Change or Run |
| `EXTERNAL_STATE_DIVERGED` | The System of Record differs from expected state | Stop automation and use the controlled operations workflow |

CAS Conflict, Base Divergence, Idempotency, and Unknown Side-effect Outcome reuse the codes defined in document 08. Context Integrity and Authority problems MAY reuse the fixed codes from document 06. Implementations SHOULD avoid creating duplicate names for the same condition.

### 15.3 Failure Result

A Promotion or Rollback Failure Trace or Result SHOULD retain at least:

```text
code
changeId
runId
stage
actorId
currentAcceptedRef
candidateRef / rollbackBasisRef
expectedIdentity
actualIdentity
invalidatedBindings
nextAllowedAction
traceId
```

This is a conceptual format. When using the current Agent Result, the Role SHOULD select a valid `blocked`, `failed`, or `incomplete` Status and place the fixed Code in an existing Payload, Blocker, Risk, Summary, or Trace. It MUST NOT add undefined Schema fields.

## 16. Handoff and current Schema compatibility

### 16.1 Using the current Handoff

[`handoff-envelope.schema.json`](../schemas/handoff-envelope.schema.json) has no `promotionId`, `rollbackBasisRef`, `rollbackCandidateRef`, `invalidatedGates`, or `targetAuthority` fields and uses `additionalProperties: false`. Writers cannot add these fields to the current Handoff.

The first implementation can use:

- `requiredInputRefs` to pin the Current Accepted Artifact, Rollback Basis, Current Source, Requirement, and Evidence.
- Artifact Reference `uri` and `sha256` to pin a version.
- `description` to identify a Reference as Candidate, Accepted, Rollback Basis, or Diagnostic Input.
- `reason` to describe the Promotion or Rollback task and its trigger.
- `acceptanceCriteriaRefs` to point to Regression Tests, Requirements, and Gate Criteria.
- `onSuccess`, `onFailure`, and `onConflict` to select existing legal Phases and Owners.
- Orchestrator Metadata to carry Expected Revision, Idempotency Key, and Policy Version.
- Trace or Host Journal to retain Promotion Decisions, Rollback Records, and invalidation details.

### 16.2 Handoff fragment

The following fragment omits other fields required by the Handoff Schema and is for reading only:

```json
{
  "from": "coordinator",
  "to": "implementer",
  "stage": "fix-from-review",
  "reason": "Create a compensating rollback candidate for regression REG-204",
  "requiredInputRefs": [
    {
      "refId": "current-accepted-v2",
      "type": "agent-result",
      "uri": "artifact://feature-example/run-004/implementation-v2.json",
      "sha256": "2222222222222222222222222222222222222222222222222222222222222222",
      "description": "Current accepted implementation; regression target"
    },
    {
      "refId": "rollback-basis-v1",
      "type": "agent-result",
      "uri": "artifact://feature-example/run-003/implementation-v1.json",
      "sha256": "1111111111111111111111111111111111111111111111111111111111111111",
      "description": "Previously accepted implementation; rollback basis only"
    }
  ]
}
```

`description` states purpose and does not raise Authority. The Orchestrator still verifies the Rollback Basis source, Integrity, Retention, and current applicability.

### 16.3 Current State compatibility

The current [`current-state.schema.json`](../schemas/current-state.schema.json) can represent:

- Current Phase, Owner, and Revision.
- The Accepted Pointer in `latestArtifacts`.
- Gates recalculated after Promotion or Rollback.
- A Blocker with a fixed Failure Code.
- A new Handoff and Next Action.

It cannot fully represent Promotion History, a Supersede Graph, or a Rollback Basis. Trace, a Host Journal, an Execution Summary, or an independent Artifact retains that data. Writers cannot insert custom fields into Current State.

## 17. Conceptual Promotion and Rollback Records

The following YAML illustrates a Decision Record and does not conform to a formal Schema in this repository:

```yaml
schemaVersion: 0.1.0-draft
recordId: promotion-record-004
recordType: accepted-context-promotion
changeId: feature-example
runId: run-004

from:
  authority: candidate
  artifactRef: artifact://feature-example/run-004/implementation-v3.json

to:
  authority: accepted
  factType: implementation

bindings:
  sourceBase: git:def456
  diffHash: sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
  validationRef: evidence://feature-example/run-004/validation-v3.json
  reviewRef: artifact://feature-example/run-004/review-v3.json
  expectedRevision: 11

decision:
  outcome: accepted
  appliedRevision: 12
  policyVersion: 1.0.0
  traceId: trace-promotion-004
```

A Rollback Record can be stored separately:

```yaml
schemaVersion: 0.1.0-draft
recordId: rollback-record-001
recordType: compensating-rollback
changeId: feature-example
runId: run-004

currentAcceptedRef: artifact://feature-example/run-004/implementation-v2.json
rollbackBasisRef: artifact://feature-example/run-003/implementation-v1.json
rollbackCandidateRef: artifact://feature-example/run-004/implementation-v3.json

reason:
  code: REGRESSION_CONFIRMED
  incidentRef: repo://incidents/REG-204.md

invalidatedBindings:
  - review-v2
  - readiness-v2
  - approval-v2

result:
  acceptedRef: artifact://feature-example/run-004/implementation-v3.json
  appliedRevision: 12
  traceId: trace-rollback-001
```

Before formal adoption, the repository needs at least:

```text
schemas/promotion-decision.schema.json
schemas/rollback-record.schema.json
templates/promotion-policy.template.yaml
examples/accepted-context-promotion/
examples/compensating-rollback/
```

These assets belong to a later contract upgrade. Local adoption of this document does not depend on them.

## 18. Complete local example

### 18.1 v2 Promotion

Current State is:

```text
revision: 8
accepted implementation: v1
source base: abc123

candidate v2
  source base: abc123
  validation: passed
  review: approved
```

After validating Bindings and Gates, the Orchestrator submits v2 with `expectedRevision: 8`. A successful CAS creates Revision 9, `latestArtifacts.implementationResult` points to v2, and the downstream Handoff pins v2 and its Evidence.

### 18.2 Regression discovered

Readiness or a later Human check finds that v2 broke existing behavior:

```text
regression: REG-204
current accepted: v2
known good basis: v1
current source: includes v2
```

Because Source contains v2, the Orchestrator cannot point directly back to v1. The Coordinator creates a Rollback Plan, names v1 as the Rollback Basis, and gives the original Regression Test, current Requirement, and Source Base to the Implementer.

### 18.3 Create v3

The Implementer produces v3 against the current Source Base:

```text
candidate v3
  rollback basis: v1
  current base: source-with-v2
  changed files:
    - src/payment/service.ts
    - tests/payment/service.test.ts
  validation:
    regression-test: passed
    current-suite: passed
```

v3 restores the acceptable behavior from v1 while retaining valid Dependency, Schema, and Security changes introduced after v2. It receives a new Diff Hash, Validation, Review, and Readiness Result.

### 18.4 Gate Invalidation and reacceptance

Creating v3 invalidates Review, Readiness, and Human Approval for v2. The Orchestrator routes Review and Readiness through new Handoffs, then updates State with a new Expected Revision:

```text
accepted implementation: v3
review result: review-v3
readiness result: readiness-v3
revision: 9 → 10 → ... → 12
```

State Revision continues forward. v1, v2, and v3 retain their original URIs. The Accepted Pointer ultimately points to v3.

### 18.5 Archive and Git

The Archive Execution Summary records the v2 Regression, v1 Rollback Basis, v3 compensation, and Validation results. If a Commit is required, Human Approval binds the v3 Diff Hash and Commit Scope. After execution, the controlled Executor queries Git and retains the Commit Reference.

If the Regression appears after the original Run reached `COMPLETED`, the workflow performs this work in a new Change or Run and leaves the original Run unchanged.

## 19. Minimum local adoption

A local implementation can adopt most of these rules without changing the existing Schemas:

1. Interpret `latestArtifacts` as the current Accepted Pointer.
2. Let Producers create Candidates and the Orchestrator update Pointers.
3. Before Promotion, validate URI, Checksum, Change ID, Run ID, Diff, and Evidence Binding.
4. Update the Accepted Pointer, Gates, Phase, Owner, and Handoff in one CAS.
5. After Promotion or Rollback, create a new Handoff bound to the new Revision.
6. When Source changed, create a Compensating Candidate instead of switching only the Pointer.
7. Give the Rollback Candidate new Diff, Validation, Review, and Readiness records.
8. Allow Current State `revision` to move forward only.
9. Keep a Terminal Run Immutable and create a new Change or Run for follow-up work.
10. Store Promotion and Rollback Decisions in Trace or a Host Journal first.
11. Let Archive promote only Accepted Outcomes and required Engineering Decisions.
12. Keep Runtime Trace, raw failures, and Secrets out of Git.
13. Retain Human Gates for Commit, Push, Deploy, Delete, and External Rollback.
14. Query the System of Record before acting on an unknown external write outcome.

### 19.1 Optional local directory

The Git-ignored Runtime MAY include Host-only records:

```text
.agent-runtime/<change-id>/
├─ current-state.json
├─ runs/
│  └─ <run-id>/
│     ├─ artifacts/
│     └─ evidence/
└─ host-journal/
   ├─ promotion-decisions/
   └─ rollback-records/
```

`host-journal/` is a Runtime Implementation Detail. Agents SHOULD NOT modify it or load the entire directory as ordinary Context.

### 19.2 Minimum Orchestrator interface

A local Orchestrator MAY wrap two controlled interfaces:

```text
promoteAcceptedContext(
  changeId,
  expectedRevision,
  candidateRef,
  evidenceRefs,
  gateDecision,
  idempotencyKey
)

applyRollbackPromotion(
  changeId,
  expectedRevision,
  rollbackBasisRef,
  rollbackCandidateRef,
  evidenceRefs,
  invalidatedBindings,
  idempotencyKey
)
```

Both interfaces handle Capability, Schema, Integrity, Binding, Transition, CAS, and Receipt. They cannot replace a Coordinator's semantic Decision, Reviewer Verdict, or Human Approval.

## 20. Acceptance checklist

### Promotion

- [ ] The Candidate passes Schema, Integrity, Target, and Source Base checks.
- [ ] Validation, Review, and Candidate bind the same Diff.
- [ ] A Producer cannot accept its own output.
- [ ] Promotion uses the Expected Revision that the Orchestrator read.
- [ ] Pointer, Gates, Phase, Owner, and Handoff use one CAS.
- [ ] A CAS failure preserves the current Accepted Pointer.
- [ ] A Superseded Artifact remains Immutable.

### Durable Knowledge

- [ ] Archive promotes only Accepted Outcomes, Deviations, Risks, and required References.
- [ ] A Requirement or Design change updates OpenSpec or an ADR.
- [ ] Full Trace, raw failures, Tokens, and Secrets stay out of Git.
- [ ] Artifact and Evidence References in the Execution Summary resolve successfully.
- [ ] Commit and Push receive their required Human Approvals separately.
- [ ] Git exposes a verifiable Commit Identity after the write completes.

### Rollback Target

- [ ] The Rollback Basis URI, Checksum, Run, and prior acceptance are verifiable.
- [ ] The Target still satisfies current Requirement, Schema, Dependency, and Security Policy.
- [ ] The workflow records Rollback Basis and Rollback Candidate separately.
- [ ] When Source changed, the workflow does more than Pointer Re-selection.
- [ ] Automation stops when it cannot prove that the Target is usable.

### Compensating Rollback

- [ ] The Rollback Candidate is based on the current Source Base.
- [ ] The new Candidate has its own Diff Hash, Changed Files, and Evidence.
- [ ] Validation covers the original Regression and current integrated behavior.
- [ ] The Reviewer reviews the Rollback Candidate again.
- [ ] Readiness and Human Approval bind the new Target.
- [ ] Current State Revision never moves backward.

### Invalidation and lifecycle

- [ ] The workflow reevaluates Gates, Handoffs, and Approvals after a Target change.
- [ ] An Agent does not replace References in an invalid Handoff and continue execution.
- [ ] The current Phase does not add an undefined `ROLLING_BACK` value.
- [ ] A Terminal Run does not reopen or change in place.
- [ ] A Cross-run Reference has explicit Purpose, Identity, and Integrity Binding.
- [ ] State Recovery and engineering Rollback use different Reason Codes.

### Schema compatibility

- [ ] Current State has no added `promotionRecord`, `rollbackTarget`, or `supersededBy` field.
- [ ] The current Handoff has no undefined Promotion or Rollback fields.
- [ ] Trace, a Host Journal, or an independent Artifact retains Decisions and History first.
- [ ] Existing Blocker, Result, or Trace structures carry Failure Codes.
- [ ] Future Schemas have Version, Fixture, and Migration rules before adoption.

## 21. References

- [Artifact-based Shared State + Structured Handoff reference architecture](./02-reference-architecture.md)
- [Repository Knowledge, Runtime State, Evidence, and Trace layers](./03-runtime-storage-and-retention.md)
- [Security, governance, and long-term maintenance](./05-security-and-maintainability.md)
- [Context Authority and Retrieval Policy](./06-context-authority-and-retrieval-policy.md)
- [Role capability and scope control](./07-role-capability-and-scope-control.md)
- [Mutation, concurrency, and conflict resolution](./08-mutation-concurrency-and-conflict-resolution.md)
- [Agent Platform Operations](../../../context-engineering/docs/03-agent-platform-operations.md)
- [Tool Governance and Evaluation](../../../agent-design/tool-schema-routing/docs/03-tool-governance-and-evaluation.md)
- [Current State Schema](../schemas/current-state.schema.json)
- [Agent Result Schema](../schemas/agent-result.schema.json)
- [Handoff Envelope Schema](../schemas/handoff-envelope.schema.json)
- [Execution Summary Template](../templates/execution-summary.template.md)
- [Workflow Policy Template](../templates/workflow-policy.template.yaml)
