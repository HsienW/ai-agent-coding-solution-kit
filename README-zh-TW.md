# 🎓 AI Agent Code Solution Kit

[![English](https://img.shields.io/badge/English-555555?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-0969da?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-555555?style=for-the-badge)](./README-zh-CN.md)

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
* **Agent Thoughts**：AI Agent 的基礎認知與操作使用，以及 AI 在軟體開發中的經驗與心得
* **可複用資產**：通用 Markdown 文件、模板、Checklist、Agent 規則與 Prompt
* **共享規則**：跨角色、跨流程重複使用的規則文件

## 📂 目錄結構

| 方向 | 核心內容 | 閱讀入口 |
| --- | --- | --- |
| OpenSpec / SDD | 規格驅動開發、階段路由、規劃與審查、實作驗收與歸檔 | [OpenSpec 工作流路由](./openspec/openspec-agent-workflow-router/README-zh-TW.md) |
| Context Engineering | 上下文路由、漸進式披露、RAG、記憶設計、Runtime 與治理 | [上下文工程](./context-engineering/README-zh-TW.md) |
| Prompt Engineering | Prompt 契約、結構化產物生成、注入防護、評估與版本管理 | [Prompt 工程](./prompt-engineering/README-zh-TW.md)、[結構化產物生成](./prompt-engineering/prompts/structured-artifact-generation/) |
| Agent Design | Tool Schema、Tool Contract、能力路由、權限邊界與評測 | [Agent 設計](./agent-design/README-zh-TW.md)、[Tool Schema 與 Routing](./agent-design/tool-schema-routing/README-zh-TW.md) |
| Agent Workflows | 多 Agent 生命週期、共享工作狀態、結構化交接與協作流程 | [OpenSpec 多 Agent 生命週期](./agent-workflows/openspec-multi-agent-lifecycle/)、[共享狀態與結構化交接](./agent-workflows/workflow-artifact-shared-state-handoff/README-zh-TW.md) |
| Agent Thoughts | AI Agent 基礎認知、操作使用與 AI 軟體開發心得 | [AI Agent 基礎認知與操作](./agent-thoughts/ai-agent-basics/)、[AI 軟體開發心得](./agent-thoughts/ai-software-development/) |
| 共享規則 | 跨角色、跨流程重複使用的規則文件 | [共享規則目錄](./shared/) |

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
