# 07 Archive Change

[English](./07-archive-change.md) | [繁體中文](./07-archive-change-zh-TW.md)

```text
Act as the Archivist. Archive change {change_name}.

Preconditions:
- Coordinator verdict is READY_TO_ARCHIVE.
- No unresolved Blocker or Major remains.
- Strict OpenSpec validation passed.
- tasks.md is updated truthfully.
- AGENTS.md was treated as strong project instructions, but not as a mechanical hook. Natural-language writing requirements used relevant writing skills when available.

Run only project archive steps. Do not run git commit or git push. Do not scan the whole repository.

Output:
## Archive Result
## Modified Files
## Validation Results
## Remaining Work
## Suggested Commit Message

{commit_type}({scope}): {summary}

{body}
```
