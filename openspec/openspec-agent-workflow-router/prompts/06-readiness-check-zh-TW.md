# 06 就緒檢查

[English](./06-readiness-check.md) | [繁體中文](./06-readiness-check-zh-TW.md)

```text
擔任 Coordinator。判斷 change {change_name} 是否已準備好 archive。

Implementation summary：
{implementation_summary}

Review summary：
{review_summary}

Validation results：
{validation_results}

不要修改程式碼。讀取 proposal.md、design.md、tasks.md、specs/、reviewer verdict、validation results，以及 changed file list/diff summary。不要掃描整個 repository。

檢查：
- AGENTS.md 被視為強專案指令，但不是機械式 hook。自然語言寫作需求應在相關 writing skills 可用時明確使用。
- Requirements、scenarios、tasks 一致。
- 已勾選的 tasks 有 implementation 與 validation evidence。
- 沒有未解決的 Blocker 或 Major。
- 沒有宣稱 validation 已執行但實際未執行。
- Diff 限於已核准範圍。

輸出：
## Readiness Verdict
READY_TO_ARCHIVE / NOT_READY
## Rationale
## Completed Tasks
## Validation Check
## Remaining Work
## Archive Notes
```
