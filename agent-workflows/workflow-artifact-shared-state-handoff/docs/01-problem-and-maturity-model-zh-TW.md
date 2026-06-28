# 01｜問題定義與多 Agent Coding Workflow 成熟度模型

[English](./01-problem-and-maturity-model.md) | [繁體中文](./01-problem-and-maturity-model-zh-TW.md)

## 1. 常見起點：多個 Agent，仍由人當中控台

一套常見的 AI 輔助開發流程如下：

```text
Coordinator / Planner
        │
        │ 人工複製規格與結論
        ▼
Implementer
        │
        │ 人工複製修改摘要、Diff、測試結果
        ▼
Reviewer
        │
        │ 人工複製 Findings 與 Verdict
        ▼
Coordinator / Human
        │
        ▼
Commit / Archive / Deploy
```

這個模式已經具備多 Agent 的基本條件：

- 角色專業化。
- 規劃、實作與審查分離。
- 單一寫入者。
- Review Gate。
- Human-in-the-loop。
- 對不可逆操作的人工控制。

較準確的名稱可以是：

- Human-orchestrated Multi-Agent Workflow
- Manual Multi-Agent Pipeline
- CLI Agent Pipeline with HITL Gates
- 人工編排式多 Agent 協作

問題在於，使用者仍負責所有跨角色協調。從系統角度看，人同時承擔了五個尚未被程式化的元件：

| 人工職責 | 系統化後的對應能力 |
|---|---|
| 記住目前進度 | Current State |
| 複製上一階段結論 | Artifact Reference |
| 告訴下一個 Agent 做什麼 | Structured Handoff |
| 判斷應回到哪一階段 | State Machine / Router |
| 執行 Commit、Push、Deploy | HITL Approval Gate |

因此，第一個核心判斷是：

> 瓶頸通常在於缺少共享、可驗證、可持久化的工作狀態。

## 2. 為什麼 Skill 與短 Prompt 仍無法徹底解決？

Skill、角色文件與 Prompt Template 非常重要，它們能保存：

- 角色責任。
- 編碼規範。
- Review 格式。
- 執行步驟。
- 安全限制。
- 專案慣例。

但這些資產偏向「靜態操作規則」，不會自動保存「本次任務的動態事實」。

例如，Reviewer 的 Skill 可以規定 Blocker／Major／Minor，但它不會自動知道：

- Implementer 本次實際改了哪些檔案。
- 哪一條 Requirement 已完成。
- 哪一個測試失敗。
- 目前 Diff 對應哪個規格版本。
- 前一輪 Review 是否已被修正。
- 本輪是否應進入 Readiness Check。

可以把兩者區分為：

```text
Skill / Rules
= 長期可複用的工作方式

Shared State / Artifact
= 本次 Run 目前成立的工作事實
```

只有 Skill，Agent 仍需要使用者重新提供本次工作上下文；只有 State，Agent 又可能不知道如何正確使用它。兩者必須配合。

## 3. Token 浪費真正發生在哪裡？

Token 浪費不只來自 Prompt 太長，還包括以下結構性重複。

### 3.1 重複敘述背景

每個 Agent 都重新讀取：

- 為什麼要改。
- 預計怎麼改。
- 不能改什麼。
- 哪些風險已接受。

### 3.2 重複轉譯結果

實作者輸出自然語言摘要，使用者再將其改寫成 Reviewer Prompt。Reviewer 的輸出又被重新整理給 Coordinator。

每一次轉譯都可能：

- 遺漏細節。
- 引入解釋偏差。
- 搞錯最新版本。
- 丟失失敗證據。

### 3.3 重複載入無關歷史

為了確保下一個 Agent 不漏上下文，使用者傾向貼入完整聊天紀錄。這會讓 Agent 重新消化大量已失效內容，而不是只讀「最新可接受狀態」。

### 3.4 無法精確快取

若每輪輸入都是不同自然語言摘要，就很難穩定識別可復用內容。結構化 Artifact 與 Reference 更容易做內容雜湊、版本辨識與局部載入。

更好的優化目標是：

> 讓每個角色只讀完成當前任務所需的最小充分事實。

## 4. 人工轉述模式的風險

### 4.1 Stale Context

使用者可能把上一輪、而非最新一輪的 Findings 貼給實作者。

### 4.2 Missing Evidence

只貼測試結論「通過」，卻沒有命令、Exit Code、範圍或未執行項目。

### 4.3 Role Leakage

Reviewer 為了補缺資料開始修改程式碼；Implementer 為了讓流程前進自行改變 Requirement。

### 4.4 Gate Bypass

Reviewer 回傳 PASS 後，流程直接進 Commit，卻沒有檢查測試、Task、Spec 與 Diff 是否一致。

### 4.5 Non-resumable Session

CLI 中斷後，下一次無法從可靠狀態恢復，只能重新閱讀聊天紀錄。

### 4.6 No Audit Boundary

無法清楚說明：哪一個角色、基於哪份輸入、在什麼狀態下產生了哪個結論。

## 5. 多 Agent Coding Workflow 成熟度模型

成熟度描述的是一條演進路徑：從人工協作，逐步走向可持久化、可程式化與可治理的系統。

## Level 0：單 Agent Session

```text
User → Agent → Output
```

特徵：

- 一個 Agent 同時規劃、實作與自我檢查。
- 沒有角色隔離。
- 沒有跨 Session 狀態。

適合：

- 短小、低風險、一次性任務。

主要限制：

- 自我審查偏差。
- 長任務容易遺失上下文。

## Level 1：人工編排多 Agent

```text
Planner → Human → Implementer → Human → Reviewer
```

特徵：

- 角色已分離。
- 人負責所有交接。
- Git 與高風險操作由人控制。

價值：

- 已能獲得不同模型與角色的互補。
- 容易建立，不需新增 Runtime。

限制：

- 人是單點瓶頸。
- 上下文無法直接互通。
- Token 與操作成本隨任務輪次線性增加。

## Level 2：Artifact-driven Workflow

```text
Agent
  ↕
Current State + Artifact References
  ↕
Agent
```

新增能力：

- Agent Result Schema。
- Handoff Envelope。
- Current State Schema。
- 明確 State Machine。
- Artifact Reference。
- Review Evidence。
- HITL Gate。

此階段可以仍由人手工啟動每個 Agent，但不再需要搬運完整內容。人只需提供：

```text
changeId
runId
currentStateRef
handoffId
```

## Level 3：Durable Workflow

新增能力：

- Checkpoint。
- Pause／Resume。
- Retry。
- Idempotency。
- Artifact Versioning。
- Runtime／Trace 分離。
- 故障恢復。

LangGraph 的 Checkpointer 就是公開案例之一：它保存圖狀態快照，用於 HITL、故障恢復與時間旅行等能力。

## Level 4：Programmatic Orchestration

新增一個 Node.js、Python 或 Workflow Engine Orchestrator：

```text
Load State
→ Validate Preconditions
→ Invoke Agent
→ Capture Result
→ Validate Schema
→ Persist Artifact
→ Apply Transition
→ Pause at HITL Gate
```

此時程式負責流程，LLM 負責節點內的專業判斷。

## Level 5：Production-grade Agent Platform

進一步加入：

- Queue 與工作租約。
- 併發控制。
- Durable Store。
- Trace、Metrics、Cost Governance。
- Secret Management。
- Policy Engine。
- Eval 與回歸測試。
- Incident Recovery。
- Artifact Retention。
- Release／Rollback。

這一層的目標是讓系統可安全營運，而非只讓 Agent 自動跑。

## 6. Node.js 腳本何時算 Orchestrator？

### 只算 Pipeline 的情況

```js
await runPlanner();
await runImplementer();
await runReviewer();
```

這段程式雖然自動呼叫三個 Agent，但缺少：

- 狀態驗證。
- 輸出契約。
- 分支。
- 錯誤回流。
- 人工批准。
- 恢復能力。

它仍屬於自動化 Pipeline，還不到完整的 Multi-Agent Workflow Runtime。

### 接近 Orchestrator 的情況

腳本至少開始處理：

```text
review.verdict == REQUEST_CHANGES
    → 回 Implementer

review.verdict == INCOMPLETE
    → 補 Evidence 或人工處理

review.designConflict == true
    → Coordinator Arbitration

allGatesPassed == true
    → Pause for Human Commit Approval
```

並且能確保：

- 同一輸入不會重複造成副作用。
- Schema 不合法時不推進狀態。
- Agent 失敗後可以從 Checkpoint 繼續。
- 不可逆操作永遠受 Gate 保護。

## 7. 推薦的成功指標

不要只用「Agent 自動跑了幾輪」評估。更有價值的指標包括：

### 交接效率

- 每次 Handoff 需要人工貼入的字數。
- 下一個角色啟動所需操作數。
- 重複背景 Token 比例。

### 正確性

- 因過期 Artifact 造成的錯誤次數。
- Schema 驗證失敗率。
- Findings 被錯誤遺漏的比例。
- 非法 State Transition 次數。

### 恢復能力

- CLI 中斷後恢復所需時間。
- 重試是否重複寫檔或產生副作用。
- 是否能指出最新可接受 Artifact。

### 人工控制

- Commit／Push／Deploy 是否仍有清楚的批准紀錄。
- 人工是否只處理決策，而不是複製貼上。

## 8. 反指標

以下現象不代表成熟：

- Agent 數量很多。
- Prompt 很長。
- 每一輪產生大量 Markdown。
- 所有操作都自動化。
- Reviewer 也能直接修改程式碼。
- LLM 可以自行決定 Commit 或 Deploy。
- 保存所有對話就宣稱具有 Memory。

真正重要的是：

- 狀態是否清楚。
- 輸入是否最小且可追溯。
- 輸出是否可驗證。
- 角色是否守住權限邊界。
- 失敗是否可以恢復。

## 9. 最小升級判斷

當以下任三項成立，就值得從 Level 1 升級到 Level 2：

- 每個任務至少經過三個角色。
- Review 通常有一輪以上修正回路。
- 使用者頻繁貼入 Diff、測試結果與 Findings。
- CLI Session 中斷後很難恢復。
- 多個 Change 同時進行，容易混淆上下文。
- 準備撰寫自動 Orchestrator。
- 需要證明某次批准基於哪些 Evidence。

## 10. 小結

人工編排是合理的第一階段。它能先驗證角色分工與流程是否有價值。

但當流程穩定後，下一步不應立即追求全自動，而應先將人的隱性工作顯性化：

```text
人的記憶
→ Current State

人的剪貼簿
→ Artifact Reference

人的交接說明
→ Structured Handoff

人的流程判斷
→ State Machine

人的高風險批准
→ HITL Gate
```

這就是從「使用多個 Agent」走向「建立多 Agent Workflow」的分水嶺。

## 參考資料

- LangGraph：Thinking in LangGraph  
  <https://docs.langchain.com/oss/javascript/langgraph/thinking-in-langgraph>
- LangGraph：Persistence  
  <https://docs.langchain.com/oss/javascript/langgraph/persistence>
- OpenAI Agents SDK：Agent Orchestration  
  <https://openai.github.io/openai-agents-js/guides/multi-agent/>
- OpenAI Agents SDK：Human-in-the-loop  
  <https://openai.github.io/openai-agents-js/guides/human-in-the-loop/>
