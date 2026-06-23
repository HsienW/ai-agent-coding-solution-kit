[English](./openspec-multi-agent-lifecycle.md) | [繁體中文](./openspec-multi-agent-lifecycle-zh-TW.md)

# OpenSpec Multi-Agent Lifecycle

This lifecycle describes a tool-neutral workflow for OpenSpec / SDD projects. It separates planning, review, implementation, readiness, and archive so each agent reads only the prompt and context needed for its current stage.

## Lifecycle

```text
Requirement
  -> plan-change
  -> review-plan
  -> apply-change
  -> review-result
  -> fix-from-review, if needed
  -> readiness-check
  -> archive-change
```

## Roles

| Role | Responsibility |
| --- | --- |
| Planner | Turns a requirement into proposal, design, tasks, and spec deltas. |
| Reviewer | Reviews plans or implementation results and reports findings. |
| Implementer | Applies approved changes and fixes reviewer findings. |
| Coordinator | Checks whether requirements, tasks, validation, and review gates are complete. |
| Archivist | Runs archive steps and drafts the commit message. |

Concrete tools are project-specific. For example, a project may map Planner to CCR, Reviewer to Qwen, and Implementer to Codex.

## Stage Gates

### 1. plan-change

Input:

- Requirement summary.
- Known constraints.
- Relevant project rules or existing specs.

Output:

- Change name.
- Proposal draft.
- Design draft.
- Tasks draft.
- Validation plan.
- Open questions.

Gate: the plan must be concrete enough for review.

### 2. review-plan

Input:

- Proposal.
- Design.
- Tasks.
- Specs or spec deltas.

Output:

- Findings.
- Open questions.
- Verdict: `PASS`, `PASS_WITH_MINOR`, or `FAIL`.

Gate: implementation should not start while the plan has unresolved Blocker or Major findings.

### 3. apply-change

Input:

- Approved OpenSpec artifacts.
- Relevant project rules.
- Directly affected files and tests.

Output:

- Implementation summary.
- Modified files.
- Validation results.
- Updated tasks, only when work and validation are complete.

Gate: do not mark tasks complete before implementation and validation pass.

### 4. review-result

Input:

- Implementation summary.
- Modified files.
- Diff summary.
- Validation results.
- Related OpenSpec artifacts.

Output:

- Findings.
- Open questions.
- Verdict: `PASS`, `PASS_WITH_MINOR`, or `FAIL`.

Gate: unresolved Blocker or Major findings must go back to `fix-from-review`.

### 5. fix-from-review

Input:

- Reviewer findings.
- Files and tests referenced by those findings.
- Related requirements and tasks.

Output:

- Addressed findings.
- Modified files.
- Focused validation results.
- Remaining risks.

Gate: rerun review or readiness only after fixes and relevant validation are complete.

### 6. readiness-check

Input:

- Final implementation summary.
- Review verdict.
- Validation results.
- Changed file list or diff summary.

Output:

- Verdict: `READY_TO_ARCHIVE` or `NOT_READY`.
- Rationale.
- Remaining work.
- Archive notes.

Gate: archive requires `READY_TO_ARCHIVE`.

### 7. archive-change

Input:

- Readiness verdict.
- Final OpenSpec artifacts.
- Validation results.

Output:

- Archive result.
- Modified files.
- Validation result.
- Suggested commit message.

Gate: the workflow may draft a commit message, but `git commit` and `git push` remain human-controlled by default.

## Context Discipline

Each stage should read only:

- The matching stage prompt.
- User-provided summaries, diffs, and validation results.
- Relevant OpenSpec artifacts.
- Named files and adjacent tests.
- Relevant project rules.

Avoid repository-wide scans. Ignore generated, vendored, dependency, build, cache, and coverage directories. Read lockfiles only when dependency changes are in scope.

## Failure Rules

Stop and report instead of continuing when:

- Specs conflict with project rules.
- Requirements conflict with existing contracts.
- Review finds unresolved Blocker or Major issues.
- Validation is required but was not run.
- The diff includes unrelated scope.
