# 🎓 AI Agent Code Solution Kit

[English](./README.md) | [简体中文](./README-zh-CN.md)

> 持續更新的 AI Agent Coding 文件、模板與工程實踐知識庫。

本倉庫整理我在 AI Agent 開發與 AI 輔助程式設計過程中，實際研究、使用與驗證過的文件、Prompt、模板、規範與工作流程。
這些零散經驗會逐步整理成可閱讀、可複用、可持續維護的工程資產，供個人專案與團隊開發流程使用。

## 📚 主要內容

本倉庫目前聚焦以下方向：

* **OpenSpec / SDD**：需求研究、規格設計、任務拆解、Change 管理與驗收流程
* **Context Engineering**：上下文路由、漸進式披露、記憶設計與長任務管理
* **Prompt Engineering**：研究、規劃、開發、Review 與除錯 Prompt
* **Agent Design**：Agent 執行期架構、Tool Contract、Routing、Guardrail、治理與評測
* **Agent Workflows**：Claude Code、Codex CLI、Gemini CLI 等工具的協作模式
* **可複用資產**：通用 Markdown 文件、模板、Checklist、Agent 規則與 Prompt
* **共享規則**：跨角色、跨流程重複使用的規則文件

## 📂 目錄結構

```text
ai-agent-coding-solution-kit/
├─ openspec/
│  └─ openspec-agent-workflow-router/
│     ├─ docs/
│     ├─ prompts/
│     └─ templates/
│        └─ codex-skill/
├─ context-engineering/
│  ├─ docs/
│  ├─ patterns/
│  └─ templates/
├─ prompt-engineering/
│  ├─ docs/
│  ├─ prompts/
│  │  └─ structured-artifact-generation/
│  └─ templates/
├─ agent-design/
│  └─ tool-schema-routing/
│     ├─ docs/
│     ├─ patterns/
│     └─ templates/
├─ agent-workflows/
│  ├─ openspec-multi-agent-lifecycle/
│  └─ workflow-artifact-shared-state-handoff/
│     ├─ docs/
│     ├─ examples/
│     │  └─ generic-feature-change/
│     ├─ patterns/
│     ├─ prompts/
│     ├─ schemas/
│     └─ templates/
└─ shared/
```

各類內容有明確分工：

* `docs`：概念、方法與工程實踐紀錄
* `templates`：可直接複製、修改的文件骨架
* `prompts`：交給 AI Agent 使用的指令
* `patterns`：可跨專案複用的設計模式
* `examples`：實作與工作流程範例
* `schemas`：描述資料結構或交付物格式的規格

## 🌱 持續維護

本倉庫會隨 AI Coding Agent、OpenSpec、Context Engineering 與 Prompt Engineering 的實務發展持續更新。
內容主要來自個人研究與工程實踐，使用前仍應依實際模型、工具版本與專案環境完成驗證。
