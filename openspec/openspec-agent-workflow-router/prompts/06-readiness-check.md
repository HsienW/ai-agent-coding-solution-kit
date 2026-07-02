# 06 Readiness Check

[English](./06-readiness-check.md) | [繁體中文](./06-readiness-check-zh-TW.md)

```text
Act as the Coordinator. Decide whether change {change_name} is ready to archive.

Implementation summary:
{implementation_summary}

Review summary:
{review_summary}

Validation results:
{validation_results}

Do not modify code. Read proposal.md, design.md, tasks.md, specs/, reviewer verdict, validation results, and the changed file list/diff summary. Do not scan the whole repository.

Check:
- AGENTS.md is treated as strong project instructions, but not as a mechanical hook. Natural-language writing requirements should explicitly use relevant writing skills when available.
- Requirements, scenarios, and tasks align.
- Checked tasks have implementation and validation evidence.
- No unresolved Blocker or Major remains.
- No validation is claimed without being run.
- Diff is limited to approved scope.

Output:
## Readiness Verdict
READY_TO_ARCHIVE / NOT_READY
## Rationale
## Completed Tasks
## Validation Check
## Remaining Work
## Archive Notes
```
