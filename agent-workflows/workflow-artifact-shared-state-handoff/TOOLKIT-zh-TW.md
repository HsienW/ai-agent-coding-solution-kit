# Artifact-based Shared State + Structured Handoff 工程資產包

這個目錄提供可直接複製到專案中的工程資產，用來把人工轉述式多 Agent Coding Workflow 升級為可驗證、可交接、可逐步編排的工作流。

## 本包內容

- `schemas/`：三份 Draft 2020-12 JSON Schema，定義 Agent Result、Handoff Envelope 與 Current State。
- `templates/`：Current State、Handoff、Execution Summary、Workflow Policy、`.gitignore` 與導入 Checklist。
- `prompts/`：Coordinator、Implementer、Reviewer、Readiness / Archive 的工具中立 Prompt。
- `examples/`：一個脫敏的 generic feature change，展示從規劃、實作、審查到歸檔的 Artifact 流轉。

## 設計邊界

本資產包不綁定任何公司、業務領域、CLI、模型供應商或 Agent Framework。角色統一抽象為：

- Coordinator：規劃、仲裁、Scope 控制。
- Implementer：實作、測試、產出 Evidence。
- Reviewer：唯讀審查，產出 Review Result。
- Readiness / Archive：根據 State、Result、Evidence 判定是否可歸檔。
- Human Approver：控制 Commit、Push、Deploy 等不可逆操作。

## 最小導入順序

1. 把 `schemas/` 複製到專案的工作流規格目錄。
2. 把 `templates/gitignore.snippet` 合併到專案 `.gitignore`。
3. 根據 `templates/current-state.template.json` 建立第一份 Current State。
4. 讓每個 Agent 只輸出符合 `agent-result.schema.json` 的結果。
5. 以 `handoff-envelope.schema.json` 管理下一個角色需要讀取的 Artifact Reference。
6. 只有完成 Readiness 後，才產出 `execution-summary.template.md` 對應的長期摘要。

## 重要原則

- Shared State 不等於把所有 Agent 對話寫成 Markdown。
- Handoff 傳 Artifact Reference，不複製全文。
- Reviewer 預設唯讀，不直接修改程式碼或執行不可逆操作。
- Git Commit / Push / Deploy 應保留 Human-in-the-loop Gate。
- `.agent-runtime/` 預設不進 Git；若需要團隊共享，應升級為遠端 Runtime Store，而不是把 Runtime Log 提交進 Repository。
