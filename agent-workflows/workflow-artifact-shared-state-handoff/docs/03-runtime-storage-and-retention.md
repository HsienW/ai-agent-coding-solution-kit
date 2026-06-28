# 03 | Separating repo documents, runtime state, evidence, and trace

[English](./03-runtime-storage-and-retention.md) | [繁體中文](./03-runtime-storage-and-retention-zh-TW.md)

## 1. Main concern: will Shared State create more documents than code?

Yes, if every agent output is treated as permanent Markdown and committed to Git.

The wrong shape often looks like this:

```text
execution/
├─ planner-001.md
├─ coder-001.md
├─ reviewer-001.md
├─ coder-002.md
├─ reviewer-002.md
├─ coordinator-001.md
├─ test-output-001.md
└─ trace-001.md
```

Long-term consequences:

- Users cannot tell which result is latest.
- Agents read all history to be safe.
- Git diff gets polluted by runtime noise.
- Short-lived failures become permanent maintenance cost.
- Specs, state, evidence, and debug logs are mixed in one layer.
- Document count grows without improving understanding.

The problem is the missing lifecycle separation, not Shared State itself.

## 2. Four-layer separation model

Split information into four layers.

## 2.1 Durable Knowledge: long-term engineering knowledge

Usually stored in the Git repository.

Typical content:

- Requirement / Proposal.
- Design.
- Acceptance Criteria.
- ADR / Decision.
- Task Plan.
- Final Execution Summary.
- Reusable schema, template, and prompt assets.

Rule of thumb:

> If maintainers still need to know it six months later, it belongs in Durable Knowledge.

## 2.2 Runtime State: current operational snapshot

Typical content:

- Current phase.
- Current owner.
- Gate status.
- Latest artifact refs.
- Attempt.
- Blockers.
- Next action.

The first version can store it at:

```text
.agent-runtime/<change-id>/current-state.json
```

and add that path to `.gitignore`.

## 2.3 Execution Evidence: verifiable construction evidence

Typical content:

- Changed files.
- Git Diff / Diff Stat / Diff Check.
- Test / Lint / Build commands and exit codes.
- Validation Summary.
- Review Result.
- Skipped items.

Evidence is different from Trace. Evidence is the minimum proof needed for an engineering decision. Trace is complete execution observation.

## 2.4 Trace: debugging and observability history

Typical content:

- Model request / response metadata.
- Token usage.
- Latency.
- Tool invocation.
- Retry.
- stdout / stderr.
- Full event sequence.

Trace usually belongs in:

- A dedicated trace platform.
- CI artifact store.
- Object storage.
- Database.
- Local Git-ignored directory.

It should not go into the project documentation directory.

## 3. Suggested data matrix

| Data | Normal reader | Commit to Git | Suggested retention | Main purpose |
|---|---|---:|---|---|
| Requirements / Design | Humans and agents | Yes | Long-term | Product and engineering contract |
| Current State | Orchestrator / Agent | No | During the run | Flow progression |
| Latest Agent Results | Next agent | No | Short retention after change completion | Handoff |
| Diff / Validation Evidence | Reviewer / Coordinator | No | Based on audit needs | Validation and review |
| Full Trace | Platform / operations | No | Based on cost and compliance | Debugging, observability, cost analysis |
| Execution Summary | Future maintainers | Yes | Long-term | Final fact consolidation |

## 4. Minimum local directory

The first stage does not need a database. Start with:

```text
.agent-runtime/
└─ feature-example/
   ├─ current-state.json
   ├─ artifacts/
   │  ├─ coordinator-result.json
   │  ├─ implementation-result.json
   │  ├─ review-result.json
   │  └─ handoff.json
   └─ evidence/
      ├─ changed-files.txt
      ├─ git-diff.patch
      ├─ git-diff-stat.txt
      ├─ git-diff-check.txt
      └─ validation-summary.json
```

A latest-only strategy is enough for the first version:

- `implementation-result.json` is overwritten by the next round.
- `review-result.json` stores the current latest review.
- Current State points to the latest accepted version.

If history is needed, create subdirectories by run first. Do not start with full Event Sourcing.

## 5. Why agents should normally read snapshots

If every run replays full history:

```text
all planner messages
+ all implementation attempts
+ all reviews
+ all test logs
```

it creates:

- Rising token cost.
- Stale information that interferes with judgment.
- Slower recovery.
- Ambiguity about which result was accepted.

A better read pattern is:

```text
Current State Snapshot
+ Required Artifact References
+ Directly Relevant Durable Knowledge
```

Only inspect full history when:

- Design arbitration is needed.
- The source of an error is being investigated.
- Security audit requires it.
- Different attempts must be compared.
- State must be reconstructed.

Azure's Materialized View pattern reflects the same idea: source data is not always the right shape for every query. Current State can be treated as the workflow's operational view.

## 6. Event Sourcing: valuable, but not too early

A full event log might look like:

```text
ChangeCreated
PlanApproved
ImplementationStarted
ImplementationCompleted
ReviewRequested
ChangesRequested
ImplementationRevised
ReviewApproved
HumanCommitApproved
```

Benefits of Event Sourcing:

- Complete audit history.
- State reconstruction.
- Easier workflow history analysis.
- Good fit for systems that need high reliability and compensating actions.

Costs are also high:

- Event Schema Versioning.
- Idempotency.
- Event Ordering.
- Replay.
- Snapshot.
- Migration.
- Compensating Action.

For most individuals or small teams, the first version should use:

```text
Current State
+ Latest Artifacts
+ a small number of important events / human approval records
```

Upgrade when cross-machine recovery, concurrent runs, or audit requirements become real.

## 7. Artifact Version and the latest accepted version

"Latest produced" is not always "latest accepted."

Example:

```text
implementation v1 -> review approved
implementation v2 -> validation failed
```

The system should not replace v1 automatically just because v2 is newer.

Separate:

- `createdVersion`
- `candidateVersion`
- `acceptedVersion`
- `supersedes`
- `status`

The A2A Task Lifecycle also notes that the Client / Orchestrator is better suited than the serving agent to manage artifact revisions and select the latest accepted version. This responsibility should not belong entirely to the artifact producer.

## 8. Reference instead of Embed

Do not put a full 5 MB diff into `current-state.json`:

```json
{
  "diff": "...millions of characters..."
}
```

Store a reference:

```json
{
  "diffRef": {
    "artifactId": "diff-002",
    "uri": "runtime://evidence/git-diff.patch",
    "mediaType": "text/x-diff",
    "sha256": "...",
    "status": "accepted"
  }
}
```

Benefits:

- State stays small.
- Large artifacts can be stored and cleaned independently.
- Content hash validation is easier.
- Handoff does not need to copy full text.
- Different artifacts can have different permission controls.

This is close to Claim Check: messages pass controlled references, while content stays in a suitable store.

## 9. Storage evolution from local to production

## Stage A: local Git-ignored files

Best for:

- One person.
- One working tree.
- A small number of changes at a time.

Benefits:

- No external dependency.
- Easy to inspect.
- Good for validating schema and flow.

Limits:

- Not suitable for multiple machines.
- Weak concurrency and locking.
- Hard to recover after local disk failure.

## Stage B: CI artifact store

Best for:

- Agents running in CI or remote runners.
- Evidence that must be bound to a run.

Can store:

- Diff.
- Test reports.
- Structured results.
- Trace bundle.

## Stage C: Database + Object Storage

Best for:

- Multiple users.
- Concurrent runs.
- Query, audit, and retention policies.

Common split:

```text
Database
= State, Metadata, Indexes, Approval

Object Storage
= Diff, Log, Report, Large Artifact
```

## Stage D: Workflow Runtime / Checkpointer

Use LangGraph checkpointers, Temporal, Step Functions, or another durable workflow mechanism to provide:

- Checkpoint.
- Retry.
- Resume.
- Timeout.
- Workflow History.
- Human Approval.

Choose a runtime after contracts have stabilized. Do not let the framework define an immature business process.

## 10. Retention Policy

Artifacts do not all need permanent retention.

Suggested policies:

| Type | Suggested policy |
|---|---|
| Current State | Keep briefly after run completion, or delete after Final Summary |
| Latest Agent Results | Keep for days to weeks after change completion |
| Validation Evidence | Keep based on review / audit needs |
| Full Trace | Decide based on cost, privacy, and debugging value |
| Human Approval | Long-term or legally required retention |
| Execution Summary | Commit to Git for long-term retention |

Retention must consider:

- Storage cost.
- Privacy and secret exposure.
- Issue investigation window.
- Compliance requirements.
- Whether a Git commit or CI run already provides traceability.

## 11. Promotion: which runtime knowledge should enter Git?

When a change completes, create a concise Execution Summary from runtime artifacts:

- What was actually completed.
- Differences from the original design.
- Main modified files.
- Validation results.
- Accepted risks.
- Incomplete items.
- Important decisions.
- Commit / Release Reference.

Do not promote:

- Full chat.
- Private reasoning.
- Original text of every failed round.
- All stdout.
- Token details.
- Temporary summaries superseded by later versions.

This can be called:

> Runtime-to-Durable Knowledge Promotion

## 12. Maintainability checks

A healthy Shared State design should satisfy:

- Long-term documents in Git do not grow without bound as agents retry.
- Agents do not need full history for normal work.
- Every artifact can identify its producer, run, and scope.
- Current State can identify the latest accepted artifact.
- Trace can be deleted independently without breaking project knowledge.
- If Runtime Store is damaged, the workflow can be rebuilt from Durable Knowledge and Git state.

## References

- LangGraph Persistence / Checkpointers  
  <https://docs.langchain.com/oss/javascript/langgraph/persistence>  
  <https://docs.langchain.com/oss/javascript/langgraph/checkpointers>
- A2A: Life of a Task  
  <https://a2a-protocol.org/latest/topics/life-of-a-task/>
- Azure: Event Sourcing Pattern  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing>
- Azure: Materialized View Pattern  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view>
- Azure: Claim Check Pattern  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check>
