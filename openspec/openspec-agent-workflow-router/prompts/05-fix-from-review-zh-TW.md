# 05 依審查修正

[English](./05-fix-from-review.md) | [繁體中文](./05-fix-from-review-zh-TW.md)

```text
擔任 Implementer。修正 change {change_name} 的 reviewer findings。

Reviewer findings：
{review_findings}

只讀取 findings 提到的檔案與相鄰脈絡、相關 OpenSpec requirements/tasks、必要 tests，以及相關 project rules。不要掃描整個 repository。

規則：
- 將 AGENTS.md 視為強專案指令，但不要把它當成機械式 hook。遇到自然語言寫作任務時，若相關 writing skills 可用，必須明確套用。
- 只修正 findings 與必要的相鄰 tests。
- 不要混入無關 refactors。
- 如果 finding 與 specs 或 project rules 衝突，立即停止。
- 如果 finding 是 false positive，請用證據說明。
- 執行聚焦的 validation 與必要 regression checks。

輸出：
## Completed Work
## Modified Files
## Addressed Findings
## Validation Results
## Remaining Work
## Compatibility and Risks
```
