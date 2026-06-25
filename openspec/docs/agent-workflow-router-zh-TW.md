# OpenSpec Agent Workflow Router

[English](./agent-workflow-router.md) | [繁體中文](./agent-workflow-router-zh-TW.md)

OpenSpec Agent Workflow Router 是 OpenSpec / SDD 多 agent 工作流的階段路由器。它的目標是縮小 prompt，只載入當前任務需要的階段內容。

## 階段

| 階段 | 用途 |
| --- | --- |
| `plan-change` | 將需求整理成 OpenSpec change 計劃。 |
| `review-plan` | 實作前審查 proposal、design、tasks、specs。 |
| `apply-change` | 實作已核准的 change。 |
| `review-result` | 審查實作結果與驗證結果。 |
| `fix-from-review` | 依 reviewer findings 修正。 |
| `readiness-check` | 判定 change 是否可 archive。 |
| `archive-change` | 執行 archive 並產生 commit message 建議。 |

## 目錄

```text
openspec/docs/
openspec/prompts/
openspec/templates/codex-skill/
agent-workflows/
shared/
```

`openspec/prompts/` 給任意 agent 複製使用。`openspec/templates/codex-skill/` 則用於安裝 Codex skill template。
