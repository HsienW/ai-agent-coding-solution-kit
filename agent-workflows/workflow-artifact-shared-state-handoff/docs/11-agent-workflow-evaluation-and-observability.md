# 11 | Agent workflow evaluation and observability

[English](./11-agent-workflow-evaluation-and-observability.md) | [繁體中文](./11-agent-workflow-evaluation-and-observability-zh-TW.md)

This document defines how a multi-agent workflow evaluates artifacts, agent stages, and the end-to-end execution flow. It also defines how Metrics, Events, Traces, Evidence, and Audit Records retain enough signal to assess quality and operational health. The intended readers are Agents, Coordinators, Orchestrators, Reviewers or Evaluators, Runtime or Telemetry Hosts, and Human Approvers.

A Workflow Run that reports `COMPLETED` has reached the end of its State Machine. The team must still determine whether the output satisfies the Requirement, whether the Agent used the correct Context, whether Retry concealed unstable behavior, and whether a new Prompt, Router, or Workflow Policy caused a regression in a high-risk scenario. Evaluation supplies a repeatable comparison method. Observability records runtime signals. Together, they support Release Gates, diagnosis, and governance.

This document follows these principles:

- Evaluation pins the Subject, Dataset, Rubric, Policy, versions, and environment so that results remain comparable.
- Security, authorization, integrity, and required Gates are Hard Invariants. An Aggregate Score cannot offset their failure.
- An Evaluation Result supplies Decision Input. The Orchestrator still updates the Accepted Pointer under Policy, Gate, and CAS controls.
- Metrics, Events, Traces, Evidence, and Audit Records retain data at distinct levels of detail.
- The current Schemas remain unchanged. New records belong in Trace, the Host Journal, or an independent Artifact connected by Reference.

## 1. Scope

This document extends the existing specifications:

- [`03-runtime-storage-and-retention.md`](./03-runtime-storage-and-retention.md) defines the storage boundaries for Durable Knowledge, Runtime State, Evidence, and Trace.
- [`05-security-and-maintainability.md`](./05-security-and-maintainability.md) defines Evidence Provenance, Artifact Integrity, Human Gates, and operational checks.
- [`06-context-authority-and-retrieval-policy.md`](./06-context-authority-and-retrieval-policy.md) defines Context Authority, Freshness, Integrity, Binding, and Budget.
- [`07-role-capability-and-scope-control.md`](./07-role-capability-and-scope-control.md) defines Runtime Identity, Capability, Policy Version, and Authorization Trace.
- [`08-mutation-concurrency-and-conflict-resolution.md`](./08-mutation-concurrency-and-conflict-resolution.md) defines Revision, CAS, Mutation Receipts, Retry, and Conflict.
- [`09-context-promotion-and-rollback.md`](./09-context-promotion-and-rollback.md) defines Promotion, Rollback, Gate Invalidation, and the Accepted Pointer.
- [`10-provenance-audit-and-incident-replay.md`](./10-provenance-audit-and-incident-replay.md) defines the Provenance Graph, Audit Reconstruction, Replay, and Drift classification.

This document covers:

- Evaluation layers for Artifact Quality, Agent or Stage Capability, and Workflow Operations.
- Offline Evaluation, Shadow Evaluation, Canary Observation, Online Monitoring, and Replay Evaluation.
- Evaluation Datasets, Baselines, Candidates, Cohorts, Rubrics, Hard Invariants, and Verdicts.
- The responsibilities of Metrics, Normalized Events, Distributed Traces, Evidence, and Audit Records.
- Regression, Drift, SLI, SLO, Alerting, Sampling, Retention, and Redaction.
- A minimum local evaluation loop in `.agent-runtime` and criteria for upgrading Production Telemetry.

The following topics remain in other specifications:

- The Current State Phase Enum, Transition Table, and Gate fields.
- The internal format of Prompt Package assets such as Prompts, Schemas, Examples, Validators, and Lockfiles.
- Implementations tied to a model provider, observability vendor, dashboard product, or billing API.
- Organizational performance reviews, employee scoring, and business KPIs.

## 2. Terms and operational boundaries

### 2.1 Validation

Validation checks whether one input, output, or Candidate satisfies an explicit Contract, such as a JSON Schema, Domain Rule, Checksum, Target Binding, test command, or Gate precondition. Validation commonly produces `pass`, `fail`, `partial`, or `not_run` and binds the result to one fixed Target.

A Validation Failure MUST control the Flow. A Runner MUST NOT write the error to a Log and then execute an Artifact Node or external Tool that depends on the failed result.

### 2.2 Evaluation

Evaluation uses a versioned Dataset and Rubric to measure the behavior of a fixed Subject across Cases, Cohorts, or samples of live traffic. A Subject can be an Agent, Prompt Package, Router, Context Policy, Workflow, Tool Set, Model Configuration, or a Release Unit composed of several of these assets.

Evaluation determines whether a Candidate improved, regressed, or lacks sufficient evidence relative to a Baseline. It does not perform Promotion or replace a Reviewer Verdict or Human Approval.

### 2.3 Observability and Telemetry

Observability is the team's ability to infer internal system state from execution signals. Telemetry consists of Metrics, Events, Traces, and Resource Usage emitted by the Runtime. Full stdout, a Raw Prompt, or a complete Tool Payload does not gain a right to retention merely because it is labeled Telemetry. Data Classification, Redaction, and Retention Policies still apply.

### 2.4 Evidence, Trace, and Audit Record

- Evidence supports an engineering claim, such as a specific Diff passing a named test or an Evaluation Suite completing.
- Trace retains diagnostic detail and causal links for one Run. It may use shorter Retention and controlled Sampling.
- An Audit Record retains stable facts required to reconstruct Decisions, Authorizations, Gates, and State Mutations.

An Evaluation Report can reference Evidence, Traces, and Audit Records. An Aggregate Metric cannot replace Target Binding.

### 2.5 Baseline, Candidate, and Cohort

- A Baseline is a fixed comparison reference, such as the current Stable Release, a prior Accepted Artifact, or an approved Policy Snapshot.
- A Candidate is the Release Unit under evaluation. Its contents remain fixed during execution.
- A Cohort groups Cases or Runs that share conditions such as Domain, Risk Level, Repository, Workflow Stage, Model Version, language, or Release Mode.

A global average can conceal a small high-risk Cohort. An Evaluation Report MUST retain segmented results and sample counts.

### 2.6 Regression and Drift

A Regression occurs when a Candidate performs below the Baseline under comparable conditions or violates an established Invariant. Drift is a change over time in input distribution, Dependencies, Models, Tools, Policies, Latency, or output characteristics. One failure can trigger diagnosis, but it does not establish statistical Drift by itself.

## 3. Roles and responsibilities

| Role | Responsibilities | Prohibited actions |
|---|---|---|
| Coordinator | Defines the evaluation objective, Subject, Risk Cohort, Rubric, Hard Invariants, and allowed variation | MUST NOT ignore a high-risk regression because a global Aggregate Score improved; MUST NOT alter original Case Results |
| Orchestrator | Pins versions, creates the Evaluation Manifest, runs or schedules the Suite, verifies Binding, and applies Gate Policy | MUST NOT rewrite an Evaluator Verdict, evaluate a mutable Candidate, or relax a Threshold without authorization |
| Agent or Producer | Produces contract-valid Artifacts, Evidence, and minimum Telemetry | MUST NOT declare its output globally accepted, update the Accepted Pointer, or store private reasoning in Telemetry |
| Reviewer or Evaluator | Performs read-only checks of the Subject, Evidence, Rubric, and Case Results; records Verdicts, limits, and uncertainty | MUST NOT modify Source, Candidate, Current State, Gates, or Approval |
| Runtime or Telemetry Host | Creates Identity, collects Events and Traces, applies Redaction, aggregates Metrics, and enforces Retention | MUST NOT export Secrets or complete sensitive Payloads to a Telemetry Backend; MUST NOT treat a sampled Trace as a complete Audit |
| Human Approver | Approves high-risk Thresholds, Releases, data access, and Policy exceptions | MUST NOT replace an Approval Record with verbal consent or treat a Reviewer Verdict as authorization for an external Side Effect |

`Evaluator` and `Telemetry Operator` are responsibility labels. They do not add Roles to the current Schemas. A concrete Runtime Identity uses `reviewer`, `automation`, or a Host Identity authorized under document 07.

## 4. Evaluation and observability invariants

Every Workflow Host MUST enforce these conditions:

1. Each Evaluation has a unique `evaluationId` and pins the Subject, Dataset, Rubric, Policy, and environment identity.
2. The Baseline and Candidate use comparable Release Units. A result records compatibility limits when their version sets differ.
3. A Dataset remains Immutable during execution. Adding an Incident Case creates a new Version.
4. Each Case Result traces to its Input, Expected Invariants, Actual Output, Evaluator, and Evidence.
5. Security, authorization, Schema Integrity, Target Binding, and external Side Effect controls fail closed.
6. An Aggregate Score cannot override a Hard Invariant Failure.
7. Evaluation execution status and quality Verdict remain separate. Successful Suite execution does not mean that a Candidate passed.
8. A Producer MUST NOT accept or Promote its own Candidate.
9. A Reviewer or Evaluator Verdict does not update Current State. The Orchestrator submits a Mutation under Policy and Expected Revision.
10. Metrics retain aggregate trends. Events and Traces retain Run, Artifact, and Invocation Identity.
11. Events, Traces, Evidence, and Audit use a shared `changeId`, `runId`, `stage`, and required References for correlation.
12. Timestamps provide time information. Parent References, Span Links, Revisions, and Target Bindings establish causality.
13. Telemetry passes Redaction before leaving the Runtime Trust Boundary.
14. Sampling MUST NOT remove the Audit Records required for Incidents, Policy Denials, Human Overrides, Integrity Failures, or Unknown Side Effect Outcomes.
15. Missing Evaluation or Observability data produces `inconclusive`, `INCOMPLETE`, or a fixed failure code. The system MUST NOT substitute the most recent available data.
16. A Terminal Run remains closed. Later Evaluation, Replay, or regression analysis creates new Identities and Artifacts.

## 5. Three evaluation layers

### 5.1 Artifact quality

Artifact Quality checks whether a deliverable satisfies approved Requirements, Schemas, Domain Rules, and engineering quality criteria. Typical signals include:

- Requirement and Acceptance Criteria Coverage.
- Schema and Domain Validation results.
- Test, Lint, Build, and Diff checks.
- Review Finding Severity, Status, and Target Binding.
- Unrun validation, Residual Risk, and pending human judgment.

The evaluation unit for Artifact Quality is a fixed URI, Checksum, Diff Hash, or Commit. Results MUST NOT carry over when a reused filename points to different content.

### 5.2 Agent or stage capability

Agent or Stage Capability checks whether a Role completed assigned work within approved Scope:

- The Router selected the correct Domain and Skill.
- The Context Builder used the correct Authority, Version, and Budget.
- The Agent followed the Output Contract, Tool Policy, and Handoff Scope.
- A Validation Failure reached the correct Fallback.
- Evidence is complete and Reason Codes are machine-readable.
- Retry, Clarify, Escalation, and Human Review follow Policy.

This layer evaluates Agent behavior within a specific Stage. Reviewer read-only work and Implementer mutation capability require different denominators.

### 5.3 Workflow operations

Workflow Operations measures end-to-end reliability and cost:

- Run Completion, Accepted Outcome, and Rework.
- Stage Latency, Handoff Wait, Human Gate Wait, and Queue Time.
- Retry, Timeout, Conflict, Rollback, and Incident rates.
- Schema Invalid, Context Stale, State Revision Conflict, and Partial Write events.
- Token, Model Call, Tool Call, Storage, and Telemetry costs.

A Workflow can reach `COMPLETED` and then require an immediate Rollback. Completion Rate alone gives the wrong conclusion, so Outcome metrics include Acceptance, Regression, and Incident windows.

## 6. Evaluation subject and version binding

### 6.1 A Subject can be one asset or a Release Unit

Common Subjects include:

| Subject | Minimum version identity |
|---|---|
| Prompt Package | Prompt, Schema, Example, Eval Dataset, Validator, and Lockfile Hash |
| Router | Router Code or Policy, Domain Registry, Alias, Threshold, and Model Version |
| Context Builder | Builder Version, Retrieval Policy, Allowed Fields, Budget, and Source Index Version |
| Agent | Agent or Skill Version, Prompt Package, Tool Set, Model or Adapter, and Capability Policy |
| Workflow | Workflow Definition, Node Implementation, Fallback Policy, and Transition Policy |
| Tool | Tool, Schema, and Behavior Version; Authorization, Timeout, Retry, and Idempotency Policy |
| Release Unit | A compatible set of the assets above plus an Environment Reference |

Recording only `model=gpt-x` or `prompt=latest` cannot support a comparable Evaluation. The Manifest MUST point to an exact Snapshot or verifiable Hash.

### 6.2 Minimum binding

An Evaluation Manifest pins at least:

```text
changeId / runId
evaluationId
subjectRef / subjectHash
baselineRef / baselineHash
datasetRef / datasetVersion / datasetHash
rubricRef / rubricVersion
evaluationPolicyVersion
model / adapter / tool versions
workflow / router / prompt lockfile versions
environmentRef
risk cohort and release mode
```

The Evaluator records a limitation when any output-affecting version is missing. When the Baseline and Candidate use different Datasets or Rubrics, the system MUST NOT calculate a Regression Percentage directly.

### 6.3 Subject changes invalidate results

The following changes require a new Evaluation:

- A change to a Prompt, Example, Schema, Validator, or Lockfile.
- A change to Router Priority, Confidence Threshold, or the Domain Registry.
- A change to Allowed Context Fields, Retrieval Policy, Index, or Budget.
- A change to a Model, Adapter, Tool, Dependency, or Environment that Policy classifies as Material.
- A change to a Workflow Node, Condition, Fallback, Retry, or Gate.
- A change to a Candidate Artifact, Diff Hash, or Source Base.

Policy may allow results to survive non-semantic Metadata changes, but it MUST list the allowed fields. A Producer cannot make that decision.

## 7. Evaluation modes

| Mode | Data source | Controls formal Actions | Primary use |
|---|---|---:|---|
| Contract Test | Fixed Fixtures | No | Schemas, Reason Codes, Transitions, and Bindings |
| Offline Evaluation | Versioned Dataset | No | Baseline and Candidate Regression |
| Simulation | Synthetic environment and Tool Stubs | No | Timeouts, Conflicts, Fallbacks, and Side Effect controls |
| Replay Evaluation | Pinned original Input, Policy, and environment | No | Decision or Validation recomputation and Incident analysis |
| Shadow Evaluation | Parallel Candidate on live inputs | No | Route, Output, Latency, and Cost comparison |
| Canary Observation | Limited live traffic | Conditional | Runtime behavior and Rollback conditions |
| Online Monitoring | Production Runtime Telemetry | Existing flow | Error, Drift, Cost, and SLO Breach detection |
| Human Benchmark | Sampled and blinded Cases | No | Semantic quality, risk, and LLM Evaluator calibration |

Shadow, Simulation, and Replay deny Commit, Push, Deploy, Delete, and external API Writes by default. A Canary that allows an Action still requires Authorization from document 07, Idempotency from document 08, and Approval Binding from document 09.

## 8. Evaluation datasets and cases

### 8.1 Dataset contents

Each Evaluation Case describes at least:

```text
caseId
input reference or fixture
task class / domain / stage
risk level and tags
expected route or allowed routes
required invariants
allowed output variation
forbidden behavior
required evidence
evaluation method
```

A Case may accept several semantically equivalent outputs. Before requiring exact string equality in Expected Output, the team confirms that the output is deterministic. Free-form text, summaries, and code changes usually require a combination of Invariants, Validators, Reference Outputs, and a human Rubric.

### 8.2 Minimum case groups

A Dataset SHOULD include:

- Normal, Boundary, Missing Context, and Invalid Input cases.
- Cross-domain Similarity, Low-confidence Route, and Clarification cases.
- Stale or Conflicting Context, Wrong Authority, and Budget Exhaustion cases.
- Outputs that are Schema Valid and Domain Invalid.
- Prompt Injection, Scope Denial, and Unauthorized Action cases.
- Tool Timeout, Invalid Tool Output, Duplicate Request, and Unknown Side Effect Outcome cases.
- Review Finding, Validation Binding Mismatch, Retry Exhaustion, and Human Reject cases.
- Parallel Mutation, State Revision Conflict, Rollback, and Incident-derived cases.

### 8.3 Dataset sources and governance

Cases can come from synthetic data, anonymized live Runs, Production Incidents, or human-authored Adversarial Scenarios. After resolving an Incident, the team SHOULD promote its minimum reproduction into a new Case and remove Secrets, personal data, private Repository content, and replayable Credentials.

A Dataset requires an Owner, Version, Change History, Data Classification, and Retention. Changes to an Expected Result require Review. Relaxing an answer to make a Candidate pass is a Dataset change and cannot be mixed into the same Candidate comparison.

### 8.4 Leakage and circular evaluation

A model that receives complete Eval Answers through a Prompt, Few-shot Example, or Retrieval Source may produce inflated results. Teams SHOULD separate Training or Tuning Cases, Development Cases, and Release Gate Cases, and prevent Agents from enumerating hidden Datasets.

When a Producer and Evaluator use the same model, the Result MUST record that limitation. High-risk semantic decisions SHOULD combine Deterministic Checks with a different Evaluator or Human Calibration so that one model does not grade its own preferences.

## 9. Baseline, Candidate, and Cohort

### 9.1 Baseline eligibility

A Baseline SHOULD meet these conditions:

- It has a fixed URI, Hash, Release Unit, and Environment Reference.
- Its Dataset, Rubric, and Metric Definitions remain available.
- Known limitations and Incidents are recorded.
- Retention has not removed data required for Binding.

The previous version is not always a valid Baseline. If it uses a different Domain, Schema, or incompatible environment, the Coordinator selects a comparable version or marks the result `baseline-incompatible`.

### 9.2 Cohort dimensions

At minimum, evaluate whether to segment by:

```text
domain / task class
risk level
workflow stage
repository or component
model / prompt / router / policy version
tool operation type
language / locale
release mode / environment
```

Each Cohort retains its sample count, successes, failures, and confidence limits. When the sample is too small, the Result uses `inconclusive`; a percentage must not create false certainty.

## 10. Rubric, hard invariants, and verdict

### 10.1 Rubric structure

A Rubric can include:

- Deterministic Checks for Schemas, Checksums, Enums, Target Bindings, and Exit Codes.
- Rule-based Checks for Domain Rules, Scope, Forbidden Behavior, and Evidence Completeness.
- Semantic Evaluation for Requirement Coverage, grounded answers, and Review Quality.
- Operational Thresholds for Latency, Retry, Cost, Timeout, and Error Rate.

Each item needs a name, version, applicable Cohort, calculation, data source, Threshold, and failure route. When free-form prose cannot be machine-read, provide a Reviewer Checklist and Evidence Reference.

### 10.2 Hard invariants

The following conditions cannot be offset by a weighted average:

- Unauthorized Action or Scope Violation.
- Secret or Credential Leakage.
- Schema, Checksum, Provenance, or Target Binding Failure.
- A downstream Action running after the Domain Validator failed.
- An unresolved Blocker or Major Finding under the applicable Policy.
- Missing Required Evidence.
- An Unknown Side Effect Outcome without Reconciliation.
- An external Write during a Replay or Shadow Invocation.

Any Hard Invariant Failure produces an Evaluation Verdict of `fail` or `inconclusive`. Policy MUST NOT release the Candidate because its Overall Score met a threshold.

### 10.3 Completion status and verdict

Evaluation execution status:

```text
completed
partial
failed_to_run
cancelled
```

Evaluation quality verdict:

```text
pass
fail
inconclusive
not_comparable
```

`completed + fail` means the Suite ran completely and the Candidate did not pass. `failed_to_run + inconclusive` means infrastructure or required data was unavailable. A single `success` field MUST NOT combine these dimensions.

## 11. Evaluator selection and calibration

### 11.1 Evaluation method priority

Use a Deterministic Validator for mechanical facts. When multiple semantic answers are valid, use a Rule-based Evaluator, Reference Comparison, LLM Evaluator, or Human Review. Evaluator flexibility and cost do not replace repeatable Contract Checks.

### 11.2 LLM evaluator

When an LLM evaluates semantic quality, the Manifest pins:

- Evaluator Model, Version, Prompt, Rubric, and Temperature.
- Whether Candidate Identity is blinded.
- Input fields, Context Sources, and Redaction Policy.
- Repeated-run, Majority or Aggregation, and Tie-breaker rules.
- The date and delta from Human Benchmark calibration.

The Evaluator SHOULD emit structured Reason Codes, Criterion Results, Evidence References, and Confidence. Long-form commentary can explain a result, but it cannot serve as the only Gate Input.

### 11.3 Independence

A Producer may run self-checks and publish them as Candidate Evidence. A formal Release Gate requires an independent Reviewer, Evaluator Host, or Deterministic Suite. The Reviewer remains read-only, while the Orchestrator retains Transition Authority.

## 12. Lifecycle stage evaluation matrix

| Stage | Evaluation Subject | Required checks | Primary signals |
|---|---|---|---|
| `plan-change` | Requirement or Plan | Scope, Constraints, Open Questions, and testability | Requirement Coverage, Clarification Rate |
| `review-plan` | Plan Candidate | Design Conflict, Risk, and Acceptance Criteria | Finding Severity, Approval or Change Request |
| `apply-change` | Implementation Candidate | Changed Files, Diff, Validation, and Scope Deviation | Test Pass, Schema Invalid, Retry, and Duration |
| `review-result` | Candidate and Evidence | Target Binding, Findings, and Residual Risk | Review Verdict, Correction Rate, and Evidence Completeness |
| `fix-from-review` | Revised Candidate | Finding Closure, Regression, and Diff Delta | Rework Attempt and Reopened Finding |
| `readiness-check` | Accepted Candidate | Gates, Unresolved Items, and Archive Eligibility | Gate Pass, Human Wait, and Incomplete Rate |
| `archive-change` | Execution Summary | Durable Facts, References, and Remaining Work | Summary Completeness and Archive Failure |
| `manual-gate` | Fixed Action or Payload | Approval Binding, Scope, and Expiry | Approve or Reject, Wait Time, and Invalidation |

The current Lifecycle has no `evaluate-workflow` Stage. A Host Workflow runs an independent Evaluation Suite. When a Reviewer or Readiness Agent needs the result, the Host passes it by Reference into an existing Stage.

### 12.1 Prompt runtime nodes

For the Runtime Execution described in [`ai-agent-prompt-runtime-workflow-phase2-zh-TW.md`](../../../ai-agent-prompt-runtime-workflow-phase2-zh-TW.md):

| Node | Evaluation question | Suggested metric or Invariant |
|---|---|---|
| Runtime Input | Is the Input Contract and Trust Boundary complete? | Input Contract Failure, Untrusted Field Rejection |
| Domain Router | Did it select the correct Domain, Skill, and Reason? | Route Accuracy, Domain Mismatch, Clarify Precision |
| Skill Loader | Did it resolve a pinned Prompt Package and Validator? | Package Binding, Lockfile Match |
| Context Builder | Did it use minimal and authoritative Context? | Authority Coverage, Source Overuse, Token by Source |
| Prompt Assembly | Did it pin the Model, Prompt, Examples, and Schema? | Snapshot Completeness, Version Missing |
| Schema Validator | Does the Output have the correct shape? | Schema Failure, Compact Retry Success |
| Domain Validator | Do semantic and cross-field rules hold? | Domain Failure, Cross-domain Contamination |
| Artifact Transformer | Did it run only after the Spec passed? | Gate Bypass Count MUST equal 0 |
| Fallback | Did the Failure reach the correct exit? | Fallback Accuracy, Retry Exhaustion |
| Release | Did Formal Flow pass Evaluation, Approval, and Canary? | Gate Coverage, Rollback Trigger |

## 13. Signal model

| Signal | Retained data | Suitable questions | Responsibilities it does not carry |
|---|---|---|---|
| Metrics | Aggregate counts, ratios, distributions, and Resource Usage | Trends, SLOs, Alerts, and Cohort comparisons | Complete causality for a single Run |
| Event or Log | Normalized Events, Status, Reason Code, and safe summaries | What occurred at a point in time | Large Raw Payloads and durable Decision Evidence |
| Trace or Span | Causality and Timing across Runs, Stages, Invocations, and Tools | Latency location, Retry, Fallback, and cross-service diagnosis | Permanent Audit and Accepted Evidence |
| Evidence | Commands, validation, and Review results bound to a Target | Whether a claim is verifiable | Complete execution detail and platform trends |
| Audit Record | Authorization, Gate, Mutation, and stable References | Who made a Decision under which Policy | High-volume Debug Payloads |

One Failure may emit a Metric, Event, Trace, and Evidence. Each signal retains only the fields required for its job. Copying complete content increases Secret exposure, Retention cost, and inconsistency.

## 14. Trace and correlation

### 14.1 Span structure

Recommended causal structure:

```text
Workflow Run Span
└─ Stage Span
   └─ Agent Invocation Span
      ├─ Context Resolution Span
      ├─ Model Call Span
      ├─ Tool Action Span
      ├─ Validation Span
      └─ Artifact Publication Span
```

Handoffs, Queues, Async Workers, Shadow Candidates, and Cross-run Evaluations may not have a direct Parent or Child relationship. The Host uses a Span Link or explicit Reference to connect the source Run and MUST NOT invent a parent-child relationship.

### 14.2 Identity

Trace follows the Identity separation from document 10:

```text
changeId       engineering change scope
runId          one Workflow Run
invocationId   one Agent Invocation
traceId        one diagnostic correlation set
spanId         one operation within a Trace
evaluationId   one Evaluation Execution
caseRunId      one execution of an Evaluation Case
mutationId     one controlled mutation
```

A Retry creates a new `invocationId` or `caseRunId` and references the original Attempt. Reusing one ID prevents correct separation of Latency, Token usage, and Failure counts.

### 14.3 Minimum span attributes

```text
changeId / runId / stage
actorId / role
workflow / agent / prompt / model / adapter / tool versions
policyVersion / stateRevision
operation / status / reasonCode
startedAt / endedAt / durationMs
inputRef hashes / outputRef hashes
retryAttempt / releaseMode / riskLevel
```

Span Attributes MUST NOT contain Raw Prompts, complete Source Files, Secrets, Credentials, Approval Proof, or private model reasoning.

## 15. Metrics

### 15.1 Outcome

```text
accepted_task_rate = accepted tasks / eligible tasks
rework_rate = tasks entering fix-from-review / reviewed tasks
rollback_rate = rollback changes / released changes
incident_rate = declared incidents / released changes
```

A Metric Definition explicitly defines `eligible tasks`, `accepted tasks`, and the observation window. Cancelled Runs, test Runs, and Runs with missing required data do not silently enter the denominator.

### 15.2 Quality and capability

- Requirement Coverage, Validation Pass, and Evidence Completeness.
- Route Accuracy, Domain Mismatch, and Clarification Accuracy.
- Context Authority Coverage, Freshness Failure, and Source Overuse.
- Review Finding Severity, Human Correction, and Unsupported Claims.
- Tool Selection, Argument Validation, and Authorization Denial.

### 15.3 Flow and reliability

- p50, p95, and p99 for Run and Stage Latency.
- Queue, Handoff, Human Gate, and Dependency Wait.
- Retry Attempt, Retry Amplification, Timeout, and Fallback Success.
- Schema Invalid, State Conflict, Partial Write, Lease Expiry, and Unknown Outcome.
- Checkpoint Resume, Cancel, Abandoned Run, and Orphan Candidate.

An average cannot describe long-tail waits. Latency and Cost retain a distribution or Percentiles.

### 15.4 Governance and safety

- Unauthorized Action Rejection.
- Scope Violation, Policy Version Mismatch, and Approval Invalidation.
- Secret or Sensitive Data Redaction Failure.
- Replay Side Effect Blocked.
- Gate Bypass, Accepted Pointer Conflict, and Provenance Gap.

An increase in Denials may come from an attack, a Policy change, or expected risk control. Alerts need drill-down by Cohort and Reason Code; they MUST NOT classify every Denial as an Agent Failure.

### 15.5 Efficiency

```text
cost_per_accepted_task
tokens_per_accepted_artifact
tool_calls_per_successful_stage
retry_cost_amplification
context_bytes_or_tokens_by_source
cache_hit_rate
```

Accepted Tasks or Successful Stages provide the primary denominator for cost. Cost per Run alone can decline when the system fails early, without improving delivery efficiency.

## 16. Metric labels and cardinality

Suitable Metric Labels:

```text
environment
workflow_id / workflow_version
stage
domain
risk_level
release_mode
status
reason_code
model_family / tool_id
```

The following Identities usually have high cardinality and belong in Events, Traces, or Exemplars:

```text
changeId
runId
invocationId
artifact URI / checksum
userId / repository path
raw error message
prompt or tool payload
```

The Runtime Host limits Label Length, Value Sets, and unknown values. A free-form Error used as a Label increases storage cost and may export sensitive content outside the Trust Boundary.

## 17. Regression and drift

### 17.1 Comparability checks

Before calculating Candidate Delta, the Evaluator compares:

- Dataset, Rubric, Metric Definition, and Threshold Version.
- Domain, Risk, Language, Repository, and Release Cohort.
- Model, Prompt, Router, Tool, Policy, and Environment.
- Sample Size, Sampling Method, Timeout, and Retry Policy.

When differences exceed the Comparison Policy, the Result uses `not_comparable`. A report may show the raw values side by side, but it cannot claim a percentage improvement.

### 17.2 Regression decisions

A Regression Policy includes:

- Hard Invariants that block Promotion on any failure.
- Absolute Thresholds, such as an upper bound for Domain Mismatch.
- Relative Delta allowed between the Candidate and Baseline.
- Cohort Rules with stricter conditions for high-risk Domains.
- Minimum Sample requirements that return `inconclusive` when unmet.

### 17.3 Drift

Online Drift Detection records the observation window, Reference Window, Cohort, Feature or Metric Definition, and Confidence. After detecting Drift, the Orchestrator creates diagnostic work or a Candidate Evaluation. It does not rewrite a Router, Prompt, Threshold, or Context Policy directly.

Replay Drift in document 10 concerns pinned Incident inputs and an original Decision. Online Drift in this document concerns execution distributions over a time window. They may share Traces and Comparison Policies, but their conclusions remain scoped to their respective evaluations.

## 18. SLI, SLO, and error budget

### 18.1 SLI

An SLI must derive from explicit Events, for example:

```text
workflow_acceptance_sli
stage_completion_latency_sli
required_evidence_availability_sli
state_mutation_success_sli
high_risk_gate_bypass_sli
```

Statements such as `high quality`, `smart Agent`, or one LLM Judge score are unsuitable operational SLIs.

### 18.2 SLO

Each SLO defines:

- The Service or Workflow and applicable Cohort.
- The SLI Formula, data source, and observation window.
- The Target, Excluded Events, and Minimum Traffic.
- The Error Budget, Alert Policy, and Owner.
- The Actions permitted after a Breach.

Safety Invariants such as an external Write Gate Bypass have a Target of 0 and do not use a general Error Budget. Latency, Availability, and non-high-risk Retry may use Budgets appropriate to the service.

### 18.3 SLO breach

An SLO Breach may pause Progressive Rollout, restrict a high-risk Cohort, restore a Stable Release, or request human judgment. It does not authorize the Orchestrator to change Requirements, Rubrics, Permissions, or Human Approval Policy.

## 19. Alerts and routing

### 19.1 Alert conditions

Alerts are appropriate for:

- Hard Invariant, Integrity, Security, or Gate Bypass failures.
- Error Rate, Latency, Retry, or Cost exceeding a Threshold during an observation window.
- Missing data or Redaction Failure in the Telemetry Pipeline.
- Sustained Candidate Canary Regression relative to a Stable Cohort.
- Missing Provenance, Audit, or Required Evidence.

The Workflow Fallback handles an ordinary Validation Failure, so each such event does not need an Operator notification. An Alert Policy sets its Window, Minimum Count, Deduplication Key, Cooldown, Severity, and Owner.

### 19.2 Severity and default routing

| Severity | Condition | Default route |
|---|---|---|
| Critical | Unauthorized Side Effect, Secret Leakage, Gate Bypass, or Integrity Failure | Stop the related Release or Action; notify the Human or Incident Operator |
| High | High-risk Cohort Regression, rapid SLO consumption, or Required Evidence Gap | Pause Rollout; create Coordinator or Reviewer work |
| Medium | Retry, Latency, Cost, or Fallback remains outside the Baseline | Create a diagnostic item; retain safe traffic |
| Low | Trend change, low-sample Drift, or a non-blocking Telemetry Gap | Record it and schedule Review |

An Alert message contains only a safe summary, Cohort, Reason Code, Observed Value, Threshold, and Trace or Dashboard Reference. Sensitive data stays in the controlled Store.

## 20. Sampling, retention, and redaction

### 20.1 Sampling

The Host may sample detailed Traces for successful Runs while retaining Metrics and required Audit Records. The following events remain retained by default:

- Hard Invariant Failures, Policy Denials, and Approval Invalidations.
- Incidents, Rollbacks, Human Overrides, and Canary Aborts.
- Integrity, Provenance, State Mutation, and Unknown Side Effect Outcomes.
- Evaluation Regressions, Inconclusive Results, and Telemetry Redaction Failures.

Tail-based Sampling may decide to retain a full Trace after a Run completes, based on Status, Latency, or Reason Code. The Sampling Policy is versioned.

### 20.2 Retention

| Data | Recommended principle |
|---|---|
| Aggregate Metrics | Retain longer for trend and capacity analysis |
| Full Trace | Retain for a shorter period based on cost, privacy, and diagnostic windows |
| Evaluation Manifest or Report | Retain through the Release, Rollback, and Regression comparison period |
| Case Result | Retain under Dataset and compliance requirements; summaries and failed samples may remain |
| Evidence or Audit | Retain under engineering audit, Human Approval, and Incident Policies |

After Retention expires, reports identify the data gap. The system MUST NOT rebuild old numbers from a Latest Artifact with different content.

### 20.3 Redaction

Before Export, the Telemetry Pipeline handles:

- Secrets, Credentials, Cookies, Tokens, and Approval Proof.
- Personal data, customer content, private Repository excerpts, and complete Prompts.
- Tool Argument or Output fields that diagnosis does not require.
- Private model reasoning and unfiltered chat history.

When content comparison is required, retain a normalized Hash, Data Classification, and controlled Reference. A Hash MUST NOT allow a low-privilege Agent to test whether a sensitive value exists.

## 21. Evaluation gate and accepted pointer

### 21.1 Decision flow

```text
publish immutable Candidate
→ build Evaluation Manifest
→ run Evaluation Suite
→ validate Result and Evidence Binding
→ Reviewer evaluates findings and residual risk
→ Orchestrator applies versioned Gate Policy
→ Human approves high-risk Action when required
→ CAS State and Accepted Pointer
→ record Mutation Receipt, Provenance and Trace
```

An Evaluation `pass` is one Gate Input. Review, Readiness, Human Approval, State Revision, or Candidate Binding may still stop Promotion.

### 21.2 Gate invalidation

A change to the Candidate, Dataset, Rubric, Policy, Environment, or Required Evidence invalidates the related Evaluation Gate. The Orchestrator creates a new Manifest and Result. It MUST NOT replace the Subject Reference in an old Report and reuse the conclusion.

### 21.3 Current phases

The Current State Phase Enum has no `EVALUATING`, `OBSERVING`, `DEGRADED`, or `CANARY_FAILED` value. An independent Host Workflow can run Evaluation. When the main Run must expose a blocker, it uses a legal Phase such as `INCOMPLETE`, `FAILED`, `CHANGES_REQUESTED`, or `NEEDS_COORDINATOR_ARBITRATION` and includes the Evaluation result in a Blocker, Handoff Reason, or Reference.

## 22. Fixed failure codes and routing

### 22.1 Evaluation

| Code | Condition | Safe route |
|---|---|---|
| `EVALUATION_SUBJECT_MISSING` | Subject Reference or Release Unit is incomplete | `INCOMPLETE`; supply pinned versions |
| `EVALUATION_BINDING_MISMATCH` | Result, Evidence, Candidate, or Baseline points to a different Target | Stop the Gate; rebuild the Manifest |
| `EVALUATION_DATASET_UNAVAILABLE` | Dataset, Case, or Hash is unavailable | `inconclusive`; do not substitute an old Dataset |
| `EVALUATION_RUBRIC_VERSION_MISSING` | Rubric or Threshold Version is unknown | `inconclusive`; return to the Coordinator |
| `EVALUATION_POLICY_VERSION_MISSING` | Gate, Sampling, or Comparison Policy is unavailable | Stop Promotion |
| `EVALUATION_EVIDENCE_INCOMPLETE` | A required Case Result, Trace, or Evidence is missing | `INCOMPLETE`; list missing References |
| `EVALUATION_INVARIANT_FAILED` | A Hard Invariant failed | `fail`; stop or isolate according to Failure type |
| `EVALUATION_RESULT_INCONCLUSIVE` | Sample size, Evaluator agreement, or data is insufficient | Expand the controlled sample or request Human Review |
| `EVALUATION_BASELINE_INCOMPATIBLE` | Release Unit, Dataset, Rubric, or Cohort is not comparable | Return `not_comparable` |
| `EVALUATION_REGRESSION_DETECTED` | Candidate exceeds allowed regression or fails a high-risk Cohort | Pause Rollout; create a corrective Handoff |

### 22.2 Observability

| Code | Condition | Safe route |
|---|---|---|
| `TELEMETRY_BINDING_MISSING` | Event or Trace cannot link to a Run, Stage, or Subject | Isolate the data; mark report limitations |
| `TELEMETRY_REDACTION_FAILED` | The Host cannot confirm that sensitive content was handled before Export | Stop Export; retain a safe local Event |
| `TELEMETRY_CARDINALITY_LIMIT` | A Label or Attribute exceeds Policy | Drop the invalid Label; retain a fixed Code and count |
| `OBSERVABILITY_SIGNAL_GAP` | A Metric, Event, Trace, or time range is missing | `inconclusive`; do not calculate a definitive conclusion |
| `ALERT_POLICY_INVALID` | Threshold, Window, Owner, or route is incomplete | Disable the Rule and notify the Operator |
| `ALERT_THRESHOLD_BREACHED` | An SLI or Metric exceeds a versioned Threshold | Route by Severity and Release Policy |

### 22.3 Reuse of existing codes

The following conditions use codes from existing specifications:

| Condition | Code source |
|---|---|
| Context Authority, Freshness, Version, or Binding | `CONTEXT_*` codes from document 06 |
| Authorization, Scope, Approval, or Policy | Fixed denial codes from document 07 |
| State, Mutation, Retry, Conflict, or unknown external outcome | Fixed codes from document 08 |
| Promotion, Gate Invalidation, and Rollback | Fixed codes from document 09 |
| Provenance, Audit, Replay, and Incident Drift | Fixed codes from document 10 |

One condition uses one primary Reason Code. An Evaluation Report may add a category, but it does not create a synonym solely for a Dashboard.

## 23. Compatibility with current schemas

### 23.1 Handoff Envelope

[`handoff-envelope.schema.json`](../schemas/handoff-envelope.schema.json) has no `evaluationId`, `datasetVersion`, `rubricVersion`, `traceId`, or `metricThresholds`. A current Handoff can use:

- `requiredInputRefs` to point to an Evaluation Manifest, Report, Evidence, and Baseline.
- `acceptanceCriteriaRefs` to point to a Rubric, Regression Policy, and Gate criteria.
- `reason` to state the evaluation purpose, Cohort, Hard Invariants, and prohibited Side Effects.
- `onSuccess`, `onFailure`, and `onConflict` with current legal Transitions.

An independent Evaluation cannot add an `evaluate-workflow` Stage. The Host stores `evaluationId` in Invocation Metadata and Trace, then passes the Report Reference to a current Reviewer or Readiness Stage.

### 23.2 Agent Result

[`agent-result.schema.json`](../schemas/agent-result.schema.json) has no `evaluation_result` Kind. A current Reviewer or Readiness Result can:

- Reference the Evaluation Manifest and Candidate in `inputRefs`.
- Reference an independent Evaluation Report or Evidence in `outputRefs`.
- Record the Suite name, Status, Command, and Evidence Ref in `verification`.
- Express review of the Evaluation Result in `reviewPayload.reviewedArtifacts` or `readinessPayload.gates`.

Each Kind's `payload` uses `additionalProperties: false`. It cannot add undefined `metrics`, `traceId`, or `evaluationVerdict` fields. Store the full result as an independent Artifact.

### 23.3 Current State

[`current-state.schema.json`](../schemas/current-state.schema.json) retains only the current Phase, Owner, Revision, Accepted Artifacts, Handoff, Gates, Blockers, and Next Action. It does not retain Metrics History, Evaluation Datasets, Traces, or SLOs.

The current Gates have no `evaluationPassed` field. The Orchestrator uses Workflow Policy to require an Evaluation Report as Review or Readiness input. Until a formal Schema adds the Gate, the `gates` object MUST NOT contain an extra field.

### 23.4 Conceptual records

The Evaluation Manifest, Evaluation Result, Telemetry Event, and Alert Record in this document are conceptual formats. Before different Runners or Services exchange them, a team SHOULD define Versioned Schemas, Compatibility Rules, Migration Policies, and Example Fixtures.

## 24. Conceptual records

### 24.1 Evaluation Manifest

```yaml
schemaVersion: 0.1.0-draft
evaluationId: eval-structured-artifact-v2-001
changeId: prompt-runtime-v2
runId: run-eval-001
mode: offline

subject:
  type: workflow-release-unit
  ref: repo://prompt-engineering/templates/workflow.example.yaml
  version: 2.0.0
  sha256: aaaa
  promptLockRef: repo://prompt-engineering/prompts/structured-artifact-generation/prompt-lock.json

baseline:
  ref: artifact://prompt-runtime/releases/structured-artifact-v1.json
  sha256: bbbb

dataset:
  ref: repo://prompt-engineering/prompts/structured-artifact-generation/eval-cases.yaml
  version: 1.1.0
  sha256: cccc

rubric:
  ref: repo://prompt-engineering/evaluation/structured-artifact-rubric.yaml
  version: 1.0.0

policyVersion: evaluation-policy-1.0.0
environmentRef: runtime://evaluation/environments/local-001.json
cohorts:
  - normal
  - cross-domain
  - prompt-injection
  - high-risk-action
sideEffects: denied
createdAt: 2026-08-15T09:00:00Z
traceId: trace-eval-001
```

The `sha256` values are abbreviated for readability. Production records MUST use the complete format required by their adopted Schema.

### 24.2 Evaluation Result

```yaml
schemaVersion: 0.1.0-draft
evaluationId: eval-structured-artifact-v2-001
executionStatus: completed
verdict: fail

binding:
  subjectHash: aaaa
  baselineHash: bbbb
  datasetHash: cccc
  rubricVersion: 1.0.0
  policyVersion: evaluation-policy-1.0.0

summary:
  totalCases: 40
  passedCases: 39
  failedCases: 1
  inconclusiveCases: 0

invariantChecks:
  - id: domain-failure-must-stop-artifact-generation
    status: failed
    caseId: cross-domain-action-07
    code: EVALUATION_INVARIANT_FAILED

regressions:
  - cohort: cross-domain
    baselinePassRate: 1.0
    candidatePassRate: 0.875
    code: EVALUATION_REGRESSION_DETECTED

evidenceRefs:
  - evidence://prompt-runtime/evaluations/eval-structured-artifact-v2-001/case-results.json
  - runtime://prompt-runtime/telemetry/traces/trace-eval-001.json

nextAllowedAction: fix-workflow-gate
createdAt: 2026-08-15T09:08:12Z
traceId: trace-eval-001
```

### 24.3 Telemetry Event

```yaml
schemaVersion: 0.1.0-draft
eventId: evt-eval-001-domain-validation
eventType: validation-completed
changeId: prompt-runtime-v2
runId: run-eval-001
stage: apply-change
evaluationId: eval-structured-artifact-v2-001
caseRunId: case-run-cross-domain-action-07

actor:
  actorId: automation:evaluation-host-local
  role: automation

operation:
  nodeId: validate_spec_domain
  status: failed
  reasonCode: DOMAIN_FORBIDDEN_ACTION
  startedAt: 2026-08-15T09:04:01Z
  endedAt: 2026-08-15T09:04:01Z
  durationMs: 18

binding:
  workflowVersion: 2.0.0
  policyVersion: evaluation-policy-1.0.0
  inputHash: dddd
  outputHash: eeee

traceId: trace-eval-001
spanId: span-domain-validation-07
parentSpanId: span-case-run-07
```

### 24.4 Evaluation Report

A Report contains at least:

```text
Manifest Reference and integrity status
Baseline／Candidate comparability
Dataset and cohort coverage
Hard Invariant results
Metric definitions and deltas
Failed／inconclusive case references
Evaluator identity and limitations
Regression classification
recommended next allowed action
```

The Report's `recommendation` is not a State Transition. The Orchestrator still loads the latest Current State, Gates, Approval, and Expected Revision.

## 25. Complete local example

### 25.1 Baseline

The team currently uses Structured Artifact Workflow v1. It runs the Domain Router, Skill Loader, Context Builder, Structured Spec generation, Schema Validation, Domain Validation, and Artifact Transformer in order. v1 passed Dataset 1.1.0 and became the Stable Baseline.

### 25.2 Candidate v2

Candidate v2 changes the Router and Retry Flow. Offline Evaluation reports:

```text
overall case pass rate: 92.5% → 97.5%
normal cohort latency p95: 840 ms → 710 ms
cross-domain cohort pass rate: 100% → 87.5%
hard invariant: failed
```

Overall pass rate and Latency improve, while `cross-domain-action-07` still executes `generate_artifact` after the Domain Validator returns Failure. This violates the Hard Invariant that a Domain Validation Failure blocks the Artifact Node.

The Evaluation Host emits `EVALUATION_INVARIANT_FAILED` and `EVALUATION_REGRESSION_DETECTED`, then writes the Case Result, Workflow Version, Node Trace, and Candidate Hash into the Report. After confirming Target Binding, the Reviewer returns `request_changes`. The Orchestrator keeps the v1 Accepted Pointer, and v2 remains a Candidate.

### 25.3 Candidate v3

The Implementer corrects `generate_artifact.when` and the Domain Failure route, producing Candidate v3. The Orchestrator reruns the Suite with the same Dataset, Rubric, Policy, and a compatible environment:

```text
all hard invariants: passed
cross-domain cohort: 8 / 8 passed
normal cohort latency p95: 735 ms
evaluation verdict: pass
```

v3 enters Shadow first, where Candidate Output does not control external Actions. After Shadow produces no Gate Bypass or new Domain Mismatch, a Human approves a controlled Canary. Canary Metrics and Traces remain bound to the v3 Release Unit. A change to its Prompt, Router, Workflow, or Policy invalidates the prior result.

### 25.4 Promotion

Reviewer `approve`, Evaluation `pass`, and a healthy Canary are Promotion Inputs. The Orchestrator checks Readiness, Human Approval, Current State Revision, and Candidate Binding before updating the Accepted Pointer with CAS. It also records the Mutation Receipt, Provenance Event, and Trace.

## 26. Minimum local adoption

### 26.1 Optional directory

```text
.agent-runtime/<change-id>/
├─ current-state.json
├─ runs/
│  └─ <run-id>/
│     ├─ artifacts/
│     ├─ evidence/
│     └─ evaluations/
│        └─ <evaluation-id>/
│           ├─ evaluation-manifest.json
│           ├─ case-results/
│           └─ evaluation-report.json
├─ telemetry/
│  ├─ events/
│  ├─ traces/
│  └─ metric-snapshots/
└─ host-journal/
   ├─ provenance/
   └─ mutation-receipts/
```

`evaluations/` and `telemetry/` are Git-ignored Runtime Data. Datasets, Rubrics, Metric Definitions, and Evaluation Policies that require long-term maintenance belong in Git or a governed Registry. An Agent reads only References authorized by its Handoff and cannot enumerate the entire Telemetry area or Host Journal.

### 26.2 Minimum Host interface

```text
createEvaluationManifest(subject, baseline, dataset, rubric, policy)
runEvaluationSuite(manifest)
appendTelemetryEvent(event)
publishEvaluationResult(result)
verifyEvaluationBinding(evaluationId)
compareBaselineAndCandidate(evaluationId)
evaluateReleaseGate(reportRef, stateRevision)
recordMetricSnapshot(scope, window)
applyTelemetryRetention(policy)
```

The Host uses Create-only Artifacts, Atomic Writes, and fixed References. The Evaluator interface does not expose `updateCurrentState()` or `acceptCandidate()`.

### 26.3 Minimum adoption order

1. Select one repeatable Workflow Vertical Slice.
2. Pin the Workflow, Prompt, Router, Model, Tool, and Policy Versions.
3. Build a Dataset from 5 to 10 Normal, Boundary, and Failure Cases.
4. Start with Schema, Domain, Binding, and Side Effect Hard Invariants.
5. Record Stage, Status, Reason Code, Duration, Attempt, and References for each Run.
6. Produce one Baseline and Candidate Evaluation Report.
7. Let the Reviewer read the Report and let the Orchestrator route under current Gates.
8. Retain full Failure Traces and sample successful Traces under Policy.
9. Use a known Gate Bypass or Binding Error Fixture to verify Alerting and fail-closed behavior.

The first version does not require a Dashboard, Distributed Trace Backend, or automated Drift Model. Command-line reports, JSON Artifacts, and local Traces are enough to verify data Binding and Decision boundaries.

## 27. Production upgrade criteria

A team SHOULD evaluate OpenTelemetry or an equivalent Telemetry standard, a centralized Metrics and Trace Backend, an Evaluation Registry, and Alert Routing when any of these conditions applies:

- Multiple Orchestrators, Workers, Repositories, or environments execute the Workflow.
- Diagnosis requires cross-service correlation across Client, Router, Model, Tool, Artifact, and State Mutation.
- The number of Evaluation Datasets, Prompts, Policies, and Release Units exceeds practical local-file indexing.
- Canary, Shadow, and multi-version Cohorts require near-real-time comparison.
- SLOs, Cost Budgets, On-call work, or Incident Response depend on reliable Alerts.
- Compliance requires data partitioning, Access Audit, Legal Hold, WORM, or regional restrictions.
- Telemetry volume requires Tail Sampling, Cardinality Control, and tiered Retention.
- CI or the Workflow Runtime must enforce Evaluation Gates automatically.

An upgraded system retains the same semantics:

```text
evaluation subject is immutable
baseline and candidate are explicitly bound
hard invariants fail closed
metrics do not replace evidence
trace does not replace audit
evaluator does not own state transition
accepted pointer changes through policy and CAS
```

For one machine, a Single Writer, low Run volume, and no compliance requirement, local Evaluation Artifacts and Traces can support a basic Regression Gate. Establish trustworthy data for one Workflow before adding platform components.

## 28. Acceptance checklist

### Subject and dataset

- [ ] The Evaluation Manifest pins the Subject, Baseline, Dataset, Rubric, Policy, and environment.
- [ ] Prompt, Router, Workflow, Model, Tool, and Adapter use exact Versions or Hashes.
- [ ] The Dataset has an Owner, Version, Risk Tags, Change History, and Data Classification.
- [ ] Cases include Normal, Boundary, Negative, Cross-domain, Injection, and Side Effect scenarios.
- [ ] A Dataset change creates a new Version and does not modify an old Result in place.
- [ ] An Incident fix adds a minimum reproducible Regression Case.

### Evaluation

- [ ] Completion Status and quality Verdict are retained separately.
- [ ] An Aggregate Score cannot offset a Hard Invariant Failure.
- [ ] An incomparable Candidate and Baseline return `not_comparable`.
- [ ] High-risk Cohorts have separate Thresholds and sample counts.
- [ ] Evaluator Identity, Model, Prompt, Rubric, and limitations are traceable.
- [ ] A Producer self-check does not become the Release Gate Verdict directly.
- [ ] The Evaluation Result binds the Candidate, Evidence, and Dataset Hash.

### Observability

- [ ] Metrics, Events, Traces, Evidence, and Audit Records use separate storage layers.
- [ ] Run, Stage, Invocation, Evaluation, and Mutation Identities remain distinct.
- [ ] Async Handoffs and Shadow Candidates use Span Links or explicit References.
- [ ] Latency and Cost retain Distributions or Percentiles rather than averages alone.
- [ ] High-cardinality Identity does not enter Metric Labels.
- [ ] Events use fixed Status and Reason Codes; Raw Errors remain in controlled Traces.
- [ ] A Signal Gap produces `inconclusive` and does not create a definitive conclusion.

### Gates and lifecycle

- [ ] An Evaluation Result cannot update Current State or the Accepted Pointer.
- [ ] Reviewer Verdict, Orchestrator Transition, and Human Approval remain separate.
- [ ] Changes to a Candidate, Rubric, Policy, or Evidence invalidate the old Gate.
- [ ] Shadow, Replay, and Simulation deny external Writes by default.
- [ ] Promotion uses the latest State Revision, Gates, Approval, and Candidate Binding.
- [ ] Current State contains no undefined Evaluation Phase or Gate.
- [ ] Handoff and Agent Result contain no fields undefined by their Schemas.

### Safety, retention, and operations

- [ ] Telemetry Export applies Redaction and Data Classification first.
- [ ] Traces exclude Secrets, Credentials, Approval Proof, complete sensitive Payloads, and private model reasoning.
- [ ] Sampling does not remove Audit Records for Incidents, Policy Denials, Human Overrides, or Integrity Failures.
- [ ] Metrics, Traces, Evaluation Reports, Evidence, and Audit each have a Retention class.
- [ ] Each Alert Rule defines a Window, Threshold, Minimum Count, Owner, Cooldown, and route.
- [ ] A Critical Alert can pause the related Release or high-risk Action.
- [ ] The Telemetry Backend does not expand an Agent's access to Repository, Runtime, or customer data.

## 29. References

- [Prompt Package runtime execution in a Workflow](../../../ai-agent-prompt-runtime-workflow-phase2-zh-TW.md)
- [Artifact-based Shared State and Structured Handoff reference architecture](./02-reference-architecture.md)
- [Repository Knowledge, Runtime State, Evidence, and Trace layers](./03-runtime-storage-and-retention.md)
- [Security, governance, and long-term maintainability](./05-security-and-maintainability.md)
- [Context Authority and Retrieval Policy](./06-context-authority-and-retrieval-policy.md)
- [Role Capability and Scope Control](./07-role-capability-and-scope-control.md)
- [Mutation, Concurrency, and Conflict Resolution](./08-mutation-concurrency-and-conflict-resolution.md)
- [Context Promotion and Rollback](./09-context-promotion-and-rollback.md)
- [Provenance Audit and Incident Replay](./10-provenance-audit-and-incident-replay.md)
- [Context Engineering Core](../../../context-engineering/docs/01-context-engineering-core.md)
- [Agent Platform Operations](../../../context-engineering/docs/03-agent-platform-operations.md)
- [Tool Governance, Evaluation, and Observability](../../../agent-design/tool-schema-routing/docs/03-tool-governance-and-evaluation.md)
- [Handoff Envelope Schema](../schemas/handoff-envelope.schema.json)
- [Current State Schema](../schemas/current-state.schema.json)
- [Agent Result Schema](../schemas/agent-result.schema.json)
- [Workflow Policy Template](../templates/workflow-policy.template.yaml)
