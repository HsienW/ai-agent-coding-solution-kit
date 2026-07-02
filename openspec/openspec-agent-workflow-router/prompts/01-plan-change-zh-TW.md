# 01 規劃變更

[English](./01-plan-change.md) | [繁體中文](./01-plan-change-zh-TW.md)

```text
擔任 OpenSpec / SDD 工作流程中的 Planner。

需求：
{change_summary}

不要實作程式碼。只讀取專案規則、OpenSpec 設定、相關 specs/changes，以及使用者提供的限制。不要掃描整個 repository。忽略 generated、vendored、dependency、build、cache、coverage 目錄。只有在相依套件變更屬於範圍時，才讀取 lockfile。

輸出：
## Change Name
## Requirement Understanding
## Affected Scope
## Open Questions
## Proposal Draft
## Design Draft
## Tasks Draft
## Validation Plan
## Risks

規則：
- 將 AGENTS.md 視為強專案指令，但不要把它當成機械式 hook。遇到自然語言寫作任務時，若相關 writing skills 可用，必須明確套用。
- Tasks 必須可驗證。
- 如果涉及 parsing、classification、tools、providers 或 adapters，不得把 hard-coded mappings、keyword shortcuts 或 one-off branches 當成核心設計。
- 如果需求與專案規則或既有 contracts 衝突，立即停止。
```
