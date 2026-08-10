# 06 | Context authority and retrieval policy

[English](./06-context-authority-and-retrieval-policy.md) | [繁體中文](./06-context-authority-and-retrieval-policy-zh-TW.md)

This document defines how agents, coordinators, and orchestrators select, pin, and load context at each workflow stage. It ensures that every executor of a stage receives the same authoritative sources at exact versions and stops when a source is missing, stale, conflicting, or outside the permitted scope.

This document uses the following normative terms:

- "Must" marks a requirement that the workflow cannot omit.
- "Should" marks the default approach. A deviation requires a recorded reason.
- "May" marks an optional capability based on team size and risk.

## 1. Scope

Structured Handoff can already list `requiredInputRefs`, and an Artifact Reference can store a URI and checksum. These contracts identify which references pass to the next role. Another set of rules must answer:

- Which facts does the current stage need to establish?
- Which source governs each type of fact?
- Which version should the workflow select when a source has several versions?
- Is the source still valid, and does it belong to the current change, run, and state revision?
- When required context is missing, should the agent stop, request more input, or return the task for arbitration?

This document covers source authority, version pinning, freshness, integrity, context budgets, progressive disclosure, and failure routing. Other documents cover the following areas:

- For the basic structure of Current State, Agent Result, Handoff, and Artifact Reference, see [`02-reference-architecture.md`](./02-reference-architecture.md).
- For snapshots, artifacts, evidence, traces, and retention, see [`03-runtime-storage-and-retention.md`](./03-runtime-storage-and-retention.md).
- For path safety, prompt injection, stale artifacts, and evidence provenance, see [`05-security-and-maintainability.md`](./05-security-and-maintainability.md).
- For general retrieval, memory, token budgets, and context assembly, see [`../../../context-engineering/docs/01-context-engineering-core.md`](../../../context-engineering/docs/01-context-engineering-core.md).

This document does not cover embeddings, chunking, reranking, vector stores, or a general RAG pipeline. It does not change any existing JSON Schema.

## 2. Role responsibilities

| Role | Context responsibility | Prohibited action |
|---|---|---|
| Coordinator | Determine the fact types required by the stage, resolve authority conflicts, and approve necessary context expansion | Replace an approved Durable Contract with a chat summary |
| Orchestrator | Resolve references, pin versions, validate change/run/revision/checksum identity, and create the Handoff | Rewrite requirements or relax role permissions |
| Agent | Read only the context required by the Handoff and deliver the defined output contract | Search for another version to fill a gap or declare a Candidate accepted |
| Reviewer | Confirm that the review target, diff, validation, and acceptance criteria refer to the same change | Replace the reviewed version with a newer unvalidated artifact |
| Human | Arbitrate conflicting authoritative sources and approve high-risk operations or permission changes | Replace a durable Approval Record with verbal consent |

The same local script or operator may perform both coordinator and orchestrator work, but the workflow should record the responsibilities separately. The coordinator makes semantic decisions. The orchestrator performs reproducible contract checks.

## 3. Context data classes

Each context class has a different lifecycle and authority model.

| Class | Contents | Common location | Default use |
|---|---|---|---|
| Durable Contract | Requirement, Proposal, Design, Tasks, ADR, Schema, Workflow Policy | OpenSpec/Git | Define goals, constraints, and acceptance criteria |
| Operational State | Phase, Owner, Gate, Blocker, Next Action | `.agent-runtime/<change-id>/current-state.json` | Routing and recovery |
| Candidate/Accepted Artifact | Plan Result, Implementation Result, Review Result, Readiness Result | `.agent-runtime/<change-id>/<run-id>/artifacts/` | Stage delivery and version selection |
| Execution Evidence | Diff, Changed Files, Validation Summary, Command Result | `.agent-runtime/<change-id>/<run-id>/evidence/` | Verify execution results |
| Advisory Context | Chat Summary, Memory, previous runs, general search results, unapproved notes | Session/Memory/Trace/external source | Background and diagnosis |

Advisory Context can aid interpretation, but it cannot independently change a requirement, gate, permission, or Accepted Artifact Pointer. Load a complete trace only for arbitration, audits, incident investigation, or state reconstruction.

## 4. Select source authority by fact type

A single global ranking cannot express authority for every question. Requirements, runtime routes, implementation, validation, and permissions describe different facts. A source may govern one question and serve only as a reference for another.

### 4.1 Authority levels

| Level | Meaning | Use rule |
|---|---|---|
| `authoritative` | The source with final authority for a specific fact type | Prefer it during conflicts. Changes must pass the defined gate |
| `accepted` | An artifact that passed the required validation and gate | Downstream stages read it by default |
| `candidate` | An artifact currently being produced, validated, or reviewed | Only the designated stage may use it, and the stage must pin its version |
| `advisory` | Context that may add background but cannot override a higher authority | The workflow may exclude it, and it cannot advance a gate by itself |
| `diagnostic` | Context loaded for incident investigation, comparison, or failure analysis | It cannot automatically become an execution input |
| `prohibited` | Context blocked because of permission, sensitivity, staleness, or unknown provenance | The workflow must exclude it and record the reason |

Recency does not raise authority. A new agent summary, an unvalidated implementation, or the latest conversation can still rank below an approved contract or accepted artifact.

### 4.2 Fact type authority matrix

| Fact type | Authoritative source | Supporting source | Content that cannot override the authoritative source |
|---|---|---|---|
| Requirement/Scope | Approved OpenSpec, issue acceptance criteria, verifiable human decision | ADR, existing code, discussion summary | Chat, Memory, unapproved Proposal |
| Design Constraint | Approved Design, ADR, Schema, Workflow Policy | Existing implementation and tests | Agent inference, stale examples |
| Workflow Route | Current State at the expected revision and a valid Handoff | The `nextHandoff` proposal in an Agent Result | A next step in free text, stale Current State |
| Implementation Fact | Source from the designated worktree/branch, a pinned Diff, the corresponding Implementation Result | Diff Summary, Changed Files | Summary from an old run, unpinned working tree state |
| Validation Fact | Evidence bound to the same Candidate/Diff | Test description and Reviewer Notes | Natural language claims such as "tested" without evidence |
| Review Verdict | A schema-valid Review Result for the designated Candidate | Comments, diagnostic findings | A review of another version, Implementer self-review |
| Permission | Workflow Policy, Agent Adapter, Runtime Sandbox, Human Approval Record | A narrower scope in the Handoff | Permission expansion requested by an Artifact, Prompt, or Handoff |
| External Live State | A read-only tool result with a source, observation time, and validity period | Documentation, Cache, Retrieval Result | Memory, stale Tool Result, model inference |

If two `authoritative` sources give incompatible answers for the same fact type, the orchestrator must return the conflict to the coordinator or a human. It cannot choose a source by itself.

## 5. Current State, Context Plan, Context Manifest, and Handoff

These four records answer different questions:

| Contract | Question | Updated by |
|---|---|---|
| Current State | Which phase is active, who owns it, and which gates have passed? | Orchestrator/controlled workflow |
| Context Plan | Which fact types, budgets, and expansion conditions does this stage require? | Coordinator/Policy |
| Context Manifest | Which exact sources did this run load or exclude? | Orchestrator/Context Builder |
| Handoff | What must the next role do, which references must it read, and which output must it produce? | Orchestrator/Coordinator |

Use the following data flow:

```text
Current State + Stage Policy
            ↓
       Context Plan
            ↓
Resolve Authority / Version / Freshness / Permission
            ↓
      Context Manifest
            ↓
          Handoff
            ↓
          Agent
```

Keep Current State as a small Operational View. Store resolution results and exclusion reasons in the Context Manifest without copying complete source contents into Current State.

### 5.1 Compatibility with current schemas

The `requiredInputRefs` field in [`handoff-envelope.schema.json`](../schemas/handoff-envelope.schema.json) already supports `uri`, `sha256`, and `description`. The schema also sets `additionalProperties: false`. Until the repository adds a Context Manifest Schema, a Handoff must use the current fields as follows:

- Point each URI to an exact path. Do not use a `latest` alias that can drift over time.
- Set `sha256` when a content hash is available.
- Use `description` to record the fact type, purpose, and Candidate/Accepted identity.
- Validate the change ID and run ID from the referenced Artifact.
- Use `expiresAt` to limit the validity of a Handoff when needed.

Do not place fields such as `authority`, `version`, or `validUntil` directly in a Handoff because the current schema does not define them. When the workflow needs the full description, it may store a Conceptual Context Manifest as a separate Artifact and reference it through `requiredInputRefs`.

## 6. Lifecycle stage context matrix

Each stage loads only the context required to complete its work. If any required source in the table is missing, the agent should return `incomplete`, and the orchestrator should route it according to section 11.

| Stage | Required sources | Excluded by default |
|---|---|---|
| `plan-change` | Requirement, current Main Spec, relevant ADR/Schema, Current State, known constraints | Complete old conversations, all Source Files, historical Trace |
| `review-plan` | Candidate Proposal/Design/Tasks, Requirement, Acceptance Criteria, relevant ADR/Schema | Implementation Diff, unrelated Source, complete old reviews |
| `apply-change` | Approved Scope/Design/Tasks, Current State, Handoff, required Source Files, applicable Workflow Policy | Unapproved Proposal, Artifacts from other changes, complete history |
| `review-result` | Pinned Candidate Implementation Result, Diff for the same version, Validation from the same run, Approved Scope/Design/Acceptance Criteria | Other Candidates, stale Validation, unreferenced Implementer chat explanations |
| `fix-from-review` | Designated Candidate, unresolved Findings, Approved Scope, required Source, Focused Validation requirements | Closed unrelated Findings, Diffs from other runs |
| `readiness-check` | Artifact that passed Review, matching Validation, Review Result, Task Completion, Current State Gates | New Candidates that failed Validation, Raw Trace, unrelated historical versions |
| `archive-change` | Readiness Result, Accepted Artifact References, final Diff/Validation Summary, Scope Deviation, Human Gate status | Complete failed attempts, unresolved Candidates, Debug Log |

### 6.1 Exception for reading Candidates

Downstream stages read Accepted Artifacts by default. The following work may read a Candidate:

- `review-plan` reviews a Candidate Plan.
- `review-result` reviews a Candidate Implementation that completed the required Validation.
- `fix-from-review` changes the Candidate pinned by the Handoff.
- The coordinator performs conflict arbitration or failure diagnosis.

The Handoff must pin the Candidate URI and checksum. The agent cannot switch to a newer file in the same directory.

## 7. Context selection pipeline

An orchestrator or Context Builder should process context in this fixed order:

1. Read Current State and validate `changeId`, `runId`, `phase`, `currentOwner`, and `revision`.
2. Use the Stage Policy to list required fact types, optional fact types, the output contract, and the context budget.
3. Resolve the authoritative source for each fact type. Apply the Accepted/Candidate rules when several versions exist.
4. Pin the URI, Version Identity, checksum, change ID, run ID, and state revision.
5. Check role permission, path scope, sensitivity, and Handoff expiration.
6. Check freshness, integrity, Validation Target, and cross-run contamination.
7. Remove duplicates and exclude sources that have low authority, are stale, are irrelevant, or cannot be loaded.
8. Apply the context budget. If required sources exceed the budget, stop instead of truncating a contract or evidence arbitrarily.
9. Record Required, Optional, and Excluded Sources, including the reason for every exclusion.
10. Create the Handoff and pass exact references to the designated agent.

The following pseudocode illustrates the process:

```text
state = loadCurrentState(changeId)
assert state.revision == expectedRevision
assert state.currentOwner == targetRole

plan = resolveStagePolicy(state.phase, targetRole)
sources = resolveSources(plan.requiredFactTypes)
sources = pinVersionsAndChecksums(sources)
sources = filterByAuthorityFreshnessAndPermission(sources)

if missingRequiredSource(sources):
  return INCOMPLETE

manifest = buildContextManifest(state, plan, sources)
handoff = buildHandoff(state, manifest)
```

Before starting work, the agent should confirm that the Handoff has not expired, the Current State revision has not changed, and every required Artifact passes checksum validation. If any condition fails, the agent must not execute the original work command.

## 8. Version, freshness, and integrity

Each source type uses a different version identity:

| Source | Version identity | Freshness check | Integrity check |
|---|---|---|---|
| OpenSpec/Git File | Commit, Blob Hash, Change Revision, exact path | Confirm it remains the approved version | Git Object Identity/Checksum |
| Current State | `changeId`, `runId`, `revision` | Confirm it is still the current revision | Schema Validation/Atomic Read |
| Agent Result | `artifactId`, `runId`, exact URI | Confirm it remains the Candidate/Accepted version pinned by the Handoff | `sha256`/Schema Validation |
| Diff/Validation Evidence | Target Artifact, Diff Hash, Command Scope | Confirm it was produced for the current Candidate | Checksum/Exit Code/Target Binding |
| External Tool Result | Source ID, Observation ID | `observedAt`, `validUntil`, source SLA | Tool Signature/Response Hash/Trace ID |

### 8.1 Version pinning

Required sources remain pinned after the workflow creates a Handoff. If a new version appears during execution:

- The original Handoff continues to use its pinned version if that version remains valid and the Current State revision has not changed.
- If the new version changes scope, permission, acceptance criteria, or Current State, the orchestrator should invalidate the old Handoff and create a new one.
- The agent cannot treat a file with a newer modification time in the same directory as an automatic replacement.

### 8.2 Freshness

Set a freshness policy for each source type. A single TTL does not fit every type of context:

- Judge an approved OpenSpec by its approved version and change status, not a short TTL.
- Current State must match the expected revision.
- Validation must bind to the reviewed Candidate and cannot reuse a result from the previous attempt.
- External Live State should record an observation time and validity period.
- Chat Summary and Memory are Advisory by default. If the workflow cannot verify the source, it cannot raise their authority.

### 8.3 Integrity and binding

A matching checksum only proves that content has not changed. It does not prove that the content belongs to the current stage. The orchestrator must also verify that:

- The Artifact `changeId` matches the current change.
- The Artifact `runId` matches the Handoff or an explicitly allowed earlier run.
- Validation refers to the same Candidate/Diff.
- The Review Result covers the same Artifact used by Readiness.
- A coordinator or orchestrator updates the Current State Pointer. A Producer cannot accept its own output.

## 9. Minimal context and progressive disclosure

Initial context contains Required Sources only. Load Optional Sources when the required evidence is insufficient. Keep the reference and reason for every Excluded Source without loading its content.

### 9.1 Context budget

A local workflow can begin with budgets that are easy to verify:

- `maxRequiredRefs`
- `maxOptionalRefs`
- `maxTotalBytes`
- `maxFilesOpened`
- `maxDisclosureLevel`
- `reservedOutputTokens`

The budget cannot truncate requirements, acceptance criteria, required evidence, or negative conditions. If required sources exceed the budget, the agent should stop and ask the coordinator to narrow the scope, split the stage, or approve context expansion.

### 9.2 Expansion rules

A Context Expansion must record:

- The missing fact or evidence type.
- The sources already read and the decision they could not support.
- The requested reference or fact type.
- The estimated additional file, byte, or token cost.
- The maximum Disclosure Level.

An agent may propose an Expansion Request. It cannot expand path scope, tool permission, or sensitive data access by itself.

### 9.3 Default exclusions

Unless the Stage Policy explicitly requires them, initial context should exclude:

- Complete conversation history.
- Artifacts from other changes or runs.
- Full test stdout and Tool Trace.
- Candidates that failed Validation.
- Stale Memory that the workflow can retrieve again from an authoritative source.
- Paths and sensitive fields outside the role's access scope.

## 10. Conceptual Context Manifest

The following JSON is a conceptual example. It does not yet correspond to a formal JSON Schema in this repository. Do not embed it directly in an object validated by `handoff-envelope.schema.json`. Store it as a separate Artifact and reference it from the Handoff.

```json
{
  "schemaVersion": "0.1.0-draft",
  "changeId": "feature-example",
  "runId": "run-20260804-001",
  "stage": "readiness-check",
  "consumerRole": "readiness",
  "stateRevision": 7,
  "generatedAt": "2026-08-04T10:30:00Z",
  "requiredSources": [
    {
      "refId": "approved-scope",
      "factType": "requirement_scope",
      "authority": "authoritative",
      "uri": "repo://openspec/changes/feature-example/proposal.md",
      "version": "git:abc1234",
      "sha256": "1111111111111111111111111111111111111111111111111111111111111111",
      "purpose": "Verify accepted implementation against approved scope"
    },
    {
      "refId": "accepted-implementation-v1",
      "factType": "implementation",
      "authority": "accepted",
      "uri": "artifact://feature-example/run-20260804-001/implementation-result-v1.json",
      "version": "1",
      "sha256": "2222222222222222222222222222222222222222222222222222222222222222",
      "purpose": "Readiness target"
    },
    {
      "refId": "validation-v1",
      "factType": "validation",
      "authority": "accepted",
      "uri": "evidence://feature-example/run-20260804-001/validation-summary-v1.json",
      "version": "1",
      "sha256": "3333333333333333333333333333333333333333333333333333333333333333",
      "boundTo": "accepted-implementation-v1",
      "purpose": "Verify validation coverage and result"
    }
  ],
  "optionalSources": [
    {
      "refId": "focused-test-log-v1",
      "factType": "diagnostic_evidence",
      "authority": "diagnostic",
      "uri": "evidence://feature-example/run-20260804-001/focused-test-log-v1.txt",
      "loadWhen": "validation summary lacks failure details"
    }
  ],
  "excludedSources": [
    {
      "refId": "implementation-v2",
      "uri": "artifact://feature-example/run-20260804-001/implementation-result-v2.json",
      "reason": "validation_failed",
      "authority": "candidate"
    }
  ],
  "budget": {
    "maxFilesOpened": 12,
    "maxTotalBytes": 200000,
    "maxDisclosureLevel": 1
  },
  "fallbackPolicy": {
    "missingRequiredSource": "INCOMPLETE",
    "authorityConflict": "NEEDS_COORDINATOR_ARBITRATION"
  }
}
```

If a later version introduces a formal schema, it should validate Required/Optional/Excluded Sources, Authority, Version Identity, Freshness, Checksum, Binding, Budget, and Fallback Policy. Until then, the existing Handoff, Agent Result, and Current State schemas remain authoritative.

## 11. Fixed failure codes and routes

An agent should not use free text to decide how to handle a context error. A Result may store the following codes in `summary`, `risks`, `payload`, or the existing Blocker structure. The orchestrator then updates state through fixed rules.

| Code | Trigger | Agent Status | Recommended route |
|---|---|---|---|
| `CONTEXT_SOURCE_MISSING` | No usable source exists for a required fact type | `incomplete` | Supply the source or return to Coordinator |
| `CONTEXT_SOURCE_STALE` | Current State, Validation, or Live State is stale | `incomplete` | Resolve sources again and create a new Handoff |
| `CONTEXT_VERSION_MISMATCH` | Actual content does not match the pinned Version/checksum | `incomplete` | Invalidate the original Handoff |
| `CONTEXT_RUN_MISMATCH` | The Artifact belongs to another change/run without explicit permission | `incomplete` | Isolate the Artifact and resolve again |
| `CONTEXT_INTEGRITY_FAILED` | Checksum, Schema, or Target Binding validation fails | `failed` | Stop the workflow and retain Evidence |
| `CONTEXT_AUTHORITY_CONFLICT` | Authoritative sources conflict on the same fact type | `blocked` | `NEEDS_COORDINATOR_ARBITRATION` |
| `CONTEXT_SCOPE_DENIED` | The role cannot read a required source | `incomplete` | Narrow the work or request human permission review |
| `CONTEXT_BUDGET_EXCEEDED` | Required context exceeds the budget and cannot be compressed safely | `incomplete` | Split the scope or approve Expansion |

When `CONTEXT_SCOPE_DENIED` occurs, the agent cannot try another path, tool, or account to bypass the restriction. A human or a controlled Permission Workflow must handle any access increase.

## 12. Local example: separate Latest from Accepted

Assume an Implementer produces v1 and both Validation and Review pass. It then produces v2, but Validation for v2 fails:

```text
implementation v1
  → validation passed
  → review approved
  → accepted

implementation v2
  → validation failed
  → remains candidate
```

The current `current-state.schema.json` uses `latestArtifacts`. Until the schema adds `acceptedArtifacts`, the orchestrator must interpret these pointers as the latest usable and accepted Artifacts. It cannot point to v2 based on creation time.

The Readiness Handoff should pin v1:

```json
{
  "from": "reviewer",
  "to": "readiness",
  "stage": "readiness-check",
  "requiredInputRefs": [
    {
      "refId": "accepted-implementation-v1",
      "type": "agent-result",
      "uri": "artifact://feature-example/run-20260804-001/implementation-result-v1.json",
      "sha256": "2222222222222222222222222222222222222222222222222222222222222222",
      "description": "Accepted implementation for readiness; candidate v2 failed validation"
    }
  ]
}
```

This is a reading excerpt and omits other fields required by the Handoff Schema. It cannot be validated as written.

If the objective changes to investigating why v2 failed, the coordinator should create a new Diagnostic Handoff that references v2 and its Validation Evidence. Reading v2 for diagnosis does not change the Accepted status of v1 or allow v2 to enter Readiness.

## 13. Minimum local adoption

A workflow can adopt most of this policy before adding another schema:

1. List the required fact types in the Stage Policy or Coordinator Prompt.
2. Use exact URIs in a Handoff. Do not use a `latest` filename or an ambiguous directory reference.
3. Set `sha256` in an Artifact Reference when the workflow can calculate a content hash.
4. Use `description` to record source purpose, Candidate/Accepted identity, and pinned version.
5. Before execution, check the Current State revision, Handoff expiration, and Artifact run ID.
6. When required context is missing, return `incomplete` with a fixed failure code.
7. Record the references that the orchestrator loaded and excluded. The first version may store this record in Trace.
8. Let the coordinator or orchestrator manage the Accepted Pointer. A Producer only creates Candidates.

When context selection must work across agent adapters or machines, or when strict audit requirements apply, consider adding:

```text
schemas/context-manifest.schema.json
templates/context-manifest.template.json
examples/context-authority-conflict/
```

These assets are a later contract upgrade and are not prerequisites for adopting this policy.

## 14. Acceptance checklist

### Authority

- [ ] Every required fact type has an explicit authoritative source.
- [ ] Advisory, Diagnostic, and Prohibited Context cannot override an authoritative source.
- [ ] An Authority Conflict enters coordinator or human arbitration.

### Version and freshness

- [ ] The Handoff uses exact URIs and stores checksums when possible.
- [ ] The workflow validates the Current State revision, change ID, and run ID before execution.
- [ ] Validation and Review bind to the same Candidate/Diff.
- [ ] External Live State has an observation time and validity policy.

### Context assembly

- [ ] Each stage loads only required sources.
- [ ] The workflow loads Optional Context according to the Expansion Policy.
- [ ] Every excluded source has an inspectable reason.
- [ ] The Context Budget does not truncate requirements, negative conditions, or required evidence.

### Agent behavior

- [ ] The agent does not switch to a newer Artifact on its own.
- [ ] The agent does not infer missing requirements, permissions, or gates from Chat or Memory.
- [ ] Missing required context produces a fixed error code and an `incomplete`/`blocked` status.
- [ ] A Producer cannot accept an Artifact or update the Accepted Pointer.

### Schema compatibility

- [ ] The current Handoff uses only fields defined by `handoff-envelope.schema.json`.
- [ ] A Conceptual Context Manifest remains a separate Artifact and is not embedded in the current Handoff.
- [ ] This document does not change the semantics of the Current State, Agent Result, or Handoff schemas.

## 15. References

- [Artifact-based Shared State and Structured Handoff reference architecture](./02-reference-architecture.md)
- [Separating repository knowledge, runtime state, evidence, and trace](./03-runtime-storage-and-retention.md)
- [Security, governance, and long-term maintainability](./05-security-and-maintainability.md)
- [Context Engineering Core](../../../context-engineering/docs/01-context-engineering-core.md)
- [Collaboration in Multi-Agent Systems: A Practical Guide to Getting It Right](https://www.puppyone.ai/en/blog/collaboration-in-multi-agent-systems-practical-guide)
- [Harness + Mut: The Recommended Split for AI Agent Delivery and Context Control](https://www.puppyone.ai/en/blog/harness-mut-recommended-split-for-ai-agent-delivery-and-context-control)
