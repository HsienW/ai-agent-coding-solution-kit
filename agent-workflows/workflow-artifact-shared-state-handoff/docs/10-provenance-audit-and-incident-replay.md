# 10 | Provenance audit and incident replay

[English](./10-provenance-audit-and-incident-replay.md) | [繁體中文](./10-provenance-audit-and-incident-replay-zh-TW.md)

This document defines how a multi-Agent Workflow records Provenance, reconstructs an Audit causal chain, and runs a controlled Incident Replay. It applies to Agents, Coordinators, Orchestrators, Reviewers, Runtime Store maintainers, and Human Approvers.

Provenance MUST let an investigator determine which Actor, under which authorization, read which pinned inputs, performed which Action, produced which Artifact and Evidence, and caused a State Transition through a specific Policy and Decision. Incident Replay uses this causal chain to recompute a Decision or rerun Validation while protecting the original Run and external systems.

This document follows three rules:

- Audit retains the stable facts required to reconstruct a Decision. It excludes complete Prompts, private reasoning, and unfiltered output.
- Replay runs in an isolated environment by default and blocks Commit, Push, Deploy, Delete, and external API Write Actions.
- The current Schemas remain unchanged. Additional records belong in Trace, the Host Journal, or an independent Artifact.

## 1. Scope

This document continues the existing specifications:

- [`03-runtime-storage-and-retention.md`](./03-runtime-storage-and-retention.md) defines the storage layers for Durable Knowledge, Runtime State, Evidence, and Trace.
- [`05-security-and-maintainability.md`](./05-security-and-maintainability.md) defines Evidence Provenance, Artifact Integrity, Human Gates, and the Threat Model.
- [`06-context-authority-and-retrieval-policy.md`](./06-context-authority-and-retrieval-policy.md) defines Context Authority, Version, Freshness, Integrity, and Binding.
- [`07-role-capability-and-scope-control.md`](./07-role-capability-and-scope-control.md) defines Runtime Identity, Capability, Policy Version, and Authorization Trace.
- [`08-mutation-concurrency-and-conflict-resolution.md`](./08-mutation-concurrency-and-conflict-resolution.md) defines Mutation Identity, Receipts, Idempotency, and Side-effect Reconciliation.
- [`09-context-promotion-and-rollback.md`](./09-context-promotion-and-rollback.md) defines Promotion, Rollback, Terminal Runs, and Cross-run Binding.

This document covers:

- Provenance for Agent Invocation, Context Retrieval, Tool Action, Artifact Publication, Validation, Review, Approval, and State Transition.
- Audit Reconstruction for authorization, acceptance, rejection, Promotion, Rollback, and external side effects.
- Decision Replay, Validation Replay, Simulation Replay, and Effect Reconciliation.
- Incident Evidence Freeze, Replay Isolation, Result Comparison, Retention, and Redaction.
- Minimum local adoption in `.agent-runtime` and the criteria for moving to a Durable Store.

Other specifications continue to govern:

- Legal Current State Phases and the Transition Table.
- The authorization model for Role, Capability, Path, and Tool Scope.
- Accepted Pointer CAS and the Mutation Critical Section.
- Production Deployment, Database Recovery, Traffic Shift, and organizational Incident Management processes.

## 2. Terms and operation boundaries

### 2.1 Provenance

Provenance is a set of verifiable sources, identities, and causal relationships. It connects inputs, execution, outputs, Evidence, Decisions, and State Mutations.

The Provenance of an Artifact MUST identify at least:

```text
producer identity
changeId / runId / stage
input references and checksums
source base or state revision
output reference and checksum
verification evidence
policy and authorization decision
parent event or decision reference
```

A chronological Log shows an apparent order. Revision, Parent Reference, and explicit causal Edges prove which inputs a Decision used.

### 2.2 Audit Record

An Audit Record retains the stable facts required to reconstruct a Decision. It SHOULD support machine queries and integrity verification, with stricter access and retention rules than Full Trace.

An Audit Record MAY retain a normalized Tool Action, Policy Version, State Revision, Outcome, Reason Code, and Reference. It MUST NOT retain Secrets, Credentials, Session Cookies, replayable Approval Proof, or private model reasoning.

### 2.3 Trace

Trace retains diagnostic detail such as Adapter timing, Retry events, truncated Tool Output, token usage, and internal errors. A team MAY assign Trace a shorter Retention period and delete it independently after preserving the required Audit Records and Evidence.

### 2.4 Evidence

Evidence supports a verifiable claim, for example:

- A Command succeeded in a specified Working Directory.
- A pinned Diff passed a specified test version.
- A Reviewer issued a Verdict for a Candidate with a fixed Checksum.
- The System of Record shows that an external side effect completed.

Evidence MUST bind to the Target that it verifies. A test name or a `passed` label alone does not establish a valid Binding.

### 2.5 Incident Replay

Incident Replay recomputes a Decision or reruns permitted Validation against pinned original inputs, Policy, State Revision, and environment conditions. Replay produces a new Replay Result. The original Artifact, Audit Record, and Terminal Run remain unchanged.

### 2.6 Re-execution and Reconciliation

Re-execution performs an Action with a real effect again. It requires a new Run, Capability, Gate, Idempotency Key, and any required Human Approval.

Reconciliation queries the System of Record to determine whether the original side effect completed. After a timeout or connection failure, the Orchestrator runs Reconciliation before selecting the next route.

## 3. Roles and responsibilities

| Role | Responsibility | Prohibited actions |
|---|---|---|
| Coordinator | Define the Incident question, Scope, Replay Mode, Comparison Policy, and correction direction | Does not modify the original Audit Record or guess missing Provenance |
| Orchestrator | Pin References, create the Replay Manifest, verify integrity, provision isolation, apply Policy, and write the Host Journal | Does not overwrite a Terminal Run or treat Replay as a Retry of an existing Invocation |
| Agent | Read pinned inputs from the Handoff, perform authorized analysis or Validation, and produce a new Result | Does not expand Context or Tool Scope or enable an external Write |
| Reviewer / Auditor | Perform read-only checks of the Provenance Chain, Evidence Binding, and Replay differences | Does not update Current State, the Accepted Pointer, or Approval |
| Runtime Store | Retain Immutable Artifacts, Audit Records, Receipts, and Retention Metadata | Does not allow a Create-only Record to be overwritten in place |
| Human / Incident Operator | Approve sensitive data access, Live Re-execution, external side effects, and high-risk corrections | Does not replace a verifiable Approval Record with verbal consent |

`Incident Operator` names a responsibility during an incident. It does not add a Role to the current Current State Schema. The Runtime Identity still passes the Policy and Capability checks defined in document 07.

## 4. Provenance invariants

Every Workflow Host MUST preserve these invariants:

1. Stable Provenance Records are Append-only. A correction creates a new Record that references the superseded Record.
2. An Artifact Reference pins a URI and Checksum. Different content uses a new URI.
3. Every Agent Result traces back to a Producer, Change, Run, Stage, and required Input References.
4. Validation Evidence binds to the Candidate, Source Base, Command, and execution environment.
5. A Review Verdict binds to the reviewed Candidate and Evidence.
6. A State Transition references an Authorization Decision, Gate Decision, or Mutation Receipt.
7. `current-state.json` stores the current projection and does not carry the complete event history.
8. Timestamps support ordering. Revision, Parent Reference, and explicit Binding establish causal order.
9. Replay produces new Artifacts and Trace records and does not modify the original Run.
10. Replay denies Tools with external side effects by default.
11. A Provenance gap, Checksum mismatch, or missing Policy Version causes the workflow to fail closed.
12. Audit Records exclude Secrets, Credentials, complete sensitive Arguments, and private model reasoning.
13. A Terminal Run remains closed. Follow-up investigation uses new Incident and Replay Identities.
14. When Retention removes a required input, the system reports the gap and does not invent substitute data.

Provenance Records, Audit Decisions, Mutation Receipts, Approvals, and Replay Results MUST NOT use last-write-wins.

## 5. Provenance Graph

### 5.1 Nodes

A Provenance Graph MAY contain these nodes:

| Node | Content |
|---|---|
| Contract | Requirement, Proposal, Design, Task, or Acceptance Criteria |
| State Snapshot | Current State at a specified Revision |
| Handoff | Pinned Role, Stage, Input References, and Transitions |
| Invocation | One Orchestrator-controlled Agent execution |
| Context Resolution | Context, Authority, Version, and Binding that the Agent loaded |
| Tool Action | Tool, Action, Resource, Arguments Hash, and Side-effect Class |
| Artifact | Plan, Implementation, Review, Readiness, or Archive Result |
| Evidence | Command, Diff, test, scan, or external observation result |
| Decision | Authorization, Review, Gate, Promotion, Rollback, or Arbitration |
| Mutation Receipt | Commit result for State, Pointer, Source, or an external side effect |
| Approval | Human authorization bound to a fixed Target, Scope, Revision, and validity period |
| External Observation | External state read from the System of Record |

### 5.2 Causal Edges

The Host SHOULD use a fixed Vocabulary instead of free text for relationships:

| Edge | Meaning |
|---|---|
| `derivedFrom` | An Output was derived from a specified Input |
| `consumed` | An Invocation read a specified Context or Artifact |
| `produced` | An Invocation or Tool Action produced a specified Output |
| `validated` | Evidence verified a specified Target |
| `reviewed` | A Verdict reviewed a specified Candidate |
| `authorizedBy` | A specified Policy or Approval authorized an Action |
| `applied` | A Mutation committed a Decision |
| `observed` | A Record came from a System of Record query |
| `superseded` | A new Record replaced the effective authority of an older Record |
| `compensated` | A new Mutation compensated for an earlier committed effect |
| `replayOf` | A Replay Record points to an original Record |

Each Edge MUST retain source and target References. Similar `createdAt` values do not prove causality.

### 5.3 Minimum closed chain

An Accepted Implementation requires at least this chain:

```text
approved requirement / design
  → handoff
  → authorization decision
  → source base + resolved context
  → implementation candidate
  → validation evidence
  → review verdict
  → gate decision
  → accepted pointer mutation receipt
  → current state revision
```

Audit MUST report each missing required node. An Auditor cannot infer valid intermediate Gates from a successful final Current State.

## 6. Minimum Provenance Record

The following YAML is a conceptual format for the minimum Host Journal data. It is not part of the current JSON Schemas:

```yaml
schemaVersion: 0.1.0-draft
eventId: evt-review-004
eventType: review-decision
changeId: feature-example
runId: run-004
stage: review-result

actor:
  actorId: reviewer-worker-02
  role: reviewer

binding:
  stateRevision: 8
  policyVersion: reviewer-policy-1.2.0
  handoffRef: runtime://feature-example/run-004/handoffs/review-004.json
  parentEventIds:
    - evt-validation-004

inputs:
  - uri: artifact://feature-example/run-004/implementation-v2.json
    sha256: xxxx
  - uri: evidence://feature-example/run-004/validation-v2.json
    sha256: xxxx

decision:
  outcome: approve
  reasonCode: REVIEW_CRITERIA_SATISFIED

outputRef: artifact://feature-example/run-004/review-v2.json
resultHash: sha256:xxxx
occurredAt: 2026-08-14T10:12:03Z
recordedAt: 2026-08-14T10:12:04Z
traceId: trace-review-004
```

### 6.1 Identity

Keep these Identities separate:

- `eventId`: one Provenance Event.
- `traceId`: a diagnostic correlation across several Events.
- `changeId`: an engineering change scope.
- `runId`: one Workflow Run.
- `invocationId`: one Agent Invocation.
- `mutationId`: one controlled Mutation.
- `incidentId`: one incident investigation.
- `replayId`: one controlled Replay.

Reusing one ID prevents the Host from distinguishing Retry, Cross-run work, and multiple Replay attempts.

### 6.2 Tool Action Record

A Tool Action SHOULD also retain:

```text
toolId
toolVersion
action
normalizedResource
normalizedArgumentsHash
capability
policyVersion
environmentRef
sideEffectClass
idempotencyKey, when required
beforeRef / afterRef, when applicable
decision and reasonCode
```

The Host MUST normalize and redact sensitive Arguments before computing the Hash. The Audit Store MUST NOT retain content that can reveal a Secret.

## 7. Lifecycle capture points

The Orchestrator SHOULD record Provenance at these points:

| Capture point | Required record |
|---|---|
| Handoff Issued | From, To, Stage, pinned Input References, Transitions, and expiry |
| Invocation Authorized | Runtime Identity, Capability, Resource, Policy Version, State Revision, and Decision |
| Context Resolved | Loaded URI, Checksum, Authority, Freshness, and Binding |
| Tool Action Started | Tool, Action, Arguments Hash, Side-effect Class, and Idempotency Key |
| Tool Action Completed | Status, Output Ref, Before / After Ref, and System of Record Ref |
| Artifact Published | Producer, Input Ref, Output URI, Checksum, and Schema Version |
| Validation Completed | Command, Working Directory, Target, Exit Code, and Evidence Ref |
| Review Decided | Candidate Ref, Evidence Ref, Verdict, and Reason Code |
| Gate Decided | Gate Type, Target Binding, Decision Owner, and Approval Ref |
| State Committed | Expected Revision, Applied Revision, Mutation Receipt, and Accepted Pointer |
| External Effect Observed | Query Target, Observation Time, Observed Identity, and Result Ref |

The Host MUST publish an Output Artifact before committing a State Mutation that references it. If a crash occurs between these operations, the Recovery flow in document 08 identifies an Orphan Candidate or rebuilds State.

## 8. Audit Reconstruction

### 8.1 Investigation questions

An Audit Bundle SHOULD let a Reviewer or Incident Operator answer:

1. Which Runtime Identity initiated the Action?
2. Which Role, Capability, and Effective Scope did it have at that time?
3. Which Policy Version and State Revision did the Orchestrator use for authorization?
4. Which Context did the Agent read, excluding entries listed in the Handoff that it never loaded?
5. Which Source Base, Diff, and Input Checksum bind to the Candidate?
6. Do Validation and Review point to the same Candidate?
7. Which Gate or Approval permitted the Accepted Pointer or external state to change?
8. Did the Mutation commit? What result does the System of Record show?
9. Which inputs, environment conditions, and results differ between the original execution and Replay?

### 8.2 Audit Bundle

An Audit Bundle is a Reference Manifest for the incident scope. It does not copy every record:

```yaml
schemaVersion: 0.1.0-draft
bundleId: audit-bundle-REG-204-01
incidentId: REG-204
sourceChangeId: feature-example
sourceRunId: run-004

scope:
  fromEventId: evt-implementation-004
  toEventId: evt-pointer-accept-004

records:
  - ref: runtime://feature-example/host-journal/provenance/evt-implementation-004.json
    sha256: xxxx
  - ref: runtime://feature-example/host-journal/provenance/evt-review-004.json
    sha256: xxxx

missingRecords: []
integrityStatus: verified
createdAt: 2026-08-14T11:00:00Z
```

The Bundle includes only Records with a causal relationship to the investigation question. Add Full Trace only to locate a gap, reconstruct a Tool Failure, or clarify the timeline.

### 8.3 Audit completion

An Audit conclusion MUST include:

- The investigation scope and pinned original References.
- Whether the Provenance Chain closes.
- Integrity and Retention gaps.
- Confirmed facts, unresolved items, and Evidence sources.
- The earliest provable node that caused an incorrect Decision or Mutation.
- The recommended Replay, Rollback, Correction, or manual route.

When Evidence is missing, the conclusion can only report `insufficient-evidence`. The Auditor MUST keep inference separate from confirmed fact.

## 9. Integrity, ordering, and clocks

### 9.1 Content Integrity

Stable Records and Artifacts SHOULD use Checksums. The Host verifies:

```text
reference exists
schema or record format is readable
checksum matches
changeId / runId binding matches
producer identity is allowed
parent references exist
retention state permits use
```

A Checksum proves that content has not changed. It does not prove Producer identity, authorization, or record time on its own.

### 9.2 Ordering

Use these ordering sources in priority order:

1. Current State `revision` and Mutation `committedRevision`.
2. `parentEventIds` and explicit Provenance Edges.
3. A monotonic sequence or Fencing Token supplied by the Tool or Host.
4. `occurredAt` and `recordedAt`.

Clocks on separate Hosts can drift. An Incident Timeline MAY display Timestamps, but MUST NOT use Timestamps alone to assert causal order.

### 9.3 Tamper Evidence

A local implementation MAY use Create-only Files, Checksums, and strict Directory Permissions. Adopt Hash Chains, Digital Signatures, Merkle Trees, or WORM Storage when any of these requirements applies:

- The system must prove that no Host Journal entry was deleted or reordered.
- Audit Records must be exchanged across organizations.
- Compliance requires an independent timestamp or signature.
- Multiple Hosts require a global sequence.

## 10. Retention, Redaction, and access control

### 10.1 Retention classes

| Data | Recommended retention rule |
|---|---|
| Current State | Retain the Latest Snapshot; rebuild it from the Journal when required |
| Minimal Audit Record | Retain according to engineering risk, compliance, and investigation needs |
| Approval / Mutation Receipt | Cover at least the traceability period of the Change and external effect |
| Evidence | Cover Review, Readiness, Rollback, and the incident observation period |
| Full Trace | Use a shorter period; delete after preserving required Audit References |
| Replay Artifact | Retain until Incident closure and correction verification finish |

The Retention Policy MUST have a Version. After Incident declaration, the Orchestrator MAY place a Legal Hold or Incident Hold on related Records. Only an authorized Operator may release a Hold.

### 10.2 Redaction

Audit, Trace, and Replay Bundles MUST exclude:

- API Keys, Tokens, Cookies, Passwords, and Private Keys.
- Complete contents of `.env`, a Credential Store, or a Secret Manager.
- Replayable Approval Proof.
- Unredacted personal data and files outside the task scope.
- Private model reasoning and provider-internal reasoning data.

When an investigation must prove that a sensitive value participated in an Action, retain a Redacted Identifier, Secret Version, or one-way Hash. The system MUST also restrict any guessable input space for that Hash.

### 10.3 Access Scope

Audit permission does not grant read access to the entire Runtime directory. The Orchestrator still applies the Effective Scope from document 07:

```text
platform policy
∩ adapter policy
∩ role policy
∩ incident scope
∩ handoff scope
∩ approval scope
```

A Reviewer MAY receive Evidence required for the investigation. It cannot enumerate other Changes, the Secret Store, or unrelated Full Trace.

## 11. Incident declaration and Evidence Freeze

### 11.1 Declare the Incident

The Coordinator or an authorized Incident Operator creates a new `incidentId` and records:

```text
incident question
affected changeId / runId
observed symptom
observation source
suspected time range
required retention hold
investigation owner
```

The Incident Record MUST NOT be written back into the original Terminal Run. It uses Cross-run References to the original Change, Run, Artifacts, and State Revision.

### 11.2 Freeze References

Before Replay, the Orchestrator pins:

- The original Current State Revision or a verifiable Snapshot.
- Handoff, Agent Result, Evidence, Review, and Approval References.
- Source Commit, Worktree Base, Diff Hash, and Accepted Pointer.
- Policy, Adapter, Tool Registry, and environment versions.
- The System of Record Reference for the external observation.

Freeze protects References and Retention. It does not pause the whole Repository. When product work must continue, the Coordinator MAY open a new Change. The two flows require explicit Cross-run Binding.

### 11.3 Freeze failure

If a required Record is missing, expired, or has the wrong Checksum, the Orchestrator stops Replay and returns a fixed Failure Code. The Operator MAY narrow the investigation scope or accept an `insufficient-evidence` conclusion. It MUST NOT substitute the Latest Artifact for the missing original version.

## 12. Replay modes

### 12.1 Decision Replay

Decision Replay recomputes the following against pinned State Revision, Policy Version, Context Manifest, and Artifact References:

- Authorization Decision.
- Context Authority and Freshness evaluation.
- Mechanical Eligibility for Review and Gates.
- Promotion or Rollback Preconditions.
- Compliance with the State Transition Table.

This mode does not call a Tool with side effects or update Current State. Use it to investigate incorrect routing, stale Evidence, Target Binding Mismatch, and Policy Drift.

### 12.2 Validation Replay

Validation Replay reruns tests, Lint, Build, Scanners, or read-only queries in an isolated Worktree or Sandbox. The Orchestrator MUST pin the Source Base, dependency lock, Command, Working Directory, and environment version.

Validation Replay produces new Evidence and links it to the original Evidence with `replayOf`. The new Evidence cannot replace the Validation Result of the original Run directly.

### 12.3 Simulation Replay

Simulation Replay replaces an external Write with a Stub, Recorded Response, read-only Mirror, or Dry-run Adapter. It applies to:

- Reconstructing Tool Routing and Arguments Normalization.
- Verifying Idempotency Key, Scope, and Approval evaluation.
- Comparing Agent behavior after an external Response changes.

A Recorded Response MUST retain retrieval time, source, integrity, and Redaction state. Expired data MAY reconstruct a past Decision but cannot serve as current authoritative Context.

### 12.4 Effect Reconciliation

Effect Reconciliation performs a controlled query against the System of Record:

```text
query by idempotency key or external reference
verify target identity
compare before / after reference
classify committed, absent, partial, or unknown
record observation time and source
```

When the query returns `unknown`, the Orchestrator returns `SIDE_EFFECT_OUTCOME_UNKNOWN` and stops automation.

### 12.5 Live Re-execution

When Commit, Push, Deploy, Delete, or an external Write must run again, the Coordinator creates a new Change, Run, or governed compensation flow. The Orchestrator reevaluates Capability, Scope, Gates, Approval, Idempotency, and Current State Revision.

Live Re-execution MUST NOT submit an external effect under a Replay Identity or reuse an invalidated Approval.

## 13. Replay Manifest

The Replay Manifest pins the inputs and execution limits for the investigation. The following YAML is a conceptual format and does not belong to the current Schemas:

```yaml
schemaVersion: 0.1.0-draft
incidentId: REG-204
replayId: replay-REG-204-01
sourceChangeId: feature-example
sourceRunId: run-004
mode: decision-replay

sourceBinding:
  stateRevision: 9
  sourceCommit: git:def456
  policyVersion: promotion-policy-1.0.0
  adapterVersion: local-agent-adapter-0.8.0
  environmentRef: runtime://feature-example/incidents/REG-204/environment.json

requiredRefs:
  - uri: artifact://feature-example/run-004/implementation-v2.json
    sha256: xxxx
  - uri: evidence://feature-example/run-004/validation-v2.json
    sha256: xxxx
  - uri: artifact://feature-example/run-004/review-v2.json
    sha256: xxxx

blockedSideEffects:
  - git.commit
  - git.push
  - deployment.write
  - external.delete

comparisonPolicy:
  mode: semantic-and-binding
  allowedDrift:
    - timestamp
    - traceId

createdBy: incident-coordinator-01
createdAt: 2026-08-14T11:20:00Z
traceId: trace-replay-REG-204-01
```

### 13.1 Manifest validation

Before Invocation, the Orchestrator MUST check:

1. The Binding among `incidentId`, Source Change, and Source Run.
2. Whether every Required Reference exists and matches its Checksum.
3. Whether the Policy, Adapter, Tool, and environment versions are available.
4. Whether the Replay Mode permits the Action specified by the Handoff.
5. Whether the Sandbox and Adapter enforce the Side-effect Denylist.
6. Whether the Actor has Incident Scope and data access permission.

An Agent Invocation cannot begin when any check fails.

## 14. Environment Binding

### 14.1 Required Binding

Replay reproducibility depends on pinned environment data:

| Dimension | Recommended record |
|---|---|
| Source | Commit, Branch or Worktree Base, Diff Hash, and Submodule Revision |
| Dependency | Lockfile Hash, Package or Container Digest, and Runtime Version |
| Agent | Adapter Version, Role, Stage, and Prompt or Template Hash |
| Model | Provider, Model Identifier, available Version, and inference settings |
| Tool | Tool ID, Schema Version, and Binary or Service Version |
| Policy | Authorization, Retrieval, Mutation, and Promotion Policy Versions |
| Runtime | OS, Architecture, Locale, Timezone, and Environment Allowlist |
| External Context | Source, Query, Observed At, Freshness, and Recorded Response Ref |

When an exact version is unavailable, the Manifest MUST state the gap and its effect. The Orchestrator cannot present the current version as the original version.

### 14.2 Determinism boundary

The same Prompt, Model Identifier, and parameters do not guarantee byte-identical LLM output. External APIs, search indexes, package Registries, clocks, and random sources also produce differences.

The Replay Comparison Policy SHOULD match the investigation question:

- Byte-level: compare Checksums, pure function outputs, and fixed Fixtures.
- Structural: compare Schemas, fields, References, and Reason Codes.
- Semantic: compare Verdicts, risk classification, and recommended routing.
- Evidence-based: compare tests, Source Identity, and System of Record results.

Byte-for-byte replay requires the Host to pin every nondeterministic source. A source that cannot be pinned belongs in the Reproducibility Limitation.

## 15. Replay flow

```text
declare incident
  → freeze source references
  → build audit bundle
  → verify integrity and retention
  → select replay mode
  → create replay manifest
  → authorize replay invocation
  → prepare isolated environment
  → execute replay
  → publish replay result
  → compare original and replay
  → classify divergence
  → route correction / rollback / closure
```

### 15.1 Prepare

The Orchestrator:

1. Creates `incidentId` and `replayId`.
2. Pins the Source Run, State Revision, Policy Version, and Required References.
3. Verifies the Audit Chain, Checksums, Retention, and Access Scope.
4. Selects the Replay Mode and Comparison Policy.
5. Creates an isolated Worktree, Container, or read-only Sandbox.
6. Applies the Side-effect Denylist in the Adapter and Tool Gateway.

### 15.2 Execute

The Agent receives only the Input References specified by the Replay Handoff. It MAY produce analysis, Validation Evidence, and a Replay Result. It MUST NOT:

- Modify an original Artifact, Current State, or Host Journal.
- Read Context outside the Manifest.
- Use an alternative Tool to bypass a denied Action.
- Declare Replay Evidence to be Accepted Evidence for the original Run.
- Call Commit, Push, Deploy, Delete, or an external Write.

### 15.3 Compare

The Reviewer or Auditor compares the following according to the Comparison Policy:

```text
input identity
policy and environment identity
authorization outcome
tool routing and normalized arguments
artifact structure and checksum
validation and review binding
state transition eligibility
external observation
```

The comparison MUST identify the node at which a difference occurred and whether that difference can change the original Decision.

### 15.4 Route

| Result | Route |
|---|---|
| Original Decision is reproducible and the result matches | Close Replay and retain the Audit Summary |
| Result matches with nonmaterial Drift | Record the Drift and Reproducibility Limitation |
| Evidence Binding is wrong | Use `INCOMPLETE` or `FAILED`; Coordinator creates a correction flow |
| Requirement, Design, or Policy Conflict | `NEEDS_COORDINATOR_ARBITRATION` |
| Source or Accepted Pointer requires correction | Create a new Change or Run and follow document 09 for Promotion or Rollback |
| External Write is required | Pause and request new Human Approval |
| Provenance cannot close | Return a fixed Failure Code and do not assert root cause |

A Replay Result does not update the original Current State. A new correction Run updates its own State through the legal Transition Table.

## 16. Result comparison and Drift

### 16.1 Result classification

| Classification | Definition | Next action |
|---|---|---|
| `exact-match` | Pinned fields and content match exactly | Record success |
| `semantically-equivalent` | Text or noncritical fields differ while Decision and Binding match | Record permitted differences |
| `acceptable-drift` | Environment or time differences listed by the Comparison Policy | Retain Drift Detail |
| `outcome-diverged` | Verdict, Evidence, Routing, or Eligibility changed | Coordinator determines correction scope |
| `non-reproducible` | A required version or nondeterministic source cannot be reconstructed | Record the limitation and do not claim equivalence |
| `integrity-failure` | Checksum, Schema, or Identity does not match | Isolate the data and stop |
| `insufficient-evidence` | The Provenance Chain lacks a required Record | Obtain data or narrow the conclusion |

### 16.2 Replay Result

Conceptual Replay Result:

```yaml
schemaVersion: 0.1.0-draft
replayId: replay-REG-204-01
replayOf:
  changeId: feature-example
  runId: run-004
  decisionRef: runtime://feature-example/host-journal/provenance/evt-review-004.json

mode: decision-replay
status: completed
classification: outcome-diverged

comparison:
  inputIdentity: mismatch
  policyIdentity: match
  environmentIdentity: match
  originalOutcome: approve
  replayOutcome: incomplete

findings:
  - code: PROMOTION_BINDING_MISMATCH
    originalEvidenceRef: evidence://feature-example/run-004/validation-v2.json
    expectedTargetRef: artifact://feature-example/run-004/implementation-v2.json
    actualTargetRef: artifact://feature-example/run-003/implementation-v1.json

nextAllowedAction: coordinator-arbitration
createdAt: 2026-08-14T11:42:00Z
traceId: trace-replay-REG-204-01
```

`completed` and `classification` belong to the conceptual Replay Record. When the current Agent Result carries this information, the Producer MUST use a Status allowed by the Schema and place the classification in the corresponding Payload, Summary, Risk, or Trace.

## 17. External Side-effect Reconciliation

### 17.1 Blind resubmission is prohibited

Replay MUST NOT run these Actions automatically:

- Git Commit, Push, Force Push, Tag, or Release Publication.
- Deployment, Rollback, Traffic Shift, or Feature Flag Write.
- Database Write, Migration, Delete, or data repair.
- Issue, Message, Email, Payment, or third-party API Write.
- Credential Rotation, Permission Change, or Secret Update.

The Tool name does not affect the decision. If an Agent uses Shell, a Script Runner, or another Adapter to invoke the same external effect, Policy still denies the Action.

### 17.2 Unknown Outcome

When the original Action times out or loses its connection:

1. Read the Mutation Receipt, Idempotency Key, and External Reference.
2. Query the System of Record through a read-only operation.
3. Compare Target Identity, Before Ref, and After Ref.
4. Classify the result as `committed`, `absent`, `partial`, or `unknown`.
5. Retain a new External Observation Record.

An `unknown` result MUST return `SIDE_EFFECT_OUTCOME_UNKNOWN`. The Orchestrator stops Retry and Replay until the Coordinator or a Human decides the next action.

### 17.3 Compensating Action

If the workflow must undo a committed external effect, the Coordinator creates a Compensating Action. It uses a new Mutation Identity, Idempotency Key, Approval, and Receipt, with a `compensated` Edge pointing to the original Mutation.

A Compensating Action is a new governed change and sits outside the read-only scope of Replay.

## 18. Terminal Runs and Cross-run work

### 18.1 Terminal Run

A Run with `terminal: true` remains closed. After an Incident, the system may only:

- Read original Artifacts, Evidence, Trace, State Snapshots, and Receipts.
- Create an external Incident Record, Audit Bundle, and Replay Result.
- Point to the original Run through References.

The system MUST NOT modify the original Result, Gates, Accepted Pointer, Handoff, Revision History, or Execution Summary.

### 18.2 Cross-run Binding

When a Replay or correction Run references the original Run, it pins at least:

```text
source changeId / runId
target incidentId / replayId / runId
purpose
artifact or decision URI / checksum
source state revision
source commit
authority level
retention availability
```

A Cross-run Reference is Diagnostic or Incident Input in the new Run. It does not gain Accepted Authority or inherit the original Approval.

### 18.3 Follow-up correction

After Replay confirms an error, the Coordinator creates one of these paths:

- A new Implementation or Validation Run.
- A new Context Promotion or Compensating Rollback.
- A Durable Knowledge Correction.
- An external correction that requires Human Approval.

The new Run uses current valid Requirements, Policy, and State. Original Incident data supplies diagnostic and correction evidence only.

## 19. Fixed failure codes and routing

### 19.1 Provenance Failure

| Code | Condition | Safe route |
|---|---|---|
| `PROVENANCE_RECORD_MISSING` | A required Event, Decision, Receipt, or Evidence does not exist | `INCOMPLETE`; list the missing References |
| `PROVENANCE_CHAIN_BROKEN` | A Parent Reference or required Edge cannot close | `INCOMPLETE`; stop root-cause determination |
| `PROVENANCE_INTEGRITY_FAILED` | Checksum, Schema, Producer, or Binding does not match | `FAILED`; isolate the Record |
| `PROVENANCE_SCOPE_DENIED` | The Actor cannot read required data | Pause and request an Operator with the required Scope |
| `PROVENANCE_RETENTION_EXPIRED` | Required Trace or Evidence expired under Policy | `INCOMPLETE`; record the investigation limit |
| `PROVENANCE_POLICY_VERSION_MISSING` | The original Authorization or Retrieval Policy is unavailable | Stop Decision Replay |

### 19.2 Replay Failure

| Code | Condition | Safe route |
|---|---|---|
| `REPLAY_INPUT_UNAVAILABLE` | An input pinned by the Manifest is missing or unreadable | `INCOMPLETE` |
| `REPLAY_ENVIRONMENT_MISMATCH` | The Host cannot create a compatible Source, Dependency, Tool, or Runtime environment | Stop and record the difference |
| `REPLAY_POLICY_VERSION_UNAVAILABLE` | The original Policy cannot be loaded | Stop Decision Replay |
| `REPLAY_SIDE_EFFECT_BLOCKED` | Replay attempts an external Write | `FAILED`; terminate the Invocation |
| `REPLAY_OUTCOME_DIVERGED` | The result exceeds the differences allowed by the Comparison Policy | Route to the Coordinator |
| `REPLAY_AUTHORIZATION_REQUIRED` | Sensitive data or Live Re-execution is required | Pause and request Approval |
| `REPLAY_MANIFEST_INVALID` | Manifest Binding, Mode, or References are incomplete | `INCOMPLETE`; create a corrected Manifest |

### 19.3 Reused codes

The following conditions use existing codes:

| Code | Source and use |
|---|---|
| `SIDE_EFFECT_OUTCOME_UNKNOWN` | Document 08: an external effect cannot be confirmed; Reconcile first |
| `STATE_REVISION_CONFLICT` | Document 08: Replay or correction uses a stale State precondition |
| `TERMINAL_RUN_IMMUTABLE` | Document 09: an operation attempts to modify a closed Run |
| `ROLLBACK_TARGET_INTEGRITY_FAILED` | Document 09: Rollback Basis integrity fails |
| Context Integrity / Binding Code | Document 06: Context URI, Checksum, Authority, or Target does not match |

### 19.4 Failure Result

A Failure Result or Trace retains at least:

```text
code
incidentId
replayId
source changeId / runId
stage
actorId
recordRef or inputRef
expectedIdentity
actualIdentity
comparisonPolicy
nextAllowedAction
traceId
```

A current Agent Result SHOULD use the Schema-defined `blocked`, `failed`, or `incomplete` Status and place the fixed Code in a verifiable Payload, Blocker, Risk, Summary, or Trace. Current State uses only the current Phase Enum.

## 20. Handoff and current Schema compatibility

### 20.1 Using the current Handoff

[`handoff-envelope.schema.json`](../schemas/handoff-envelope.schema.json) does not define `incidentId`, `replayId`, `replayMode`, `sourceStateRevision`, or `blockedSideEffects`, and it uses `additionalProperties: false`. A current Handoff MUST NOT add these fields directly.

The local workflow uses:

- `requiredInputRefs` to pin the Replay Manifest, Audit Bundle, Source Artifact, Evidence, and Policy References.
- Artifact Reference `uri` and `sha256` to pin content.
- `description` to identify a Reference as Incident Input, Replay Manifest, Original Evidence, or Recorded Response.
- `reason` to state the investigation question, Replay Mode, and prohibited Side-effect types.
- `acceptanceCriteriaRefs` to point to the Comparison Policy and Audit completion criteria.
- `onSuccess`, `onFailure`, and `onConflict` with legal Phases and Owners only.
- Orchestrator Invocation Metadata to carry Incident Scope, Replay ID, and Policy Version.
- Trace or Host Journal to retain Provenance Events, the Audit Bundle, and Replay Results.

### 20.2 Handoff fragment

The following fragment uses only fields from the current Handoff Schema:

```json
{
  "from": "coordinator",
  "to": "reviewer",
  "stage": "review-result",
  "reason": "Decision replay for REG-204; read-only investigation; external writes are denied by the adapter policy",
  "requiredInputRefs": [
    {
      "refId": "replay-manifest-REG-204-01",
      "type": "other",
      "uri": "runtime://feature-example/incidents/REG-204/replay-manifest-01.json",
      "sha256": "xxxx",
      "description": "Incident replay manifest; host-controlled input"
    },
    {
      "refId": "audit-bundle-REG-204-01",
      "type": "evidence",
      "uri": "evidence://feature-example/incidents/REG-204/audit-bundle-01.json",
      "sha256": "xxxx",
      "description": "Verified provenance references for the source run"
    }
  ]
}
```

`reason` provides a human-readable description. The Adapter and Policy enforce the external Write block. Natural-language restrictions cannot be the only safety boundary.

### 20.3 Agent Result

[`agent-result.schema.json`](../schemas/agent-result.schema.json) already provides:

- Producer, Change ID, Run ID, Stage, and Created Time.
- `inputRefs`, `outputRefs`, and Artifact Checksum.
- Verification, Risk, Summary, Payload, and Next Handoff.

It does not define `traceId`, `parentEventIds`, `replayOf`, `policyVersion`, or `environmentRef`. Store these values in the Host Journal or Trace, or create an independent Artifact and point to it through a current Reference.

### 20.4 Current State

[`current-state.schema.json`](../schemas/current-state.schema.json) retains the current Phase, Owner, Revision, Accepted Artifacts, Gates, Blockers, and Next Action. It does not retain the complete Provenance Graph or Replay History.

The current Phase Enum does not include `INCIDENT_REPLAYING`, `AUDITING`, or `RECONCILING`. Incident work runs in an independent Host Workflow. When the main Run must expose a blocker, the Orchestrator uses legal Phases such as `INCOMPLETE`, `FAILED`, or `NEEDS_COORDINATOR_ARBITRATION` and includes the Incident Reference.

### 20.5 Conceptual Records

The Provenance Record, Audit Bundle, Replay Manifest, and Replay Result in this document are conceptual formats. Before treating them as exchangeable assets, a team SHOULD create:

```text
schemas/provenance-event.schema.json
schemas/audit-bundle.schema.json
schemas/replay-manifest.schema.json
schemas/replay-result.schema.json
templates/provenance-policy.template.yaml
examples/incident-decision-replay/
examples/validation-replay/
```

Each new asset requires a Schema Version, Migration, Retention, Signature, and Compatibility Policy before adoption.

## 21. Complete local example

### 21.1 Original Run

`feature-example` Run `run-004` produces implementation v2:

```text
implementation v2
  → validation reported pass
  → reviewer approved
  → readiness passed
  → accepted pointer updated to v2 at revision 9
  → human approved deployment
```

Production later reports Regression `REG-204`. An external observation shows that v2 returns an incorrect result for a specific input.

### 21.2 Evidence Freeze

The Coordinator declares `REG-204`. The Orchestrator pins:

```text
source run: run-004
state revision: 9
source commit: def456
implementation v2 ref / checksum
validation v2 ref / checksum
review v2 ref / checksum
readiness and approval refs
accepted pointer mutation receipt
deployment external reference
```

The original Run is Terminal. The Orchestrator does not modify any file in it.

### 21.3 Audit Reconstruction

The Auditor checks the Binding along the Provenance Chain:

```text
implementation v2
  sha256: aaaa...

validation-v2.json
  targetRef: implementation v1
  targetSha256: 1111...

review-v2.json
  candidateRef: implementation v2
  validationRef: validation-v2.json
```

The Validation Evidence binds to v1, while Review treated it as Evidence for v2. A Current State that shows v2 as Accepted cannot fill this intermediate Binding gap.

### 21.4 Decision Replay

The Orchestrator creates `replay-REG-204-01`, loads the original Policy and Revision, and blocks every external Write. Decision Replay reevaluates Promotion Eligibility:

```text
expected validation target: implementation v2 / aaaa...
actual validation target:   implementation v1 / 1111...
outcome: incomplete
code: PROMOTION_BINDING_MISMATCH
```

The Replay Result receives the `outcome-diverged` classification. It points to the original Review Decision and does not modify the original Verdict.

### 21.5 Validation Replay

The Coordinator requests a second Replay. In an isolated Worktree pinned to `def456`, the Orchestrator runs the original Validation Command and produces:

```text
replay evidence target: implementation v2 / aaaa...
regression test: failed
side effects: none
classification: outcome-diverged
```

The new Evidence supports the Incident investigation only. It cannot replace the original Evidence from `run-004` directly.

### 21.6 Correction and Rollback

The Coordinator creates a new correction Change from the Audit and Replay Results. If the team chooses to revert Accepted Context, it follows document 09 to create a Rollback Basis, Rollback Candidate, Validation, Review, Gates, and a new Mutation Receipt.

Production Rollback requires new Human Approval and an external Side-effect Receipt. Incident Replay does not perform that operation.

### 21.7 Closure

The Incident Summary retains:

- The Provenance gap and earliest provable incorrect Decision.
- References and comparison results for Decision Replay and Validation Replay.
- The correction Change, Rollback, or Durable Knowledge Correction Reference.
- Conditions for releasing the Retention Hold.
- Follow-up Policy, Schema, test, or monitoring changes.

## 22. Minimum local adoption

### 22.1 Optional local directory

The first version MAY add a Host-only area to the Git-ignored Runtime:

```text
.agent-runtime/<change-id>/
├─ current-state.json
├─ artifacts/
├─ evidence/
├─ host-journal/
│  ├─ provenance/
│  │  └─ <event-id>.json
│  └─ mutation-receipts/
└─ incidents/
   └─ <incident-id>/
      ├─ replay-manifest.json
      ├─ audit-bundle.json
      ├─ environment.json
      └─ replay-results/
```

The Orchestrator or Runtime Host controls `host-journal` and `incidents`. An Agent reads authorized content only through Handoff References and cannot enumerate or modify the whole directory.

### 22.2 Minimum Host interface

A local Orchestrator provides at least:

```text
appendProvenanceEvent(record)
verifyProvenanceChain(scope)
buildAuditBundle(incidentId, refs)
createReplayManifest(mode, sourceRefs, policy)
prepareIsolatedReplay(manifest)
publishReplayResult(result)
reconcileExternalEffect(idempotencyKey, targetRef)
applyRetentionHold(incidentId, refs)
```

All writes use Create-only or Atomic Replace. The Host provides no Update interface for original Artifacts, Terminal Runs, or existing Journal Records.

### 22.3 Minimum adoption order

1. Make Agent Result, Handoff, and Evidence pin URI, Checksum, Change ID, and Run ID.
2. Retain Authorization Decisions, Tool Actions, Mutation Receipts, and State Revisions in the Host Journal.
3. Create a script that exports an Audit Bundle from References.
4. Support Decision Replay and Validation Replay first.
5. Apply an external Write Denylist to Replay Invocations in the Adapter.
6. Define Retention Policies for Audit, Trace, Evidence, and Incident Hold.
7. Verify the full flow with a Fixture that contains a known Binding Error.

## 23. Durable Store adoption criteria

A team SHOULD evaluate an Append-only, indexed, CAS-capable, or tamper-evident Durable Store when any of these conditions applies:

- Multiple Orchestrator Processes or Workers handle the same Change.
- Operators need complete Lineage queries across Hosts, Repositories, or environments.
- Audit requirements call for reconstruction of every Authorization, Gate, and State Transition.
- Compliance requires WORM, Digital Signatures, independent Timestamps, or Legal Hold.
- External side effects require a reliable Outbox or Inbox and transactional Receipts.
- Incident volume makes local files difficult to index, retain, or partition by permission.
- Runtime resides on a Network Filesystem that cannot provide reliable Create-only and Atomicity guarantees.

The upgraded Store MUST preserve these semantics:

```text
current state is a projection
artifact is immutable
provenance uses explicit references
decision binds policy and state revision
replay writes new records
external effects require reconciliation and approval
terminal run remains immutable
```

For a single machine, Single Writer, low incident volume, and no compliance requirement, a local Host Journal supports basic investigation. Verify the causal chain and Replay boundaries before adding infrastructure.

## 24. Acceptance checklist

### Provenance Coverage

- [ ] Agent Result traces to a Producer, Change ID, Run ID, and Stage.
- [ ] Input and Output References use pinned URIs and Checksums.
- [ ] Validation, Review, Gates, and the Accepted Pointer point to the same Target Identity.
- [ ] A State Transition traces to a Decision, Expected Revision, and Mutation Receipt.
- [ ] Parent References or Provenance Edges establish causal order.
- [ ] Current State stores only the current projection and does not carry complete history.

### Audit Integrity

- [ ] Stable Audit Records are Append-only.
- [ ] The Audit Bundle contains only References and Integrity Status within the investigation scope.
- [ ] Timestamp is not the sole source of causal ordering.
- [ ] A missing Record returns a fixed Code.
- [ ] The Auditor separates confirmed facts, inference, and data gaps.
- [ ] Producer, Policy Version, State Revision, and Decision Owner can be reconstructed.

### Replay Isolation

- [ ] Each Replay has an independent `incidentId` and `replayId`.
- [ ] The Replay Manifest pins Source, Policy, Environment, and Required References.
- [ ] Replay uses an isolated Worktree, Container, or read-only Sandbox.
- [ ] A Replay Result uses a new URI and points to the original Record.
- [ ] Replay does not modify original Artifacts, Current State, or a Terminal Run.
- [ ] The Comparison Policy distinguishes Exact, Semantic, Drift, and Non-reproducible results.

### Side-effect Safety

- [ ] The Adapter or Tool Gateway blocks Commit, Push, Deploy, Delete, and external Write during Replay.
- [ ] An alternative Tool path cannot bypass Action authorization.
- [ ] Unknown Side-effect Outcome triggers a System of Record query first.
- [ ] A Side-effect Mutation has an Idempotency Key, Before or After Ref, and Receipt.
- [ ] Live Re-execution uses a new Run, Gates, and Human Approval.
- [ ] A Compensating Action points to the original Mutation and produces a new Receipt.

### Privacy and Retention

- [ ] Audit Records exclude Secrets, Credentials, replayable Approval Proof, and private model reasoning.
- [ ] Sensitive Arguments are normalized and redacted.
- [ ] Audit, Evidence, Trace, and Replay Artifacts have separate Retention Classes.
- [ ] Incident Hold has authorization, an expiry, and release conditions.
- [ ] Audit Scope does not grant enumeration access to the entire Runtime Store.
- [ ] Expired data produces a recorded limitation and is not replaced with the Latest Artifact.

### Lifecycle and Schema compatibility

- [ ] A Terminal Run remains closed, and incident investigation uses Cross-run References.
- [ ] Replay Evidence does not become Accepted Evidence for the original Run directly.
- [ ] Correction, Rollback, and external Write create a new Change or Run.
- [ ] The current Handoff contains no undefined Incident or Replay fields.
- [ ] Current State contains no undefined Audit or Replay Phase.
- [ ] Each conceptual Record has a Schema Version and Migration Policy before adoption.

## 25. References

- [Artifact-based Shared State and Structured Handoff reference architecture](./02-reference-architecture.md)
- [Repository Knowledge, Runtime State, Evidence, and Trace layers](./03-runtime-storage-and-retention.md)
- [Security, governance, and long-term maintenance](./05-security-and-maintainability.md)
- [Context Authority and Retrieval Policy](./06-context-authority-and-retrieval-policy.md)
- [Role Capability and Scope Control](./07-role-capability-and-scope-control.md)
- [Mutation, Concurrency, and Conflict Resolution](./08-mutation-concurrency-and-conflict-resolution.md)
- [Context Promotion and Rollback](./09-context-promotion-and-rollback.md)
- [Handoff Envelope Schema](../schemas/handoff-envelope.schema.json)
- [Current State Schema](../schemas/current-state.schema.json)
- [Agent Result Schema](../schemas/agent-result.schema.json)
- [Workflow Policy Template](../templates/workflow-policy.template.yaml)
