---
name: openspec-workflow-router
description: Route OpenSpec / SDD change lifecycle work to token-efficient stage prompts. Use when an agent needs to plan, review, implement, fix, readiness-check, or archive an OpenSpec change with minimal context loading.
---

# OpenSpec Workflow Router

Use this skill as a routing layer. Pick one stage and read only the matching reference.

## Context Policy

- Prefer user-provided summaries, diffs, validation results, and named files.
- Read the minimum relevant OpenSpec artifacts and adjacent files.
- Do not scan the whole repository by default.
- Ignore generated, vendored, dependency, build, cache, and coverage directories.
- Read lockfiles only when dependency changes are in scope.

## Route Table

| Intent | Stage | Read |
| --- | --- | --- |
| Plan a change | `plan-change` | `references/01-plan-change.md` |
| Review a plan | `review-plan` | `references/02-review-plan.md` |
| Implement a change | `apply-change` | `references/03-apply-change.md` |
| Review implementation | `review-result` | `references/04-review-result.md` |
| Fix review findings | `fix-from-review` | `references/05-fix-from-review.md` |
| Check archive readiness | `readiness-check` | `references/06-readiness-check.md` |
| Archive a change | `archive-change` | `references/07-archive-change.md` |

If the stage is ambiguous, ask one concise question instead of loading multiple references.
