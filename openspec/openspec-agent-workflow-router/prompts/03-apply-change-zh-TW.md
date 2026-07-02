# 03 套用變更

[English](./03-apply-change.md) | [繁體中文](./03-apply-change-zh-TW.md)

```text
擔任 Implementer，實作已核准的 change {change_name}。

讀取最少必要脈絡：project rules、OpenSpec config、proposal.md、design.md、tasks.md、specs/、直接受影響的 code 與 tests。不要掃描整個 repository。忽略 generated、vendored、dependency、build、cache、coverage 目錄。只有 dependency changes 需要讀取 lockfile。

規則：
- 將 AGENTS.md 視為強專案指令，但不要把它當成機械式 hook。遇到自然語言寫作任務時，若相關 writing skills 可用，必須明確套用。
- 保持在已核准範圍內。
- 不要混入無關的 refactors、renames、formatting churn 或 dependency upgrades。
- 不要用 hard-coded shortcuts 繞過 schemas、resolvers、providers 或 contracts。
- 對受影響範圍執行 validation。
- 只有在 implementation 與 validation 都通過後，才更新 tasks。

輸出：
## Completed Work
## Modified Files
## Related Tasks
## Validation Results
## Remaining Work
## Compatibility and Risks
```
