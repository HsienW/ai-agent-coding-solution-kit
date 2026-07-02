# 02 Review Plan

[English](./02-review-plan.md) | [繁體中文](./02-review-plan-zh-TW.md)

```text
Act as the Reviewer. Review this OpenSpec change plan:

Change:
{change_name}

Review only. Do not modify files. Read proposal.md, design.md, tasks.md, specs/, and relevant project review rules. Do not scan the whole repository. Ignore generated, vendored, dependency, build, cache, and coverage directories.

Review:
- AGENTS.md is treated as strong project instructions, but not as a mechanical hook. Natural-language writing requirements should explicitly use relevant writing skills when available.
- Artifacts are consistent.
- Tasks are verifiable.
- Subsystem responsibilities are respected.
- API/event/state/schema/tool/error-code risks are identified.
- Hard-coded shortcuts are not used as core design.

Output:
### Findings
For each finding, include: Severity, Location, Problem, Impact, Suggested fix, Related requirement/task/contract.

### Open Questions
Write "None" if none.

### Verdict
PASS / PASS_WITH_MINOR / FAIL
```
