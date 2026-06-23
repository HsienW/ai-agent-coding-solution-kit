# OpenSpec Agent Workflow Router

[English](./agent-workflow-router-overview.md) | [繁體中文](./agent-workflow-router-overview-zh-TW.md)

## What This Is

`openspec-agent-workflow-router` is a lightweight workflow router for OpenSpec / SDD projects. It turns a large multi-agent workflow into small, stage-specific prompts.

Instead of giving every agent a full lifecycle prompt, the router first identifies the current stage, then loads only the matching prompt:

```text
plan-change
review-plan
apply-change
review-result
fix-from-review
readiness-check
archive-change
```

## Goals

- Reduce token waste by loading one stage prompt at a time.
- Keep planning, review, implementation, fixing, readiness, and archive responsibilities separate.
- Make handoffs between Planner, Reviewer, Implementer, Coordinator, and Archivist consistent.
- Encourage spec-first development instead of chat-driven implementation.
- Keep git commit and push human-controlled by default.

## Problems It Solves

### Agents Read Too Much

Large prompts often force an agent to read planning, review, implementation, and archive instructions even when the task only needs one stage. This increases cost and makes the agent less focused.

### Review Happens Too Late

Many AI coding flows review only after code exists. By adding `review-plan`, design and task problems can be caught before implementation starts.

### Handoffs Are Inconsistent

Different agents often use different output formats. This router standardizes findings, verdicts, validation summaries, and archive readiness.

### Specs Drift From Code

Without explicit gates, tasks may be marked complete before validation, or implementation may drift outside the approved spec. The workflow keeps tasks, validation, review, and archive tied together.

## Real Pitfalls This Avoids

These are problems seen in real agent workflows:

- Loading the entire repository before knowing the actual stage.
- Reading `node_modules`, build output, coverage output, generated files, or vendored code by accident.
- Treating chat history as stronger than approved OpenSpec artifacts.
- Marking tasks complete before tests, build, or validation actually passed.
- Letting reviewer findings sit unresolved while still moving to archive.
- Using hard-coded mappings, keyword shortcuts, or one-off branches to pass a single case.
- Running archive before checking whether Blocker or Major findings still exist.
- Producing a commit message before the final scope and validation are known.

## How To Use It

Use the copy-paste prompts in:

```text
openspec/prompts/
```

Or install the Codex skill template from:

```text
openspec/templates/codex-skill/openspec-workflow-router/
```

Then ask for a specific stage:

```text
Use openspec-workflow-router to run review-result for change <change_name>.
Context:
<implementation summary, changed files, diff summary, validation results>
```

If the stage is unclear, the agent should ask one concise question instead of loading multiple prompts.
