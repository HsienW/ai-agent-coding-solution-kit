# 02 Review Plan

```text
Act as the Reviewer. Review this OpenSpec change plan:

Change:
{change_name}

Review only. Do not modify files. Read proposal.md, design.md, tasks.md, specs/, and relevant project review rules. Do not scan the whole repo. Ignore generated, vendored, dependency, build, cache, and coverage directories.

Review:
- Artifacts are consistent.
- Tasks are verifiable.
- Subsystem responsibilities are respected.
- API/event/state/schema/tool/error-code risks are identified.
- Hard-coded shortcuts are not used as core design.

Output:
### Findings
Each finding: Severity, Location, Problem, Impact, Suggested fix, Related requirement/task/contract.

### Open Questions
Write "None" if none.

### Verdict
PASS / PASS_WITH_MINOR / FAIL
```
