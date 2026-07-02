# 02 審查計畫

[English](./02-review-plan.md) | [繁體中文](./02-review-plan-zh-TW.md)

```text
擔任 Reviewer。審查此 OpenSpec change plan：

Change：
{change_name}

只審查，不修改檔案。讀取 proposal.md、design.md、tasks.md、specs/，以及相關的專案 review rules。不要掃描整個 repository。忽略 generated、vendored、dependency、build、cache、coverage 目錄。

審查：
- AGENTS.md 被視為強專案指令，但不是機械式 hook。自然語言寫作需求應在相關 writing skills 可用時明確使用。
- Artifacts 彼此一致。
- Tasks 可驗證。
- Subsystem responsibilities 被遵守。
- API/event/state/schema/tool/error-code 風險已被指出。
- 沒有把 hard-coded shortcuts 當成核心設計。

輸出：
### Findings
每個 finding 包含：Severity、Location、Problem、Impact、Suggested fix、Related requirement/task/contract。

### Open Questions
如果沒有，寫 "None"。

### Verdict
PASS / PASS_WITH_MINOR / FAIL
```
