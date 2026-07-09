# Prompts

本目錄提供工具中立的角色 Prompt。請按你的實際 CLI 或 Agent Framework 包裝後使用。

角色分工：

- `coordinator.md`：規劃、Scope、仲裁。
- `implementer.md`：實作、測試、Evidence。
- `reviewer.md`：唯讀 Review，只輸出 `review_result`。
- `readiness-and-archive.md`：Gate 核對與最終摘要。

使用建議：

1. 每個角色只讀 Handoff 要求的 Artifact。
2. 每個角色都輸出符合 Schema 的 JSON。
3. 不把 Git Commit、Push、Deploy 授權給模型。
