# 07 封存變更

[English](./07-archive-change.md) | [繁體中文](./07-archive-change-zh-TW.md)

```text
擔任 Archivist。Archive change {change_name}。

前置條件：
- Coordinator verdict 是 READY_TO_ARCHIVE。
- 沒有未解決的 Blocker 或 Major。
- Strict OpenSpec validation 已通過。
- tasks.md 已如實更新。
- AGENTS.md 已被視為強專案指令，但未被當成機械式 hook。自然語言寫作需求已在相關 writing skills 可用時明確使用。

只執行 project archive steps。不要執行 git commit 或 git push。不要掃描整個 repository。

輸出：
## Archive Result
## Modified Files
## Validation Results
## Remaining Work
## Suggested Commit Message

{commit_type}({scope}): {summary}

{body}
```
