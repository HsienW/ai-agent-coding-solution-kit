# 01 | Problem definition and multi-agent coding workflow maturity model

[English](./01-problem-and-maturity-model.md) | [繁體中文](./01-problem-and-maturity-model-zh-TW.md)

## 1. Common starting point: multiple agents, with a human as the console

A common AI-assisted development flow looks like this:

```text
Coordinator / Planner
        │
        │ manually copy requirements and conclusions
        ▼
Implementer
        │
        │ manually copy change summary, diff, and validation results
        ▼
Reviewer
        │
        │ manually copy findings and verdict
        ▼
Coordinator / Human
        │
        ▼
Commit / Archive / Deploy
```

This pattern already has the basic ingredients of a multi-agent workflow:

- Specialized roles.
- Separation between planning, implementation, and review.
- A single writer.
- A review gate.
- Human-in-the-loop control.
- Manual control over irreversible operations.

More accurate names include:

- Human-orchestrated Multi-Agent Workflow
- Manual Multi-Agent Pipeline
- CLI Agent Pipeline with HITL Gates
- Human-orchestrated multi-agent collaboration

The problem is that the user still performs all cross-role coordination. From a system perspective, the human is acting as five missing programmable components:

| Human responsibility | System capability after formalization |
|---|---|
| Remember current progress | Current State |
| Copy the previous stage's conclusion | Artifact Reference |
| Tell the next agent what to do | Structured Handoff |
| Decide which stage to return to | State Machine / Router |
| Execute commit, push, deploy | HITL Approval Gate |

The first core judgment is:

> The bottleneck is usually the lack of shared, verifiable, persistent working state.

## 2. Why skills and short prompts are not enough

Skills, role documents, and prompt templates are important. They preserve:

- Role responsibilities.
- Coding standards.
- Review format.
- Execution steps.
- Safety limits.
- Project conventions.

These assets describe static operating rules. They do not automatically preserve the dynamic facts of the current task.

For example, a Reviewer skill can define Blocker, Major, and Minor findings, but it does not automatically know:

- Which files the Implementer changed in this run.
- Which requirement has been completed.
- Which test failed.
- Which specification version the current diff belongs to.
- Whether the previous review findings were fixed.
- Whether this round should enter a readiness check.

The distinction is:

```text
Skill / Rules
= reusable long-term working methods

Shared State / Artifact
= working facts that currently hold for this run
```

With only skills, agents still need the user to provide the current task context again. With only state, agents may not know how to use it correctly. The two have to work together.

## 3. Where token waste really happens

Long prompts are only one source of token waste. Structural repetition is the larger problem.

### 3.1 Repeating background

Each agent rereads:

- Why the change is needed.
- How it is expected to be done.
- What must not be changed.
- Which risks have already been accepted.

### 3.2 Re-translating results

The Implementer writes a natural-language summary. The user rewrites it into a Reviewer prompt. The Reviewer's output is then reorganized for the Coordinator.

Each translation can:

- Omit details.
- Introduce interpretation drift.
- Use the wrong latest version.
- Lose failure evidence.

### 3.3 Reloading irrelevant history

To avoid missing context, users tend to paste the full chat history for the next agent. The agent then digests a large amount of expired information instead of reading only the latest accepted state.

### 3.4 Losing precise cacheability

If every round uses a different natural-language summary, reusable content is hard to identify. Structured artifacts and references make content hashing, version identity, and partial loading easier.

A better optimization target is:

> Each role should read only the minimum sufficient facts needed for the current task.

## 4. Risks in manual retelling

### 4.1 Stale Context

The user may paste findings from a previous round rather than the latest round.

### 4.2 Missing Evidence

The user may paste "tests passed" without the command, exit code, scope, or skipped items.

### 4.3 Role Leakage

The Reviewer starts changing code to fill missing information. The Implementer changes requirements to move the workflow forward.

### 4.4 Gate Bypass

After the Reviewer returns PASS, the flow goes straight to commit without checking whether tests, tasks, specs, and diff still match.

### 4.5 Non-resumable Session

After a CLI interruption, the next session cannot recover from a reliable state and must reread chat history.

### 4.6 No Audit Boundary

It becomes unclear which role produced which conclusion, from which input, under which state.

## 5. Multi-agent coding workflow maturity model

Maturity describes an evolution path: from manual collaboration toward a persistent, programmable, and governable system.

## Level 0: Single-agent session

```text
User -> Agent -> Output
```

Characteristics:

- One agent plans, implements, and self-checks.
- No role separation.
- No cross-session state.

Best for:

- Small, low-risk, one-off tasks.

Main limitations:

- Self-review bias.
- Long tasks lose context easily.

## Level 1: Human-orchestrated multi-agent workflow

```text
Planner -> Human -> Implementer -> Human -> Reviewer
```

Characteristics:

- Roles are separated.
- The human handles all handoff.
- Git and high-risk operations remain under human control.

Value:

- Different models and roles can complement each other.
- Easy to set up without adding runtime infrastructure.

Limits:

- The human is the bottleneck.
- Context does not move directly between roles.
- Token and operation cost grow with each workflow round.

## Level 2: Artifact-driven workflow

```text
Agent
  ↕
Current State + Artifact References
  ↕
Agent
```

New capabilities:

- Agent Result Schema.
- Handoff Envelope.
- Current State Schema.
- Explicit State Machine.
- Artifact Reference.
- Review Evidence.
- HITL Gate.

At this stage, a human may still start each agent manually. The difference is that the human no longer needs to move full content around. The handoff can be reduced to:

```text
changeId
runId
currentStateRef
handoffId
```

## Level 3: Durable workflow

New capabilities:

- Checkpoint.
- Pause / Resume.
- Retry.
- Idempotency.
- Artifact Versioning.
- Runtime / Trace separation.
- Failure recovery.

LangGraph checkpointers are one public example: they save graph state snapshots for HITL, recovery, and time travel.

## Level 4: Programmatic orchestration

Add a Node.js, Python, or workflow engine orchestrator:

```text
Load State
-> Validate Preconditions
-> Invoke Agent
-> Capture Result
-> Validate Schema
-> Persist Artifact
-> Apply Transition
-> Pause at HITL Gate
```

At this point, code controls the flow and the LLM handles professional judgment inside each node.

## Level 5: Production-grade agent platform

Add:

- Queue and work leases.
- Concurrency control.
- Durable Store.
- Trace, Metrics, Cost Governance.
- Secret Management.
- Policy Engine.
- Eval and regression tests.
- Incident Recovery.
- Artifact Retention.
- Release / Rollback.

The goal at this level is safe operation, not simply running agents automatically.

## 6. When does a Node.js script count as an orchestrator?

### When it is only a pipeline

```js
await runPlanner();
await runImplementer();
await runReviewer();
```

This code calls three agents automatically, but it lacks:

- State validation.
- Output contracts.
- Branching.
- Error routing.
- Human approval.
- Recovery.

It is still an automated pipeline, not a full Multi-Agent Workflow Runtime.

### When it approaches an orchestrator

The script starts to look like an orchestrator when it handles:

```text
review.verdict == REQUEST_CHANGES
    -> return to Implementer

review.verdict == INCOMPLETE
    -> add evidence or send to manual handling

review.designConflict == true
    -> Coordinator Arbitration

allGatesPassed == true
    -> Pause for Human Commit Approval
```

It also ensures:

- The same input does not create duplicate side effects.
- Invalid schema does not advance state.
- Agent failure can resume from a checkpoint.
- Irreversible operations always stay behind gates.

## 7. Recommended success metrics

Do not evaluate the system only by "how many rounds ran automatically." More useful metrics include:

### Handoff efficiency

- Number of words the human must paste per handoff.
- Number of operations needed to start the next role.
- Ratio of repeated background tokens.

### Correctness

- Errors caused by stale artifacts.
- Schema validation failure rate.
- Ratio of findings lost during handoff.
- Illegal state transition count.

### Recovery

- Time to recover after CLI interruption.
- Whether retry duplicates file writes or side effects.
- Whether the latest accepted artifact can be identified.

### Human control

- Whether commit, push, and deploy have clear approval records.
- Whether humans handle decisions instead of copy-paste.

## 8. Anti-metrics

The following do not imply maturity:

- Many agents.
- Long prompts.
- Large amounts of Markdown every round.
- Full automation of every operation.
- Reviewers that can modify code directly.
- LLMs that decide commit or deploy by themselves.
- Saving all conversations and calling that memory.

What matters is:

- Clear state.
- Minimal and traceable input.
- Verifiable output.
- Roles that preserve permission boundaries.
- Recoverable failure.

## 9. Minimum upgrade signal

When any three of the following are true, it is worth moving from Level 1 to Level 2:

- Each task passes through at least three roles.
- Review usually has more than one fix loop.
- Users frequently paste diffs, validation results, and findings.
- CLI sessions are hard to recover after interruption.
- Multiple changes run in parallel and context gets mixed.
- The team is preparing an automatic orchestrator.
- The team needs to prove which evidence supported an approval.

## 10. Summary

Human orchestration is a reasonable first stage. It validates whether role separation and the workflow are valuable.

Once the process stabilizes, the next step is not full automation. The next step is to make the human's implicit work explicit:

```text
human memory
-> Current State

human clipboard
-> Artifact Reference

human handoff explanation
-> Structured Handoff

human routing judgment
-> State Machine

human high-risk approval
-> HITL Gate
```

That is the dividing line between "using multiple agents" and "building a multi-agent workflow."

## References

- LangGraph: Thinking in LangGraph  
  <https://docs.langchain.com/oss/javascript/langgraph/thinking-in-langgraph>
- LangGraph: Persistence  
  <https://docs.langchain.com/oss/javascript/langgraph/persistence>
- OpenAI Agents SDK: Agent Orchestration  
  <https://openai.github.io/openai-agents-js/guides/multi-agent/>
- OpenAI Agents SDK: Human-in-the-loop  
  <https://openai.github.io/openai-agents-js/guides/human-in-the-loop/>
