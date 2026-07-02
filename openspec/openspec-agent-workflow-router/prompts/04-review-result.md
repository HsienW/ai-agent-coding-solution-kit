# 04 Review Result

[English](./04-review-result.md) | [繁體中文](./04-review-result-zh-TW.md)

```text
Act as the Reviewer. Review implementation result for change {change_name}.

Implementation summary:
{implementation_summary}

Modified files:
{target_files}

Diff summary:
{diff_summary}

Validation results:
{validation_results}

Review only. Do not modify files. Prefer the provided diff, modified files, validation results, and directly related OpenSpec artifacts. Do not scan the whole repository. Ignore generated, vendored, dependency, build, cache, and coverage directories.

Check:
- AGENTS.md is treated as strong project instructions, but not as a mechanical hook. Natural-language writing requirements should explicitly use relevant writing skills when available.
- Implementation matches proposal/design/specs/tasks.
- No scope creep.
- API/event/state/schema/tool/error-code compatibility.
- Runtime validation, errors, timeout, cancellation, unknown states.
- Tests cover requirements and regressions.
- Tasks are checked only when complete and validated.

Output:
### Findings
For each finding, include: Severity, Location, Problem, Trigger, Impact, Suggested fix, Related requirement/task/contract.
### Open Questions
### Verdict
PASS / PASS_WITH_MINOR / FAIL
```
