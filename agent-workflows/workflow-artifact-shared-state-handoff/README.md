# Artifact-based shared state and structured handoff

[English](./README.md) | [繁體中文](./README-zh-TW.md)

> Moving from manual context transfer to verifiable, persistent, and orchestrated multi-agent coding workflows.

## Workflow scope

This workflow covers a common problem in multi-agent coding setups:

When planning, implementation, review, and human approval are handled by different AI agents or CLI tools, the previous role's conclusion often cannot become reliable input for the next role. The user ends up copying requirements, change summaries, diffs, validation results, and review findings between stages, then explaining the same background again.

That process already has role separation, but the human still acts as:

- Context Router
- Clipboard Transport
- Shared State Store
- Workflow Engine
- Integration Arbiter
- Git Gatekeeper

The core improvement is to add two complementary mechanisms:

1. **Artifact-based Shared State**: preserve the working facts shared by all roles.
2. **Structured Handoff**: define how work moves from one role to the next.

In short:

```text
Artifact-based Shared State
= what the workflow currently trusts

Structured Handoff
= who should act next, based on which facts, to produce which result
```

## Why this deserves its own design

Prompts, skills, AGENTS.md, and rule assets reduce repeated instructions. They mainly answer "how should the agent work?" They do not automatically answer:

- Which phase is the workflow in?
- Which implementation result is the latest accepted one?
- Which tests were actually run?
- Are there unresolved review blockers?
- Which inputs must the next role read?
- After failure, should the workflow return to implementation, specification arbitration, or manual handling?
- Is archive, commit, push, or deploy allowed?

As long as those facts live in human memory or the clipboard, the workflow is hard to automate, hard to resume, and hard to audit.

## Core architecture

```text
┌──────────────────────────────────────────────┐
│ Durable Knowledge                            │
│ Requirements / Design / Tasks / Decisions    │
└──────────────────────┬───────────────────────┘
                       │ references
                       ▼
┌──────────────────────────────────────────────┐
│ Artifact-based Shared State                  │
│ Current State / Results / Evidence Refs      │
└──────────────────────┬───────────────────────┘
                       │ structured handoff
                       ▼
┌──────────────────────────────────────────────┐
│ Coordinator -> Implementer -> Reviewer       │
│      ↑                 │          │           │
│      └── arbitration ──┴── fixes ─┘           │
└──────────────────────┬───────────────────────┘
                       │ protected gate
                       ▼
┌──────────────────────────────────────────────┐
│ Human Approval                               │
│ Commit / Push / Deploy / Destructive Action  │
└──────────────────────────────────────────────┘
```

From an architecture perspective:

- Shared State is the **Data Plane**.
- Structured Handoff is the **Control Plane**.
- The Orchestrator is the future **Process Manager** that applies state transitions.
- Human-in-the-loop approval is the **Approval Gate** for irreversible or high-risk actions.

## When to use it

Use this workflow when:

- Planner, Coder, and Reviewer use different models or CLI tools.
- The project already has SDD, OpenSpec, ADRs, issues, or other specification assets.
- The Reviewer should stay read-only and cannot change code.
- Commit, push, and deploy still require human approval.
- The team wants to introduce Node.js, Python, or a workflow engine gradually.
- The goal is to reduce repeated prompts and context transfer, not to remove humans from the loop.

You may not need it when:

- A single agent can complete a small task in one short session.
- There is no role split and no review gate.
- The task does not need recovery, auditability, retries, or version identity.

## What this workflow is not

This workflow is not:

- A vendor-specific design for one model provider.
- A CLI tutorial.
- A required agent framework.
- A documentation governance system that forces all state into Markdown.
- An autonomous system where the LLM decides every workflow and Git operation.
- A claim that every large company uses the same implementation.

Public frameworks and protocols share similar ideas around shared state, artifacts, handoff, persistence, and human-in-the-loop control. The concrete data model, permissions, and storage choices still depend on team size and risk.

## Recommended reading order

1. [`docs/01-problem-and-maturity-model.md`](docs/01-problem-and-maturity-model.md)  
   Start from human-orchestrated workflows and identify the real bottleneck.
2. [`docs/02-reference-architecture.md`](docs/02-reference-architecture.md)  
   Define Shared State, Artifact, Handoff, State Machine, and role boundaries.
3. [`docs/03-runtime-storage-and-retention.md`](docs/03-runtime-storage-and-retention.md)  
   Separate repo documents, runtime state, evidence, and trace data.
4. [`docs/04-adoption-roadmap.md`](docs/04-adoption-roadmap.md)  
   Move from manual handoff to a Node Orchestrator in phases.
5. [`docs/05-security-and-maintainability.md`](docs/05-security-and-maintainability.md)  
   Handle path safety, permissions, schema governance, versioning, concurrency, and long-term maintenance.
6. [`patterns/README.md`](patterns/README.md)  
   Map the design to established patterns such as Blackboard, Repository, State Machine, and Process Manager.

## First batch scope

This first batch focuses on knowledge and architecture. It does not provide formal schemas, prompts, or an executable orchestrator yet. Future engineering assets can use this layout:

```text
artifact-shared-state-handoff/
├─ schemas/
│  ├─ agent-result.schema.json
│  ├─ handoff-envelope.schema.json
│  └─ current-state.schema.json
├─ templates/
├─ prompts/
└─ examples/
```

## Quick check: is a Node.js script already a multi-agent orchestrator?

A script that only runs these steps in order is usually still an automated pipeline:

```text
run planner
run coder
run reviewer
```

It starts to look like a workflow orchestrator when it can:

- Load and validate Current State.
- Validate preconditions and agent output schemas.
- Persist artifacts and evidence references.
- Advance or reject state transitions based on results.
- Create fix loops from review findings.
- Escalate design conflicts to the Coordinator.
- Pause and resume at human approval gates.
- Support retry, idempotency, and recovery.

## Minimum principle

The workflow can be summarized as:

> Turn the human process into stable contracts first, then let code orchestrate those contracts.

Do not start with a database, queue, full trace platform, or fully automated Git operations. The first useful version only needs:

- A verifiable Current State.
- A shared Agent Result Envelope.
- A Structured Handoff Envelope.
- A clear state transition table.
- A read-only Reviewer boundary.
- A human Commit Gate.

## References

- LangGraph: State as shared memory, checkpointers, persistence, HITL, and recovery.  
  <https://docs.langchain.com/oss/javascript/langgraph/thinking-in-langgraph>  
  <https://docs.langchain.com/oss/javascript/langgraph/persistence>
- OpenAI Agents SDK: Handoff, code-led and LLM-led orchestration, and human-in-the-loop.  
  <https://openai.github.io/openai-agents-js/guides/handoffs/>  
  <https://openai.github.io/openai-agents-js/guides/multi-agent/>  
  <https://openai.github.io/openai-agents-js/guides/human-in-the-loop/>
- Agent2Agent Protocol: Task, Message, Artifact, and Artifact Lifecycle.  
  <https://a2a-protocol.org/latest/topics/key-concepts/>  
  <https://a2a-protocol.org/latest/topics/life-of-a-task/>
- Enterprise Integration Patterns: Process Manager, Message, Pipes and Filters, Claim Check.  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html>
- Azure Architecture Center: Event Sourcing, Materialized View, Claim Check.  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing>  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view>  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check>

The workflow is framework-neutral, model-neutral, and specification-tool-neutral. Validate the design against the actual CLI version, permission model, data sensitivity, and team process before adoption.
