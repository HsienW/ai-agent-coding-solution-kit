# 07 | Role capability and scope control

[English](./07-role-capability-and-scope-control.md) | [繁體中文](./07-role-capability-and-scope-control-zh-TW.md)

This document defines how Agents, Coordinators, Orchestrators, Agent Adapters, and Human Approvers determine which Actions a Role may perform within the current Change, Run, Stage, and resource scope. Policy, Handoff, Runtime State, and Approval Records must provide enough information to reconstruct every authorization decision. Prompts, Artifact content, and Agent claims cannot add permissions.

This document uses the following normative terms:

- "MUST" defines a requirement that the workflow cannot omit.
- "SHOULD" defines the default practice; a deviation requires a recorded reason.
- "MAY" defines an optional capability that a team can adopt according to its scale and risk.

## 1. Scope

[`06-context-authority-and-retrieval-policy.md`](./06-context-authority-and-retrieval-policy.md) defines which sources and versions an Agent should trust. This document addresses the authorization questions that follow:

- Which Runtime Identity is making the request?
- Which Role does that Identity hold?
- Which Capabilities may that Role use in the current Stage?
- Which Repository Paths, Artifacts, Tools, and Environments may those Capabilities affect?
- Does the Handoff keep the task within the approved Change Scope?
- Does the Action require Human Approval, a Gate, or another condition?
- After a denial, should the workflow stop, request missing information, request approval, or return for arbitration?

This document covers Role Boundaries, Capability Vocabulary, Scope Intersection, Tool and Path Control, Conditional Grants, Authorization Decisions, and denial routing. Other documents cover the following subjects:

- For the base structures of Current State, Agent Result, Handoff, and Artifact Reference, see [`02-reference-architecture.md`](./02-reference-architecture.md).
- For Runtime Storage, Retention, and Cross-run Isolation, see [`03-runtime-storage-and-retention.md`](./03-runtime-storage-and-retention.md).
- For Path Traversal, Prompt Injection, Gate Forgery, Artifact Tampering, and Single Writer, see [`05-security-and-maintainability.md`](./05-security-and-maintainability.md).
- For Context Authority, Version Pinning, and Retrieval Scope, see [`06-context-authority-and-retrieval-policy.md`](./06-context-authority-and-retrieval-policy.md).

This document does not define implementations for concurrent writes, Merge, Lease, or Compare-and-swap. It does not change the existing JSON Schemas.

## 2. Authorization boundary

### 2.1 Role, Runtime Identity, and Capability

The workflow MUST record these concepts separately:

| Name | Question | Example |
|---|---|---|
| Role | What responsibility does this execution hold? | `reviewer` |
| Runtime Identity | Which Agent, Process, or Human made the request? | `agent:reviewer-local-01` |
| Capability | Which Action may that Identity perform on which resource class? | `artifact.write.own` |

The same Agent CLI may hold different Roles in different Runs, but each Invocation MUST bind one Runtime Identity, Role, Change ID, Run ID, and Stage. The Orchestrator MUST NOT infer Identity from a model name, Prompt text, or directory name alone.

A Role describes responsibility. A Capability grants execution authority. Text such as "may modify," "please execute," or "approval granted" in a Prompt does not create an authorization source.

### 2.2 Policy Plane and Execution Plane

Authorization has two layers:

```text
Workflow Policy / Role Policy / Approval Record
                    ↓
          Authorization Decision
                    ↓
Agent Adapter / Sandbox / Tool Host / Filesystem
```

The Policy Plane decides whether to allow a request. The Execution Plane enforces that decision. An Agent Adapter MUST translate the decision into concrete controls such as a read-only mount, an allowed Working Directory, a Tool Allowlist, a Command Timeout, and a Network Boundary. A Prompt instruction that says "do not modify" does not create an enforceable security boundary.

### 2.3 Least privilege and default deny

Each Invocation receives only the minimum Capabilities required for its Stage. The system MUST deny a request when:

- The Capability is unknown or unregistered.
- The Role Policy does not grant the Capability.
- The Resource, Path, Tool Argument, or Environment falls outside Scope.
- The Policy, State Revision, Handoff, or Approval has expired.
- A required Gate has not passed.
- The system cannot determine whether the request has side effects.

The absence of a rule does not grant access. Before a team releases a new Tool, Action, or resource class, it MUST add the item to a versioned Policy.

## 3. Capability vocabulary

A Capability uses a stable name to describe a resource class and Action. The name does not depend on a specific CLI or model provider.

### 3.1 Recommended Capabilities

| Capability | Description |
|---|---|
| `spec.read` | Read approved Requirements, Proposals, Designs, Tasks, or ADRs |
| `spec.write` | Create or modify specifications within the approved Change Scope |
| `source.read` | Read product source, tests, and configuration |
| `source.write` | Modify assigned product source, tests, or configuration |
| `artifact.read` | Read an Agent Result or other Artifact referenced by the Handoff |
| `artifact.write.own` | Create the output Artifact for the current Role and Stage |
| `evidence.read` | Read specified Diff, Validation, or Command Evidence |
| `evidence.write.own` | Create Evidence produced by the current execution |
| `runtime.read` | Read Current State and the active Handoff |
| `runtime.transition.propose` | Propose the next Transition in an Agent Result |
| `runtime.transition.apply` | Update Current State and Handoff after validating Gates |
| `tool.execute.read` | Execute a Tool that should not change external state |
| `tool.execute.write` | Execute a Tool that changes files, services, or external systems |
| `git.commit` | Create a Git Commit |
| `git.push` | Push a Commit to a remote |
| `deploy.execute` | Deploy to a specified Environment |
| `resource.delete` | Delete a file, data record, or external resource |
| `permission.change` | Change a Role, Policy, Credential, or access control |

Capability IDs SHOULD remain small and stable. Scope or Conditions should carry Path, Tool ID, Environment, and Risk Level instead of generating a new Capability name for each file.

### 3.2 Actions must remain distinct

The workflow MUST NOT collapse the following Actions into a generic `write`:

| Action | Meaning |
|---|---|
| `enumerate` | List resource names in a container, directory, or collection |
| `read` | Read a known resource |
| `create` | Create a resource without overwriting existing content |
| `update` | Modify an existing resource |
| `delete` | Delete a resource |
| `execute` | Run a Tool, Command, Migration, or Deployment |
| `propose` | Propose a state transition or Approval Request |
| `approve` | Approve a fixed Action and Scope |
| `apply` | Apply a validated Transition or change |

Permission to `read` does not imply permission to `enumerate`. Permission to `create` does not allow `update` or replacement of an existing Artifact.

## 4. Scope dimensions

A Grant MUST limit the resources it can affect. Authorization MUST evaluate at least the following dimensions:

| Dimension | Required information | Description |
|---|---|---|
| Actor | `agentId`, `role` | Binds the requesting Identity and Role |
| Lifecycle | `changeId`, `runId`, `stage` | Prevents access across Changes, Runs, or Stages |
| Resource | URI Scheme, Resource Class | Separates Spec, Source, Runtime, Evidence, and external systems |
| Path | Canonical Root, Allowlist Pattern | Restricts Repository, Worktree, and Runtime paths |
| Action | `read`, `create`, `execute`, and others | Restricts how the Actor may use the resource |
| Tool | Tool ID, Command type, Argument Constraint | Restricts the execution interface and parameters |
| Environment | local, CI, staging, production | Prevents local permissions from reaching higher-risk environments |
| Risk | low, medium, high | Determines whether the Action requires Approval or a denial |
| Time | `issuedAt`, `expiresAt` | Limits the validity period of a Grant or Approval |
| Policy | `policyId`, `policyVersion` | Makes the decision reproducible and auditable |

Existing URI Schemes can identify resources:

```text
repo://openspec/changes/feature-example/design.md
repo://src/payment/service.ts
runtime://feature-example/run-002/artifacts/review-result.json
evidence://feature-example/run-002/validation-summary.json
```

A URI identifies a resource. The Adapter still MUST resolve the physical path, normalize it, and verify that the target remains inside an approved Root. The Path Traversal, Symbolic Link, and Junction checks in document 05 still apply.

## 5. Effective Scope

### 5.1 Intersection model

The effective permissions are the intersection of several constraints:

```text
Effective Scope =
  Platform Maximum
  ∩ Adapter Maximum
  ∩ Role Policy
  ∩ Stage Policy
  ∩ Handoff Scope
  ∩ Change / Run Boundary
  ∩ Environment Boundary
```

Each layer may narrow Scope. A lower layer cannot broaden a higher-layer limit. Explicit Deny takes precedence over every Allow.

Human Approval only satisfies an existing Conditional Grant. If Platform Maximum or Role Policy denies `permission.change`, a one-time Approval does not add that Capability to Effective Scope.

### 5.2 Evaluation order

The Orchestrator or Policy Engine MUST evaluate requests in a fixed order:

1. Validate the Runtime Identity, Role, Change ID, Run ID, and Stage.
2. Load the specified versions of the Platform, Adapter, Role, and Stage Policies.
3. Validate the Handoff and pin its Resource References.
4. Calculate the intersection of all Scope layers.
5. Normalize the Resource, Path, Tool ID, and Arguments.
6. Apply Explicit Deny before evaluating Allow.
7. Check Risk, Gate, Approval, expiration, and State Revision.
8. Produce an `allow`, `deny`, or `pause` decision and write it to the Audit Trace.

If any step cannot complete, the workflow MUST fail closed. An Agent MUST NOT change Tools, rewrite Paths, or shorten a Command to probe available permissions.

### 5.3 State and Policy binding

An Authorization Decision SHOULD bind at least:

```text
policyId + policyVersion
changeId + runId + stage
stateRevision
actorId + role
capability + resource
```

When the Current State Revision changes, the system SHOULD reevaluate prior Decisions. This prevents an Agent from reusing old authority after a Handoff update, a Scope reduction, or a revoked Gate.

## 6. Role capability matrix

The following table defines the default boundaries. A project MAY narrow them further. It MUST NOT remove a Human Gate or grant a Reviewer product write access.

| Role | Allowed by default | Fixed restrictions |
|---|---|---|
| Coordinator | Read Spec, Source, and Runtime; maintain Plan and Spec within the approved Change; create `coordinator_result`; propose Transitions | Must not modify product source, update Current State directly, or execute Git, Deploy, or Delete Actions |
| Implementer | Read the approved Contract; modify assigned Worktree and Paths; run allowed Validation; create `implementation_result` and Evidence for the current Run; propose Transitions | Must not expand Scope, approve its own result, update Current State directly, or execute Git or Deploy Actions |
| Reviewer | Read specified Spec, Source, Diff, and Evidence; create `review_result`; propose Transitions | Must not modify Spec, Source, Implementation Result, Evidence, or Current State; must not run destructive Commands |
| Readiness | Read Accepted Artifacts, Review Result, Gates, and Evidence; create `readiness_result`; propose Transitions | Must not modify the implementation, Review Verdict, Evidence, Accepted Pointer, or Current State |
| Archive | Read Artifacts that passed Readiness; create `archive_result` and a controlled Execution Summary; propose Transitions | Must not modify the implementation or prior Results, bypass the Human Git Gate, or declare a Commit complete |
| Orchestrator / Automation | Load Policy, assemble Handoff, invoke Adapters, validate Results, apply legal Transitions, and update Runtime State | Must not modify product source or approved Spec, make the Reviewer's quality decision, or approve high-risk Actions for a Human |
| Human | Arbitrate Scope, approve high-risk Actions, and execute controlled Git, Deploy, or Delete Actions under organizational Policy | Approval MUST bind a fixed Action and Scope; verbal consent does not update a Gate |

`automation` is a Runtime Identity type that may perform mechanical Orchestrator work. Automation does not receive broader permissions simply because it runs without a Human operator.

## 7. Own Output Write

### 7.1 Precise meaning of a read-only Reviewer

A Reviewer has read-only access to product source, specifications, the Implementation Result, existing Evidence, and Current State. The Reviewer still needs to create its own `review_result`, so it may receive a narrow `artifact.write.own` Grant:

```text
Capability: artifact.write.own
Action: create
Resource: runtime artifact
Path: .agent-runtime/<change-id>/<run-id>/artifacts/review-result.json
Owner: reviewer
Stage: review-result
Overwrite: deny
```

If the path already exists, the Adapter SHOULD reject the overwrite or the Orchestrator SHOULD allocate a new Artifact ID or Attempt Path. The Reviewer MUST NOT edit the Implementation Result to fix reviewed content.

### 7.2 Other Stage Outputs

The same rule applies to each Role's output:

| Role | Own Output that the Role may create | Upstream content that the Role must not rewrite |
|---|---|---|
| Coordinator | `coordinator-result.json` | Approved Requirement, Current State |
| Implementer | `implementation-result.json`, Evidence for the current Run | Review Result, Accepted Pointer |
| Reviewer | `review-result.json` | Source, Spec, Implementation Result, Evidence |
| Readiness | `readiness-result.json` | Review Result, Gate, Accepted Artifact |
| Archive | `archive-result.json`, controlled Execution Summary | Implementation, Review, Readiness Result |

Own Output MUST match `producer`, `kind`, `stage`, `changeId`, and `runId`. The Orchestrator MUST validate these fields before accepting a Result.

## 8. Path and data visibility

### 8.1 Path Scope

The Adapter MUST resolve a requested path to a Canonical Path before comparing it with an Allowlist. The following patterns cannot translate directly into Filesystem permissions:

```text
repo://src/**
runtime://feature-example/**
```

A Wildcard may act as a Policy Pattern, but execution must resolve to a concrete Root and Path. The Adapter MUST reject Root Escape through `..`, alternate data streams, Symbolic Links, Junctions, or case differences.

When one Role needs separate writable areas, grant them separately:

```text
source.write       -> assigned worktree paths
artifact.write.own -> fixed result path
evidence.write.own -> fixed evidence directory
```

The workflow MUST NOT combine these three Scopes into Write Access for the entire Repository.

### 8.2 Enumerate and Read

Directory and collection names may reveal sensitive information. A Policy MUST handle the following operations separately:

- `read` for a known URI.
- `enumerate` under a specified Root.
- `search` for content that matches a Pattern.

A Reviewer may read the Diff referenced by the Handoff. That permission does not allow the Reviewer to list `.env`, a Credential Store, or another Change's Runtime directory.

### 8.3 Field and content redaction

Data minimization still applies after authorization succeeds:

- The system MUST NOT send Secrets, Tokens, Credentials, or Approval Proof to model Context.
- Authorization Trace MUST NOT store complete sensitive Arguments.
- Error Messages MUST NOT return details that allow enumeration of protected resources.
- A Reviewer receives only the Evidence required for the decision. Unrelated Secret Stores remain inaccessible.

## 9. Tool and Command control

### 9.1 A Tool ID cannot replace Action authorization

An Allowlist SHOULD evaluate Tool ID, Action, Resource, and Arguments together. If the system denies `filesystem.write`, it MUST also deny an attempt to write the same path through a Shell, Script Runner, or Git Hook.

Tools with side effects SHOULD be separate from Read Tools. For example:

```text
issue.read     -> tool.execute.read
issue.comment  -> tool.execute.write
deploy.inspect -> tool.execute.read
deploy.release -> deploy.execute
```

A Tool Description helps an Agent select a Tool. It does not enforce authorization.

### 9.2 Shell and Command

A Shell exposes broad capabilities. If a Stage requires Shell access, the Adapter SHOULD restrict:

- Working Directory.
- Executable or Command Family.
- Arguments and Target Path.
- Environment Variables and Credential Exposure.
- Network Access.
- Timeout, Output Size, and acceptable Exit Codes.

After an Agent assembles a Command from natural language, the Execution Plane MUST authorize it again. The Agent's claim that a Command is "read-only" cannot serve as the decision source.

### 9.3 Reviewer validation

A local workflow may use either mode:

| Mode | Reviewer permissions | Evidence source |
|---|---|---|
| Strict read-only | Does not run Commands; reads only Evidence produced by the Implementer or Automation | Implementer / CI / Automation |
| Isolated validation | Runs Allowlisted Commands against a read-only Source Snapshot and controlled temporary area | Reviewer Invocation |

Isolated validation requires an explicit `tool.execute.read` Grant, a pinned Snapshot, and a disposable Output Directory. If a test tool modifies Source, a Lockfile, a Snapshot, or an external service, controlled Automation SHOULD run it.

## 10. Handoff Scope

### 10.1 A Handoff can only narrow permissions

A Handoff specifies work and inputs. It cannot create a Capability. The following content cannot broaden authority:

- Natural language in `reason` or `notes`.
- Instructions embedded in an Artifact.
- `nextHandoff` proposed by an Agent Result.
- `requiresHumanApproval: true`.
- An Artifact Reference that points to an unapproved path.

When the Orchestrator creates an Invocation, it MUST intersect the Handoff's Stage, References, Change ID, Run ID, and expiration with Role Policy and Current State.

### 10.2 Compatibility with the current Schema

The current [`handoff-envelope.schema.json`](../schemas/handoff-envelope.schema.json) uses `additionalProperties: false` and has no Capability, Allowed Path, or Tool Scope fields. An implementation of this policy MUST NOT add custom fields directly.

The current [`workflow-policy.template.yaml`](../templates/workflow-policy.template.yaml) lists Coordinator, Implementer, Reviewer, Human, and a small set of Boolean permissions. It does not express Readiness, Archive, Automation, Path, Tool Argument, or Environment. The template can set a local baseline maximum, but it cannot make a complete authorization decision. The system MUST deny an unlisted Role or Capability until the Adapter has an explicit, auditable configuration for it.

Without changing the Schema, a local workflow uses:

1. [`workflow-policy.template.yaml`](../templates/workflow-policy.template.yaml) to store the baseline capability maximum for a Role.
2. Agent Adapter settings to fix Tool, Filesystem, and Sandbox limits for each Role and Stage.
3. Exact `requiredInputRefs` in the Handoff to restrict input resources.
4. Approved OpenSpec or Change Scope to restrict writable product paths.
5. Fixed Own Output Paths allocated by the Orchestrator.
6. Trace records for the applied Policy Version and Authorization Decision.

If a team needs a portable, fine-grained Policy across Adapters or machines, it MAY add a separate Capability Policy Artifact in a later version and reference it with the existing `other` type. Before adding it, the team MUST define a Schema and version migration rules.

### 10.3 Handoff expiration

After `expiresAt`, the Orchestrator MUST stop the Invocation or deny subsequent Actions. To continue the work, a Coordinator or Orchestrator SHOULD create a new Handoff from the latest Current State. An Agent cannot extend the expiration time.

## 11. Human Approval and Conditional Grants

Reviewer Verdict, Human Approval, Transition Apply, and Action Execution are separate events:

```text
Reviewer produces a quality decision
          ↓
Orchestrator validates Gates and Policy
          ↓
Human approves a fixed high-risk Action
          ↓
Controlled Executor performs the Action
```

An Approval Record SHOULD bind:

- Approver Identity.
- Requested Capability and Action.
- Resource, Path, or Environment.
- Change ID, Run ID, Stage, and State Revision.
- Policy ID and Policy Version.
- Evidence Reference and expected side effects.
- Issued At, Expires At, and whether the approval is reusable.

An Approval SHOULD NOT be a Boolean that an Agent Result can write. Only an Orchestrator or controlled Approval Service may update `humanApproved` in the existing Current State after validating an Approval Record.

After a Human rejects a request, the Executor MUST NOT retry with changed parameters. If Scope must change, the workflow returns to a new Approval Request.

## 12. Conceptual Capability Policy

The following YAML illustrates the authorization model. It does not conform to the current `workflow-policy.template.yaml` and cannot be embedded in the current Handoff:

```yaml
schemaVersion: 0.1.0-draft
policyId: local-reviewer-policy
policyVersion: 0.1.0
role: reviewer

grants:
  - capability: spec.read
    actions: [read]
    resources:
      - repo://openspec/changes/${changeId}/**

  - capability: source.read
    actions: [read]
    resources:
      - repo://src/**
      - repo://tests/**

  - capability: runtime.read
    actions: [read]
    resources:
      - runtime://${changeId}/current-state.json
      - runtime://${changeId}/${runId}/handoff-envelope.json

  - capability: artifact.read
    actions: [read]
    resources:
      - runtime://${changeId}/${runId}/artifacts/implementation-result.json

  - capability: evidence.read
    actions: [read]
    resources:
      - evidence://${changeId}/${runId}/**

  - capability: artifact.write.own
    actions: [create]
    resources:
      - runtime://${changeId}/${runId}/artifacts/review-result.json

denies:
  - capability: source.write
  - capability: spec.write
  - capability: runtime.transition.apply
  - capability: git.commit
  - capability: git.push
  - capability: deploy.execute

conditions:
  stages: [review-result]
  maxRiskLevel: low
  requireCurrentStateRevision: true
  expiresAt: "2026-08-10T12:00:00Z"
```

Before adopting this format, a project needs at least:

```text
schemas/capability-policy.schema.json
schemas/authorization-decision.schema.json
templates/role-capability-policy.template.yaml
examples/reviewer-source-readonly-output-write/
```

These assets belong to a later contract upgrade. This document does not require them.

## 13. Authorization Decision

### 13.1 Minimum result

Each controlled Action SHOULD produce an auditable Decision. The following is a conceptual result:

```json
{
  "decision": "deny",
  "code": "RESOURCE_OUT_OF_SCOPE",
  "actorId": "agent:reviewer-local-01",
  "role": "reviewer",
  "changeId": "feature-example",
  "runId": "run-review-002",
  "stage": "review-result",
  "capability": "source.write",
  "resourceClass": "repository-path",
  "policyId": "local-reviewer-policy",
  "policyVersion": "0.1.0",
  "safeReason": "Requested path is outside the reviewer write scope.",
  "nextAllowedAction": "produce-review-result",
  "traceId": "authz-review-002-004"
}
```

This result is a conceptual format and does not belong to the current Agent Result Schema. An initial implementation MAY store equivalent fields in Trace.

### 13.2 Fixed denial codes

| Code | Condition | Default route |
|---|---|---|
| `CAPABILITY_UNKNOWN` | The Policy does not recognize the requested Capability | `deny`, return to Orchestrator configuration review |
| `CAPABILITY_NOT_GRANTED` | The Role or Stage does not have the Capability | `deny`, do not retry automatically |
| `RESOURCE_OUT_OF_SCOPE` | The Resource or Path falls outside the approved scope | `deny`, return to the Coordinator for Scope review |
| `PATH_ENUMERATION_DENIED` | The Actor may read a known resource but may not list its container | `deny`, use an exact Reference |
| `TOOL_NOT_ALLOWED` | The Tool is absent from the Adapter Allowlist | `deny`, use an approved Tool or request missing configuration |
| `ARGUMENT_SCOPE_VIOLATION` | Tool Arguments target an unapproved resource | `deny`, do not probe permissions through argument changes |
| `RISK_LEVEL_EXCEEDED` | The Action risk exceeds the Policy maximum | `pause` or `deny` |
| `APPROVAL_REQUIRED` | A Conditional Grant still needs approval | `pause`, create an Approval Request |
| `APPROVAL_INVALID_OR_EXPIRED` | The Approval does not match, has expired, or has already been used | `pause`, request a new Approval |
| `SCOPE_EXPIRED` | The Handoff or Invocation Scope has expired | `incomplete`, create a new Handoff |
| `POLICY_VERSION_MISMATCH` | The Decision used a different Policy version | `blocked`, reevaluate |
| `STATE_GATE_NOT_SATISFIED` | Current State has not passed a required Gate | `blocked`, return to a legal prerequisite Stage |

`safeReason` describes only the resource class and a safe next step. It SHOULD NOT reveal a Secret, Credential, Approval Token, protected path, or whether a protected resource exists.

## 14. Audit Trace

Allow, Deny, and Pause decisions SHOULD all create Trace records. The minimum content includes:

- Timestamp and Trace ID.
- Actor ID, Role, Change ID, Run ID, and Stage.
- Policy ID, Policy Version, and State Revision.
- Requested Capability, Action, and a sanitized Resource Reference.
- Decision, Reason Code, and the applied Gate or Approval Reference.
- The actual Adapter, Tool ID, and Exit Status.

Trace SHOULD NOT store:

- Raw Secrets, Tokens, Credentials, or Approval Proof.
- Complete sensitive Tool Arguments.
- Agent Chain-of-thought.
- Repository content unrelated to the authorization decision.

The Authorization Decision and Tool Execution for one Action SHOULD share a Trace ID. This allows an operator to distinguish an allowed Action that failed during execution from an Action that never received permission.

## 15. Local Lifecycle example

The following example uses Single Writer and `.agent-runtime`:

```text
Change: feature-example
Run: run-002
Assigned source scope:
  src/payment/**
  tests/payment/**
```

### 15.1 Coordinator: `plan-change`

The inputs are the approved Requirement and current Source. The Coordinator may update the Proposal, Design, and Tasks for this Change and create `coordinator-result.json`. Product source and Current State remain non-writable.

### 15.2 Implementer: `apply-change`

The Orchestrator restricts the Implementer to the assigned Worktree, `src/payment/**`, `tests/payment/**`, and the Output Directory for this Run. The Implementer may modify source, run Allowlisted Validation, and create an Implementation Result and Evidence. If it attempts to modify `src/identity/**`, the Adapter returns `RESOURCE_OUT_OF_SCOPE`.

### 15.3 Reviewer: `review-result`

The Reviewer reads pinned versions of the Spec, Diff, Implementation Result, and Validation Evidence. The Source Snapshot remains read-only, and the Reviewer may create only `review-result.json` for this Run. If the Reviewer attempts to fix source, the authorization layer denies `source.write`. The Reviewer SHOULD record the problem as a Finding for the Implementer.

### 15.4 Readiness: `readiness-check`

Readiness reads only the Accepted Implementation, passed Review, required Evidence, and Gates, then creates `readiness-result.json`. It may return `incomplete`, but it cannot run a Command with side effects or update `humanApproved`.

### 15.5 Archive and Human Gate

Archive creates `archive-result.json` and the Execution Summary. If the next step requires a Commit, the Orchestrator creates an Approval Request bound to a fixed Commit Scope and Diff Reference. After Human Approval, a controlled Executor with `git.commit` may execute the Action. The Archive Agent does not receive Git permission.

### 15.6 Runtime State update

Each Role may only propose a Transition in an Agent Result. The Orchestrator updates `current-state.json` only after validating the Result Schema, Capability, Gates, State Revision, and Transition Table. An Artifact Producer cannot set its own output as Accepted.

## 16. Minimum local adoption

A workflow can adopt most of this policy without changing the current Schemas:

1. Treat [`workflow-policy.template.yaml`](../templates/workflow-policy.template.yaml) as the baseline maximum for its four existing Roles. Deny unlisted Roles and Capabilities by default, and do not grant permissions dynamically through Handoff.
2. Fix the Filesystem, Tool, Shell, and Network permissions for each Role and Stage in every Agent Adapter.
3. Have the Coordinator list writable Paths for the Implementer in the approved Change document.
4. Have the Orchestrator assemble inputs from exact Handoff References without exposing the entire Runtime Root.
5. Allocate a fixed, non-overwritable Own Output Path to each Role.
6. Give the Reviewer read-only Source access and create-only access to its Review Result.
7. Allow only the Orchestrator or controlled Automation to update Current State.
8. Keep Human Gates for Commit, Push, Deploy, Delete, and Permission Change.
9. Record at least the Policy Version, Requested Capability, Decision Code, and Trace ID.
10. Deny unknown Capabilities, Scope, and Policy Versions.

Add a separate Capability Policy Schema when Boolean permissions can no longer express Path, Tool Argument, Environment, or Approval Conditions. Before that upgrade, use Trace to verify which Capabilities the workflow uses so that the Policy does not grant unused authority.

## 17. Acceptance checklist

### Identity and Policy

- [ ] Every Invocation binds Actor ID, Role, Change ID, Run ID, and Stage.
- [ ] Role, Runtime Identity, and Capability are recorded separately.
- [ ] Policy has an ID and Version; incompatible versions fail closed.
- [ ] Prompts, Artifacts, and Agent Results cannot add Capabilities.

### Scope

- [ ] Effective Scope is the intersection of Platform, Adapter, Role, Stage, Handoff, Run, and Environment constraints.
- [ ] Explicit Deny takes precedence.
- [ ] A Canonicalized Path remains inside its approved Root.
- [ ] `enumerate`, `read`, `create`, `update`, and `delete` are controlled separately.
- [ ] Tool ID, Action, Resource, and Arguments pass authorization together.

### Role boundary

- [ ] The Implementer can modify only its assigned Worktree and Paths.
- [ ] The Reviewer has read-only access to Source, Spec, and upstream Artifacts.
- [ ] The Reviewer can create only its Review Result and cannot overwrite an existing Artifact.
- [ ] Readiness and Archive can create only their Stage Outputs.
- [ ] Only the Orchestrator can apply legal Transitions and update Current State.

### Handoff and Approval

- [ ] A Handoff can only narrow existing permissions.
- [ ] The current Handoff contains no Capability fields absent from its Schema.
- [ ] `requiresHumanApproval` does not count as Approval Proof.
- [ ] Reviewer Verdict, Human Approval, Transition Apply, and Action Execution are recorded separately.
- [ ] Approval binds Action, Scope, Policy Version, State Revision, and expiration.

### Audit and failure handling

- [ ] Allow, Deny, and Pause decisions have a Trace ID and fixed Reason Code.
- [ ] A denial message does not reveal Secrets or protected resource information.
- [ ] After a denial, an Agent does not switch Tools or alter arguments to bypass the restriction.
- [ ] Authorization is reevaluated after the Current State Revision changes.
- [ ] An unknown Capability or missing required Policy results in default deny.

## 18. References

- [Artifact-based Shared State and Structured Handoff reference architecture](./02-reference-architecture.md)
- [Separating Repository Knowledge, Runtime State, Evidence, and Trace](./03-runtime-storage-and-retention.md)
- [Security, governance, and long-term maintainability](./05-security-and-maintainability.md)
- [Context Authority and Retrieval Policy](./06-context-authority-and-retrieval-policy.md)
- [Agent Platform Operations](../../../context-engineering/docs/03-agent-platform-operations.md)
- [Tool Governance and Evaluation](../../../agent-design/tool-schema-routing/docs/03-tool-governance-and-evaluation.md)
- [Handoff Envelope Schema](../schemas/handoff-envelope.schema.json)
- [Current State Schema](../schemas/current-state.schema.json)
- [Agent Result Schema](../schemas/agent-result.schema.json)
- [Workflow Policy Template](../templates/workflow-policy.template.yaml)
