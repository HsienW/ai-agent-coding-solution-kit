# Agent Design

[English](./README.md) | [繁體中文](./README-zh-TW.md)

一套不綁定特定框架的 AI Agent 設計方法、契約、模式與營運實踐集合，目標是建立可控、可評測、可治理的 Agent 系統。

> 這個 `模式` 記錄參考架構。
> 所有場景、識別符、政策、Payload 與指標皆為合成資料。
> 生產落地仍需依實際業務完成安全、隱私、法務、可靠性與評測驗證。

## 收錄範圍

`agent-design` 聚焦 Agent 執行期的能力與契約邊界：

- Tool Schema 與 Tool Contract
- Tool Routing 與候選能力選擇
- Agent 狀態與工作流邊界
- Human-in-the-loop 控制
- Guardrail、權限與授權
- 規劃與執行策略
- 評測、可觀測性、發布與回滾

它與倉庫其他 `模式` 的分工如下：

- `context-engineering`：決定模型在每一步應看到什麼資訊。
- `agent-workflows`：描述研發期間多個 Coding Agent 如何協作。
- `agent-design`：定義應用 Agent 在執行期如何選能力、調工具並安全行動。

## 模式

### [Tool Schema 與 Routing](./tool-schema-routing/README-zh-TW.md)

一套用來設計模型可見 Tool Contract、隔離相似 Domain、路由大型 Tool Catalog，並治理 Tool 行為的參考架構。

核心觀點：

- 語義分域、內核共享
- Router Contract 穩定，Domain Registry 動態擴展
- 高風險能力使用專屬 Tool，低風險長尾能力使用 Group Tool
- 結構化事實優先於模型推斷
- 評測與可觀測性必須和 Schema 一起設計
