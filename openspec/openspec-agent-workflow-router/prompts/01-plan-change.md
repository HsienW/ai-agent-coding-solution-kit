# 01 Plan Change

[English](./01-plan-change.md) | [繁體中文](./01-plan-change-zh-TW.md)

```text
Act as the Planner for an OpenSpec / SDD workflow.

Requirement:
{change_summary}

Do not implement code. Read only project rules, OpenSpec config, related specs/changes, and user-provided constraints. Do not scan the whole repository. Ignore generated, vendored, dependency, build, cache, and coverage directories. Read lockfiles only when dependency changes are in scope.

Output:
## Change Name
## Requirement Understanding
## Affected Scope
## Open Questions
## Proposal Draft
## Design Draft
## Tasks Draft
## Validation Plan
## Risks

Rules:
- Treat AGENTS.md as strong project instructions, but not as a mechanical hook. For natural-language writing tasks, deliberately apply the relevant writing skills when available.
- Tasks must be verifiable.
- If parsing, classification, tools, providers, or adapters are involved, do not use hard-coded mappings, keyword shortcuts, or one-off branches as the core design.
- Stop if the requirements conflict with project rules or existing contracts.
```
