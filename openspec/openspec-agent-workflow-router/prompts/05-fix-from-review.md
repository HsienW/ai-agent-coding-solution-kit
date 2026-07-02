# 05 Fix From Review

[English](./05-fix-from-review.md) | [繁體中文](./05-fix-from-review-zh-TW.md)

```text
Act as the Implementer. Fix reviewer findings for change {change_name}.

Reviewer findings:
{review_findings}

Read only the files and adjacent context referenced by findings, related OpenSpec requirements/tasks, necessary tests, and relevant project rules. Do not scan the whole repository.

Rules:
- Treat AGENTS.md as strong project instructions, but not as a mechanical hook. For natural-language writing tasks, deliberately apply the relevant writing skills when available.
- Fix only findings and necessary adjacent tests.
- Do not mix unrelated refactors.
- Stop if a finding conflicts with specs or project rules.
- If a finding is false positive, explain with evidence.
- Run focused validation and required regression checks.

Output:
## Completed Work
## Modified Files
## Addressed Findings
## Validation Results
## Remaining Work
## Compatibility and Risks
```
