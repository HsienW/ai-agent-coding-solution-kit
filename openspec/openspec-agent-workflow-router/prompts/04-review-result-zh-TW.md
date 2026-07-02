# 04 審查結果

[English](./04-review-result.md) | [繁體中文](./04-review-result-zh-TW.md)

```text
擔任 Reviewer。審查 change {change_name} 的 implementation result。

Implementation summary：
{implementation_summary}

Modified files：
{target_files}

Diff summary：
{diff_summary}

Validation results：
{validation_results}

只審查，不修改檔案。優先使用提供的 diff、modified files、validation results，以及直接相關的 OpenSpec artifacts。不要掃描整個 repository。忽略 generated、vendored、dependency、build、cache、coverage 目錄。

檢查：
- AGENTS.md 被視為強專案指令，但不是機械式 hook。自然語言寫作需求應在相關 writing skills 可用時明確使用。
- Implementation 符合 proposal/design/specs/tasks。
- 沒有 scope creep。
- API/event/state/schema/tool/error-code 相容性。
- Runtime validation、errors、timeout、cancellation、unknown states。
- Tests 覆蓋 requirements 與 regressions。
- Tasks 只有在完成且已驗證時才勾選。

輸出：
### Findings
每個 finding 包含：Severity、Location、Problem、Trigger、Impact、Suggested fix、Related requirement/task/contract。
### Open Questions
### Verdict
PASS / PASS_WITH_MINOR / FAIL
```
