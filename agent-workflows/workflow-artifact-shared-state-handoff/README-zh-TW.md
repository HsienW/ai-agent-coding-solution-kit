# Artifact-based Shared State + Structured Handoff

> 從人工搬運上下文，演進到可驗證、可持久化、可編排的多 Agent Coding Workflow。

## 工作流定位

這個工作流討論一個常見但容易被低估的問題：

當規劃者、實作者、審查者與人工批准者分別由不同 AI Agent 或 CLI 工具承擔時，前一個角色的結論往往無法直接成為下一個角色的可靠輸入。使用者只好手工複製規格、修改摘要、Diff、測試結果與 Review Findings，並在每個階段重新解釋背景。

這種模式已經具有多角色分工，但人仍同時扮演：

- Context Router
- Clipboard Transport
- Shared State Store
- Workflow Engine
- Integration Arbiter
- Git Gatekeeper

本文提出兩個互補機制，作為核心改造：

1. **Artifact-based Shared State**：保存各角色共同依據的工作事實。
2. **Structured Handoff**：定義任務如何從一個角色可靠交給下一個角色。

兩者可簡化為：

```text
Artifact-based Shared State
= 大家共同相信什麼

Structured Handoff
= 現在由誰，基於哪些事實，完成什麼工作
```

## 為什麼值得單獨設計？

Prompt、Skill、AGENTS.md 或其他規則資產可以降低重複指令，但它們主要解決「Agent 應如何工作」，不會自動解決：

- 現在處於哪一個階段？
- 哪一份實作結果是最新且已接受的？
- 哪些測試真的執行過？
- Review 是否存在未處理的 Blocker？
- 下一個角色必須讀取哪些輸入？
- 失敗後應回到實作、規格仲裁，還是人工處理？
- 是否允許 Archive、Commit、Push 或 Deploy？

只要這些資訊仍存在人的記憶或剪貼簿裡，流程就很難進一步自動化，也很難保證可恢復、可稽核與可重跑。

## 核心架構

```text
┌──────────────────────────────────────────────┐
│ Durable Knowledge                            │
│ Requirements / Design / Tasks / Decisions    │
└──────────────────────┬───────────────────────┘
                       │ references
                       ▼
┌──────────────────────────────────────────────┐
│ Artifact-based Shared State                  │
│ Current State / Results / Evidence Refs      │
└──────────────────────┬───────────────────────┘
                       │ structured handoff
                       ▼
┌──────────────────────────────────────────────┐
│ Coordinator → Implementer → Reviewer         │
│      ↑                 │          │           │
│      └── arbitration ──┴── fixes ─┘           │
└──────────────────────┬───────────────────────┘
                       │ protected gate
                       ▼
┌──────────────────────────────────────────────┐
│ Human Approval                               │
│ Commit / Push / Deploy / Destructive Action  │
└──────────────────────────────────────────────┘
```

從架構責任來看：

- Shared State 是 **Data Plane**。
- Structured Handoff 是 **Control Plane**。
- Orchestrator 是未來負責執行狀態轉移的 **Process Manager**。
- Human-in-the-loop 是不可逆操作與高風險邊界的 **Approval Gate**。

## 本工作流適用場景

適合：

- Planner、Coder、Reviewer 分別使用不同模型或 CLI。
- 已有 SDD、OpenSpec、ADR、Issue 或其他規格資產。
- Reviewer 採唯讀權限，不能直接修改程式碼。
- Commit、Push、Deploy 仍需要人工批准。
- 希望逐步導入 Node.js、Python 或 Workflow Engine 編排。
- 希望降低重複 Prompt 與上下文搬運，而不是追求完全無人化。

不一定需要：

- 單一 Agent 可以在一次短 Session 內完成的簡單任務。
- 沒有角色分工、沒有審查 Gate 的一次性腳本生成。
- 完全不需要恢復、稽核、重試或版本辨識的實驗。

## 這不是什麼

本工作流不是：

- 特定模型供應商的專用方案。
- 特定 CLI 的操作教學。
- 一套必須安裝的 Agent Framework。
- 要求所有狀態都寫成 Markdown 的文件治理方案。
- 讓 LLM 自由決定所有流程與 Git 操作的自治系統。
- 宣稱所有大型公司都採用同一實作的「唯一最佳實踐」。

公開框架與協議在 Shared State、Artifact、Handoff、Persistence、HITL 等方向上有明顯共通點，但具體資料模型、權限與儲存方式仍應依團隊規模與風險調整。

## 建議閱讀順序

1. [`docs/01-problem-and-maturity-model-zh-TW.md`](docs/01-problem-and-maturity-model-zh-TW.md)  
   從人工編排工作流開始，辨識真正的瓶頸與成熟度。
2. [`docs/02-reference-architecture-zh-TW.md`](docs/02-reference-architecture-zh-TW.md)  
   定義 Shared State、Artifact、Handoff、State Machine 與角色邊界。
3. [`docs/03-runtime-storage-and-retention-zh-TW.md`](docs/03-runtime-storage-and-retention-zh-TW.md)  
   解決 Repo 文件爆炸、Runtime State、Evidence 與 Trace 分離。
4. [`docs/04-adoption-roadmap-zh-TW.md`](docs/04-adoption-roadmap-zh-TW.md)  
   提供從人工流程到 Node Orchestrator 的分階段導入路線。
5. [`docs/05-security-and-maintainability-zh-TW.md`](docs/05-security-and-maintainability-zh-TW.md)  
   處理路徑安全、權限、Schema、版本、併發與長期維護。
6. [`patterns/README-zh-TW.md`](patterns/README-zh-TW.md)  
   將整套設計映射到 Blackboard、Repository、State Machine、Process Manager 等經典模式。

## 第一批文章涵蓋範圍

本批內容集中在知識與架構，不提供正式 Schema、Prompt 或可執行 Orchestrator。後續工程資產可以沿用以下目錄：

```text
artifact-shared-state-handoff/
├─ schemas/
│  ├─ agent-result.schema.json
│  ├─ handoff-envelope.schema.json
│  └─ current-state.schema.json
├─ templates/
├─ prompts/
└─ examples/
```

## 快速判斷：Node.js 腳本是否已算多 Agent Orchestrator？

只有依序執行：

```text
run planner
run coder
run reviewer
```

通常仍只是自動化 Pipeline。

當腳本開始負責以下能力時，才接近真正的 Workflow Orchestrator：

- 載入並驗證 Current State。
- 驗證前置條件與 Agent 輸出 Schema。
- 保存 Artifact 與 Evidence Reference。
- 根據結果推進或拒絕狀態轉移。
- 對 Review Findings 建立修正回路。
- 對設計衝突升級給 Coordinator。
- 在人工批准點 Pause／Resume。
- 支援 Retry、Idempotency 與 Recovery。

## 最小原則

可以用一句話概括本工作流：

> 先把人工流程轉成穩定契約，再把穩定契約交給程式編排。

不要一開始就導入資料庫、Queue、完整 Trace 與全自動 Git 操作。第一個可用版本只需要：

- 一份可驗證的 Current State。
- 一種共用 Agent Result Envelope。
- 一種 Structured Handoff Envelope。
- 一張明確的狀態轉移表。
- 一個唯讀 Reviewer 邊界。
- 一個人工 Commit Gate。

## 主要參考資料

- LangGraph：State 是節點共同使用的共享記憶；Checkpointer 可保存狀態快照並支援 HITL 與故障恢復。  
  <https://docs.langchain.com/oss/javascript/langgraph/thinking-in-langgraph>  
  <https://docs.langchain.com/oss/javascript/langgraph/persistence>
- OpenAI Agents SDK：Handoff、Code-led／LLM-led Orchestration 與 Human-in-the-loop。  
  <https://openai.github.io/openai-agents-js/guides/handoffs/>  
  <https://openai.github.io/openai-agents-js/guides/multi-agent/>  
  <https://openai.github.io/openai-agents-js/guides/human-in-the-loop/>
- Agent2Agent Protocol：Task、Message、Artifact 與 Artifact Lifecycle。  
  <https://a2a-protocol.org/latest/topics/key-concepts/>  
  <https://a2a-protocol.org/latest/topics/life-of-a-task/>
- Enterprise Integration Patterns：Process Manager、Message、Pipes and Filters、Claim Check。  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html>
- Azure Architecture Center：Event Sourcing、Materialized View、Claim Check。  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing>  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view>  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check>

---

本工作流以框架中立、模型中立與規格工具中立為原則。採用前仍應依實際 CLI 版本、權限模型、資料敏感度與團隊流程進行驗證。
