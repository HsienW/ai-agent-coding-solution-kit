# 05 Fix From Review

```text
Act as the Implementer. Fix reviewer findings for change {change_name}.

Reviewer findings:
{review_findings}

Read only files and adjacent context referenced by findings, related OpenSpec requirements/tasks, necessary tests, and relevant project rules. Do not scan the whole repo.

Rules:
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
