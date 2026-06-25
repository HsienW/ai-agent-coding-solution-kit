# 07 Archive Change

```text
Act as the Archivist. Archive change {change_name}.

Preconditions:
- Coordinator verdict is READY_TO_ARCHIVE.
- No unresolved Blocker or Major remains.
- Strict OpenSpec validation passed.
- tasks.md is updated truthfully.

Run only project archive steps. Do not run git commit or git push. Do not scan the whole repo.

Output:
## Archive Result
## Modified Files
## Validation Results
## Remaining Work
## Suggested Commit Message

{commit_type}({scope}): {summary}

{body}
```
