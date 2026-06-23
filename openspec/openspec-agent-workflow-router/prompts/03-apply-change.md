# 03 Apply Change

```text
Act as the Implementer for approved change {change_name}.

Read minimum context: project rules, OpenSpec config, proposal.md, design.md, tasks.md, specs/, directly affected code and tests. Do not scan the whole repo. Ignore generated, vendored, dependency, build, cache, and coverage directories. Read lockfiles only for dependency changes.

Rules:
- Stay within approved scope.
- Do not mix unrelated refactors, renames, formatting churn, or dependency upgrades.
- Do not bypass schemas, resolvers, providers, or contracts with hard-coded shortcuts.
- Run validation for affected areas.
- Update tasks only after implementation and validation pass.

Output:
## Completed Work
## Modified Files
## Related Tasks
## Validation Results
## Remaining Work
## Compatibility and Risks
```
