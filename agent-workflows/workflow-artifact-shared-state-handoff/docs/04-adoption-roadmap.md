# 04 | Phased adoption roadmap: from manual handoff to Node Orchestrator

[English](./04-adoption-roadmap.md) | [繁體中文](./04-adoption-roadmap-zh-TW.md)

## 1. Adoption principles

The most stable adoption order is:

```text
observe the manual process first
-> stabilize roles and gates
-> define data contracts
-> separate storage lifecycles
-> automate orchestration last
```

Reason:

> Automation amplifies the existing process. If roles, inputs, outputs, and failure routes are unclear, an orchestrator only runs the confusion faster.

This chapter provides five adoption phases. Each phase has a clear scope, completion criteria, and things not to do early.

## 2. Phase 0: establish the manual process baseline

### Goal

Confirm that role separation is actually valuable, and observe real handoff content.

### Suggested roles

- Coordinator / Planner
- Implementer
- Reviewer
- Human Approver

### Record

For each manual handoff, record what was moved:

- Spec summary.
- Scope.
- Changed files.
- Diff.
- Validation.
- Findings.
- Verdict.
- Next step.

### Completion criteria

- Role responsibilities are stable.
- Reviewer read-only status is decided.
- HITL operations are identified.
- The flow has succeeded several times.
- The most repeated copied data is visible.

### Do not

- Do not introduce queues yet.
- Do not create a dozen agents.
- Do not persist every chat permanently.
- Do not allow multiple agents to write the same working tree at the same time.

## 3. Phase 1: Contract Layer

### Goal

Let different agents work with the same data language, even when humans still start them manually.

### Changes

1. Agent Result Schema.
2. Handoff Envelope Schema.
3. Current State Schema.
4. State Transition Table.
5. Artifact Reference Convention.
6. Reviewer Read-only Contract.
7. HITL Git Gate.
8. Execution Summary Template.
9. Git-ignored Runtime Directory.

### Minimum directory

```text
.agent-runtime/<change-id>/
├─ current-state.json
├─ artifacts/
└─ evidence/
```

### The first version can install nothing

Start with:

- JSON Schema files.
- Native CLI structured output.
- Manual checks.
- Simple JSON parsing.

Validate the flow first.

Introduce Ajv, Zod, or another runtime validator at the Orchestrator phase.

### Completion criteria

- The next agent does not need a long pasted summary.
- Change ID, Current State, and Artifact Refs are enough to start.
- Invalid schema prevents handoff.
- Reviewer does not need write permission.
- Commit / push remain under human control.
- Runtime artifacts do not enter Git.

### Dry run

Choose a medium-sized, low-risk change:

```text
Coordinator
-> coordinator_result
-> Implementer
-> implementation_result + evidence
-> Reviewer
-> review_result
-> Coordinator readiness
-> Human commit
```

Measure whether pasted human content drops clearly.

## 4. Phase 2: Storage and Lifecycle Separation

### Goal

Make State, Evidence, Trace, and Durable Knowledge boundaries clear.

### Changes

- Repo / Runtime / Trace separation.
- Latest Accepted Artifact concept.
- Artifact Version and Checksum.
- Retention Policy.
- Secret Redaction.
- Cross-run Isolation.
- Runtime Cleanup.
- Failure Recovery Strategy.

### Completion criteria

- Git does not grow documents without bound during agent retries.
- Current State does not contain large diffs or full trace.
- Artifacts are traceable by ID, URI, version, and checksum.
- Trace can be cleaned without damaging Durable Knowledge.
- The system distinguishes latest produced from latest accepted.

### Optional upgrades

- CI Artifact Store.
- SQLite.
- Object Storage.
- Checkpointer.

Choose based on the use case. Looking complete is not the goal.

## 5. Phase 3: Node.js Orchestrator

### Goal

Mechanize the manual state flow after it has stabilized.

### Suggested workflow structure

```text
tools/agent-orchestrator/
├─ src/
│  ├─ state/
│  ├─ contracts/
│  ├─ adapters/
│  ├─ routing/
│  ├─ approvals/
│  └─ cli/
├─ tests/
└─ package.json
```

### Core responsibilities

```text
1. Load Current State
2. Validate State Schema
3. Check Transition Preconditions
4. Build Handoff
5. Invoke Agent Adapter
6. Capture Structured Result
7. Validate Result Schema
8. Persist Artifact and Evidence
9. Apply Transition
10. Pause at Human Gate
11. Resume after Approval
```

### Adapter abstraction

Do not tie the Orchestrator to one CLI:

```ts
interface AgentAdapter {
  invoke(input: AgentInvocation): Promise<AgentInvocationResult>;
}
```

Each tool handles its own:

- CLI arguments.
- Structured Output.
- stdout / stderr.
- Exit Code.
- Timeout.
- Sandbox / Permission.

### Validator

Node.js options include:

- Ajv: closer to JSON Schema standards and cross-language contracts.
- Zod: closer to TypeScript types and in-process use.

If the public contract must be shared across languages, CLIs, and platforms, JSON Schema is usually a better canonical contract. TypeScript types can be generated from schema or built inside adapters.

### State advancement rules

The Orchestrator should not blindly trust an agent's `nextHandoff`. It should:

1. Verify whether the agent can propose the transition.
2. Check whether gates are satisfied.
3. Validate artifact references.
4. Update state.
5. Record the transition result.

### Completion criteria

- The normal Plan -> Implement -> Review path can run automatically.
- REQUEST_CHANGES loops are handled.
- INCOMPLETE is handled.
- Design conflicts escalate to Coordinator.
- The flow pauses at Commit Gate.
- Re-running does not duplicate unsafe side effects.

## 6. Phase 4: Production Hardening

### Goal

Support multiple users, multiple runs, and long-running execution safely.

### Capabilities

- Durable Store.
- Queue.
- Lease / Lock.
- Concurrency Control.
- Timeout / Retry Policy.
- Dead-letter / Manual Recovery.
- Trace, Metrics, Alert.
- Cost Budget.
- Policy Engine.
- Evaluation.
- Release / Rollback.

### When is this needed?

When requirements include:

- Concurrent changes.
- Multiple people sharing one agent platform.
- Agent execution beyond a single process lifetime.
- Strict audit.
- SLA.
- Governance over token use, latency, and success rate.

## 7. Suggested implementation slices

Phase 1 can be split into commits performed by a human:

```text
Commit 1
docs: define artifact shared state and handoff contracts

Commit 2
feat: add agent result, handoff, and current state schemas

Commit 3
refactor: route agent stages through artifact references

Commit 4
chore: define local runtime boundary and gitignore rules

Commit 5
test: validate artifact handoff with a dry run
```

Phase 3 can be split as:

```text
Commit 1
feat(orchestrator): add contract validation and state repository

Commit 2
feat(orchestrator): add CLI agent adapters

Commit 3
feat(orchestrator): add workflow routing and retry loop

Commit 4
feat(orchestrator): add human approval pause and resume

Commit 5
test(orchestrator): cover failure and recovery paths
```

## 8. Rollout strategy

Do not switch all tasks at once.

### Shadow Mode

The Orchestrator only calculates the next step. It does not call agents. Compare it with human judgment.

### Assisted Mode

The Orchestrator creates handoff and commands. The user confirms before execution.

### Semi-automatic Mode

Run reversible stages automatically. Pause at review, commit, push, and deploy.

### Controlled Automatic Mode

Low-risk tasks move automatically. High-risk operations remain HITL.

## 9. Rollback strategy

If automation fails, the system should fall back to manual mode:

- Current State is readable by humans.
- Artifacts are standard JSON / files, not hidden inside a framework.
- Handoff can become a readable summary.
- CLI adapters can run independently.
- Durable Knowledge remains understandable without Runtime.

This is the value of open contracts: workflow data is not locked inside a closed framework state.

## 10. Quantifying adoption effect

Compare before and after:

| Metric | Target direction |
|---|---|
| Words pasted by humans per handoff | Lower |
| Manual operations per change | Lower |
| Rework caused by stale context | Lower |
| Schema errors caught early | Higher |
| Recovery time after CLI interruption | Lower |
| Reviewer evidence completeness | Higher |
| Human review rate for irreversible operations | 100% or policy-based |

Do not treat "no human operation" as the only KPI. For coding workflows, reliability and traceability often matter more than autonomy.

## 11. Common premature overdesign

### Full Event Sourcing from the start

Without a stable event model, this only adds migration and replay cost.

### Distributed queue from the start

Single-person, single-working-tree flows usually do not need it.

### Letting the LLM generate all transitions freely

This easily creates unknown states, skipped gates, and non-reproducible routing.

### Reviewer directly edits code

This breaks role independence and review evidence.

### Committing all runtime data to Git

This creates document sprawl and sensitive data risk.

### Binding the Orchestrator to one CLI

Tool upgrades or replacement would force a rewrite of the whole flow.

## 12. Adoption decision tree

```text
Is there already a stable multi-role process?
├─ No -> do Phase 0 first
└─ Yes
   ↓
Does the flow still depend on manually pasted summaries / diffs / findings?
├─ Yes -> do Phase 1
└─ No
   ↓
Are document sprawl, cross-run needs, recovery, or retention showing up?
├─ Yes -> do Phase 2
└─ No
   ↓
Are workflow contracts stable and repeatedly executed?
├─ Yes -> do Phase 3
└─ No -> keep manual dry runs
   ↓
Are there multi-user, multi-machine, SLA, concurrency, or audit needs?
├─ Yes -> do Phase 4
└─ No -> stay lightweight
```

## References

- OpenAI Agents SDK: Agent Orchestration  
  <https://openai.github.io/openai-agents-js/guides/multi-agent/>
- OpenAI Agents SDK: Human-in-the-loop  
  <https://openai.github.io/openai-agents-js/guides/human-in-the-loop/>
- LangGraph: Overview / Persistence  
  <https://docs.langchain.com/oss/javascript/langgraph/overview>  
  <https://docs.langchain.com/oss/javascript/langgraph/persistence>
- Enterprise Integration Patterns: Process Manager  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html>
