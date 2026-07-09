# Schemas

本目錄提供多 Agent Coding Workflow 的三份核心契約：

- `agent-result.schema.json`：規範 Coordinator、Implementer、Reviewer、Readiness、Archive 的統一輸出 Envelope。
- `handoff-envelope.schema.json`：規範角色間交接時需要傳遞的輸入引用、預期輸出與轉移策略。
- `current-state.schema.json`：規範目前工作流狀態、Owner、Gate、Blocker 與下一步。

建議做法：

1. 先用 Schema 驗證資料形狀。
2. 再用程式檢查 Context Invariant，例如 `changeId`、`runId`、`producer`、`kind` 是否符合當前任務。
3. 不要讓 Schema 直接承擔權限控制、Secret 掃描或 Git Gate 判斷。
