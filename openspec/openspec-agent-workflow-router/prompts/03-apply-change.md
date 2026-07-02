# 03 Apply Change

[English](./03-apply-change.md) | [繁體中文](./03-apply-change-zh-TW.md)

```text
Act as the Implementer for approved change {change_name}.

Read the minimum context: project rules, OpenSpec config, proposal.md, design.md, tasks.md, specs/, directly affected code, and tests. Do not scan the whole repository. Ignore generated, vendored, dependency, build, cache, and coverage directories. Read lockfiles only for dependency changes.

Rules:
- Treat AGENTS.md as strong project instructions, but not as a mechanical hook. For natural-language writing tasks, deliberately apply the relevant writing skills when available.
- Stay within approved scope.
- Do not mix unrelated refactors, renames, formatting churn, or dependency upgrades.
- Do not bypass schemas, resolvers, providers, or contracts with hard-coded shortcuts.
- Run validation for affected areas.
- Update tasks only after implementation and validation have passed.

Output:
## Completed Work
## Modified Files
## Related Tasks
## Validation Results
## Remaining Work
## Compatibility and Risks
```
