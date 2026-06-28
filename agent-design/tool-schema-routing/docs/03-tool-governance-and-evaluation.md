# Tool Governance, Evaluation, and Observability

[English](./03-tool-governance-and-evaluation.md) | [繁體中文](./03-tool-governance-and-evaluation-zh-TW.md)

## 1. A Schema Is Not Finished When It Compiles

A production tool is a governed capability with an owner, lifecycle, evaluation
set, release policy, and operational controls.

```text
Design
→ Review
→ Offline evaluation
→ Shadow traffic
→ Limited read-only release
→ Controlled action enablement
→ Continuous monitoring
→ Revision or rollback
```

## 2. Tool Registry Metadata

```ts
type ToolRegistration = {
  toolName: string;
  domain: string;
  owner: string;
  descriptionVersion: string;
  inputSchemaVersion: string;
  outputSchemaVersion: string;
  riskLevel: "low" | "medium" | "high";
  operationType: "read" | "write";
  authorizationPolicy: string;
  idempotencyPolicy?: string;
  timeoutMs: number;
  retryPolicy: string;
  fallbackPolicy: string;
  releaseStatus: "draft" | "shadow" | "canary" | "active" | "disabled";
  killSwitchKey: string;
  evaluationDataset: string;
};
```

The registry should be machine-readable and versioned in source control or an
approved configuration system.

## 3. Versioning Policy

Version separately:

- name and selection description
- input schema
- output contract
- execution behavior
- domain registry and aliases
- prompt or routing policy

A compatible description refinement may not require an API version change, but
it still changes model behavior and therefore needs evaluation and release notes.

Recommended rules:

- additive optional input field: minor schema version
- new enum value: minor only if all consumers tolerate unknown values
- renamed or removed field: major version
- changed side effect or authorization: major behavior version
- alias or routing-priority change: registry version and route regression test

## 4. Evaluation Dataset

A useful dataset includes more than happy paths.

```ts
type ToolEvaluationCase = {
  caseId: string;
  userText: string;
  structuredContext?: Record<string, unknown>;
  expectedDomain: string | "none";
  allowedTools: string[];
  forbiddenTools: string[];
  expectedArguments?: Record<string, unknown>;
  expectedMissingArguments?: string[];
  expectedBehavior:
    | "call_tool"
    | "clarify"
    | "answer_without_tool"
    | "reject";
  riskTags: string[];
};
```

Include:

- explicit requests
- implicit intent
- ambiguous language
- missing identifiers
- conflicting structured context
- cross-domain near duplicates
- negative cases where no tool should run
- prompt injection attempts
- unauthorized mutation requests
- stale history that conflicts with the current resource
- locale and paraphrase variants

## 5. Core Metrics

### Selection

```text
Tool selection accuracy
Domain classification accuracy
No-tool precision and recall
Cross-domain mismatch rate
Candidate shortlist recall
```

### Arguments

```text
Exact argument match
Required-field completion rate
Identifier-type mismatch rate
Enum validity rate
Unit and format accuracy
Runtime validation failure rate
```

### Execution and safety

```text
Tool success rate
Timeout and retry rate
Unauthorized-action rejection rate
Idempotency conflict rate
Fallback rate
Unsafe side-effect rate
```

### User and cost outcomes

```text
Human correction rate
Task completion rate
Clarification rate
Latency per successful task
Token cost per successful task
Tool calls per successful task
```

Do not optimize selection accuracy alone. A router can select the right domain
and still construct unsafe arguments or execute an unauthorized action.

## 6. Domain Mismatch

A mismatch occurs when trusted current context and the selected tool disagree.

```text
current structured domain = claimable_grant
selected tool domain      = redeemable_voucher
result                     = domain mismatch
```

Reference formula:

```text
domain_mismatch_rate = mismatched_calls / calls_with_trusted_domain
```

Segment by:

- model and model version
- prompt version
- router version
- registry version
- surface
- domain pair
- tool risk level

A small overall rate can hide a severe mismatch between two specific high-risk
domains.

## 7. Trace Model

```ts
type ToolTrace = {
  requestId: string;
  sessionId?: string;
  route: {
    registryVersion: string;
    declaredDomain?: string;
    selectedDomain: string;
    routeSource: string;
    confidence?: number;
    candidateTools: string[];
  };
  call: {
    toolName: string;
    inputSchemaVersion: string;
    argumentsHash: string;
    validationResult: "pass" | "fail";
    authorizationResult: "allow" | "deny";
    startedAt: string;
    durationMs: number;
    outcome: string;
  };
  result: {
    outputSchemaVersion: string;
    source: string;
    freshnessMs?: number;
    allowedAction?: string;
  };
};
```

Avoid logging raw secrets or sensitive personal data. Hash or redact arguments
according to the data-classification policy.

## 8. Rollout Strategy

### Stage 0: Offline

- schema lint
- contract tests
- routing regression tests
- adversarial and injection tests

### Stage 1: Shadow

The new router or schema observes real traffic without controlling actions.
Compare selected domains, arguments, latency, and cost with the current path.

### Stage 2: Read-only explanation

Allow live read tools and explanations, but keep operational actions on the
existing deterministic path.

### Stage 3: Limited action enablement

Enable low-risk actions for a small cohort with strict authorization and audit.

### Stage 4: Controlled rollout

Increase by domain and risk level, not only by global percentage.

### Stage 5: Full release with rollback

Maintain:

- domain-level kill switch
- tool-level disable flag
- registry rollback
- prompt and model rollback
- fallback to deterministic workflow

## 9. Security and Authorization

Tool descriptions are not security controls.

For every write tool:

- authenticate the caller
- authorize the actor, resource, and action server-side
- validate arguments against the current system of record
- use an idempotency key
- apply rate limits and abuse controls
- record an immutable audit event
- support safe retry semantics
- require confirmation or approval when appropriate

Separate read and write tools. A broad `manage_resource` tool is harder to
reason about, authorize, and evaluate than explicit read and mutation tools.

## 10. Fallback Policy

Fallback should be explicit and safe:

```text
invalid arguments
→ do not call backend; request missing context or fail closed

authorization denied
→ do not retry with altered arguments; explain allowed next step

timeout or dependency failure
→ retry only under policy; otherwise return temporary-unavailable artifact

unknown domain
→ do not map to the nearest tool; clarify or use a non-operational response

output contract failure
→ quarantine result, emit alert, and avoid generating an action from it
```

## 11. Incident Triage

When tool behavior regresses, inspect in this order:

1. trusted context and source authority
2. router and registry versions
3. model and prompt versions
4. candidate tool set
5. generated arguments and schema validation
6. authorization decision
7. backend result and output validation
8. artifact/action mapping
9. release cohort and feature flags

Common fixes should be applied at the narrowest correct layer. Do not patch a
backend policy error with a longer model description.

## 12. Governance Review Checklist

- [ ] Tool has an owner and risk level.
- [ ] Read and write operations are separated.
- [ ] Input and output schemas are versioned.
- [ ] Authorization and idempotency are server-side.
- [ ] Registry aliases and route priority are versioned.
- [ ] Evaluation includes negative and cross-domain cases.
- [ ] Traces connect route, call, result, and final artifact.
- [ ] Sensitive arguments are redacted or hashed.
- [ ] Shadow and canary stages exist.
- [ ] Domain-level kill switch and rollback are tested.
- [ ] Cost is measured per successful task.
- [ ] Output contract failure fails closed for risky actions.
