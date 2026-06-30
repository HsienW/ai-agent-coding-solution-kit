# 02｜Artifact-based Shared State + Structured Handoff 參考架構

[English](./02-reference-architecture.md) | [繁體中文](./02-reference-architecture-zh-TW.md)

## 1. 先釐清兩者的分工

Artifact-based Shared State 與 Structured Handoff 解決不同問題：

| 機制 | 核心問題 | 架構角色 |
|---|---|---|
| Artifact-based Shared State | 系統目前相信哪些事實？ | Data Plane |
| Structured Handoff | 工作應交給誰、讀什麼、輸出什麼？ | Control Plane |

只做 Shared State，Agent 可能知道「現在有什麼」，卻不知道「下一步由誰負責」。

只做 Handoff，Agent 可能知道「交給誰」，卻缺少可靠輸入與版本依據。

因此完整方案是：

> Artifact-based Shared State with Structured Handoff

## 2. 四種資料不要混在一起

在實作前，先區分四個容易混淆的名詞。

## 2.1 Durable Knowledge

長期有效、應讓未來維護者理解的工程知識，例如：

- Requirements。
- Design。
- Acceptance Criteria。
- Task Plan。
- ADR／Decision。
- 最終 Execution Summary。

它通常進 Git，生命週期跟隨專案。

## 2.2 Runtime State

本次 Run 的目前快照，例如：

- Current phase。
- Current owner。
- Attempt。
- Gate status。
- Latest artifact refs。
- Next action。

它是操作狀態，不等於永久文件。

## 2.3 Artifact

Agent 在某個階段產生的具體交付結果，例如：

- Coordinator Result。
- Implementation Result。
- Review Result。
- Test Evidence。
- Diff Patch。
- Readiness Result。

A2A Protocol 也明確區分 Message 與 Artifact：Message 是溝通內容，Artifact 是 Task 處理後的具體交付物。

## 2.4 Trace

用於除錯、稽核與觀測的完整執行資訊，例如：

- 模型請求與回應 metadata。
- Token、Latency、Retry。
- Tool calls。
- stdout／stderr。
- 詳細事件序列。

Trace 不應直接成為 Agent 每輪的正常輸入，也不應全部提交到 Git。

## 3. 參考角色模型

以下角色名稱保持模型與工具中立。

| 角色 | 主要責任 | 寫入權限 |
|---|---|---|
| Coordinator / Planner | 建立與仲裁規格，判斷流程是否可進入下一階段 | 規格與協調結果 |
| Implementer | 根據核准規格修改程式碼並產生驗證證據 | 程式碼與 Implementation Result |
| Reviewer | 唯讀檢查規格、Diff、Evidence 與風險 | 只輸出 Review Result |
| Orchestrator | 驗證契約、更新狀態、執行 Transition、Pause／Resume | Runtime State |
| Human Approver | 批准不可逆或高風險操作 | Approval Decision |

在沒有 Orchestrator 的第一版，人仍可手工啟動各角色，但應沿用相同契約。

## 4. Current State：共享狀態快照

Current State 表示「系統現在應如何被理解」，並不保存完整歷史。

概念範例如下：

```json
{
  "schemaVersion": "1.0",
  "changeId": "feature-example",
  "runId": "run-20260630-001",
  "phase": "READY_FOR_REVIEW",
  "currentOwner": "reviewer",
  "attempt": 1,
  "latestArtifacts": {
    "plan": "repo://specs/feature-example/design.md",
    "implementation": "runtime://artifacts/implementation-result.json",
    "diff": "runtime://evidence/git-diff.patch",
    "validation": "runtime://evidence/validation-summary.json"
  },
  "gates": {
    "planApproved": true,
    "implementationCompleted": true,
    "validationPassed": true,
    "reviewApproved": false,
    "humanCommitApproved": false
  },
  "blockers": [],
  "nextAction": "REQUEST_REVIEW"
}
```

### Current State 應保存什麼？

- 跨節點必須持續存在的事實。
- 最新可接受 Artifact 的 Reference。
- 合法路由所需的 Gate。
- Retry 與 Recovery 所需識別資訊。

### 不應保存什麼？

- 完整聊天紀錄。
- Agent 私有推理。
- 所有歷史版本全文。
- 可從 Artifact 推導的冗長格式化描述。
- Secret 或 Credential。

LangGraph 的建議同樣是把需要跨步驟持續存在的資料放入 State，並盡量保存原始、可重用資訊，而不是只保存特定輸出格式。

## 5. Agent Result Envelope：統一交付契約

不同角色可以有不同 Payload，但共同欄位應一致。

```json
{
  "schemaVersion": "1.0",
  "artifactId": "artifact-implementation-001",
  "changeId": "feature-example",
  "runId": "run-20260630-001",
  "producer": "implementer",
  "stage": "apply-change",
  "kind": "implementation_result",
  "status": "completed",
  "summary": "Implemented approved scope and executed validation.",
  "inputRefs": [],
  "outputRefs": [],
  "verification": [],
  "risks": [],
  "nextHandoff": {}
}
```

建議採共同 Envelope 加上 discriminated union：

```text
kind = coordinator_result
kind = implementation_result
kind = review_result
kind = readiness_result
kind = archive_result
```

這樣能同時獲得：

- 共用解析流程。
- 清楚的角色專屬欄位。
- 較低的 Schema 重複。
- 容易增加版本與新角色。

## 6. Structured Handoff：下一棒契約

Handoff 傳遞的是 Reference 與工作契約，不複製完整 Artifact。

```json
{
  "schemaVersion": "1.0",
  "handoffId": "handoff-003",
  "changeId": "feature-example",
  "runId": "run-20260630-001",
  "from": "implementer",
  "to": "reviewer",
  "stage": "review-result",
  "reason": "implementation_completed",
  "requiredInputRefs": [
    "repo://specs/feature-example/design.md",
    "runtime://artifacts/implementation-result.json",
    "runtime://evidence/git-diff.patch",
    "runtime://evidence/validation-summary.json"
  ],
  "expectedOutputKind": "review_result",
  "acceptanceCriteriaRefs": [
    "repo://specs/feature-example/requirements.md#acceptance"
  ],
  "onSuccess": "READY_FOR_READINESS_CHECK",
  "onFailure": "CHANGES_REQUESTED",
  "onConflict": "NEEDS_COORDINATOR_ARBITRATION",
  "requiresHumanApproval": false
}
```

### Handoff 的最小責任

1. 識別交接雙方。
2. 描述交接原因。
3. 列出必要輸入。
4. 定義預期輸出種類。
5. 提供成功、失敗與衝突路由。
6. 標示是否需要人工批准。

### Handoff 不應負責

- 保存完整 Diff。
- 保存完整規格。
- 保存全部 Review Findings。
- 取代 Current State。
- 直接授權不可逆操作。

## 7. Artifact Reference：傳引用，不傳全文

Reference 可以採 URI-like 格式：

```text
repo://specs/feature/design.md
runtime://artifacts/review-result.json
runtime://evidence/git-diff.patch
ci://runs/12345/test-report
object://agent-artifacts/run-001/diff.patch
```

每個 Reference 最好搭配：

- `artifactId`
- `version`
- `mediaType`
- `sha256` 或其他內容雜湊
- `createdAt`
- `producer`
- `scope`

這個設計與 Claim Check Pattern 有相似精神：大型內容放在專門儲存位置，流程訊息只攜帶可受控的引用，降低訊息體積與重複複製。

## 8. State Machine：誰可以在什麼條件下前進

建議第一版採顯式狀態，避免讓 LLM 自由生成下一個階段名稱。

```text
PLAN_DRAFT
    ↓
PLAN_REVIEW
    ├─ approved ───────────────→ PLAN_APPROVED
    └─ changes requested ──────→ PLAN_DRAFT

PLAN_APPROVED
    ↓
READY_FOR_IMPLEMENTATION
    ↓
IMPLEMENTING
    ├─ completed ──────────────→ READY_FOR_REVIEW
    ├─ design conflict ────────→ NEEDS_COORDINATOR_ARBITRATION
    └─ failed ─────────────────→ FAILED

READY_FOR_REVIEW
    ↓
REVIEWING
    ├─ approved ───────────────→ READY_FOR_READINESS_CHECK
    ├─ changes requested ──────→ CHANGES_REQUESTED
    ├─ incomplete evidence ────→ INCOMPLETE
    └─ design conflict ────────→ NEEDS_COORDINATOR_ARBITRATION

CHANGES_REQUESTED
    ↓
IMPLEMENTING

READY_FOR_READINESS_CHECK
    ├─ ready ──────────────────→ READY_FOR_ARCHIVE
    └─ not ready ──────────────→ IMPLEMENTING / REVIEWING

READY_FOR_ARCHIVE
    ↓
ARCHIVED_AWAITING_HUMAN_COMMIT
    ├─ human approved ─────────→ COMPLETED
    └─ human rejected ─────────→ READY_FOR_ARCHIVE
```

### 必須寫成 Invariant 的限制

- 測試失敗不得進入 READY_FOR_ARCHIVE。
- 未處理 Blocker／Major 不得標記 Review Approved。
- Reviewer 不得自行修改 Requirement。
- Implementer 不得自行把 Scope Deviation 視為已核准。
- Schema 驗證失敗不得推進 State。
- Commit、Push、Deploy 等不可逆操作不得由一般 Agent Verdict 直接觸發。

## 9. 誰可以更新 Current State？

這是容易被忽略的所有權問題。

### 無 Orchestrator 的第一版

可以指定 Coordinator 或人工 Host 更新 State：

```text
Agent 輸出 Result
→ Host 驗證 Result
→ Host 保存 Artifact
→ Host 更新 Current State
```

Reviewer 即使輸出 `APPROVE`，也不直接修改 State 檔案。

### 有 Orchestrator 的版本

只有 Orchestrator 可以：

- 驗證 Result Schema。
- 保存 Artifact。
- 執行 Transition。
- 更新 Current State。

Agent 只能提出：

```text
recommendedTransition
nextHandoff
```

這能避免 Agent 同時成為「決策者」與「狀態資料庫管理者」。

## 10. Reviewer 唯讀設計

Reviewer 的可靠性不來自更多寫權限，而來自更完整、可追溯的 Evidence。

Reviewer 應讀取：

- 核准規格。
- Implementation Result。
- Changed Files。
- Git Diff。
- Validation Summary。
- 未執行項目。
- 直接相關原始碼。

Reviewer 輸出：

- Verdict。
- Findings。
- Residual Risks。
- Evidence Assessment。
- Suggested Transition。

Reviewer 不應：

- 修正程式碼。
- 直接勾選 Task。
- 直接批准 Commit。
- 為補資料而執行未授權命令。

若 Evidence 不足，Reviewer 應輸出 `INCOMPLETE`，不應猜測 `APPROVE`。

## 11. 與 SDD／OpenSpec／ADR 的關係

規格文件已經是 Shared Knowledge 的重要部分，但通常只覆蓋 Planning Layer。

```text
proposal / requirements
= 為什麼做、要達成什麼

design
= 預計如何做

tasks
= 如何拆解施工

current-state
= 現在做到哪裡

implementation-result
= 實際做了什麼

review-result
= 審查發現了什麼

execution-summary
= 最終留下哪些長期有價值的事實
```

因此，不需要用 Runtime State 取代 SDD；應該讓 Runtime State 引用穩定規格。

## 12. Code-led 與 LLM-led Orchestration 的取捨

OpenAI Agents SDK 公開文件將多 Agent 編排區分為 LLM-led 與 Code-led 等方式。Coding Workflow 通常適合混合：

### LLM 適合負責

- 規格分析。
- 程式實作。
- 語意 Review。
- Finding 分類。
- 風險說明。

### 程式適合負責

- Schema 驗證。
- 狀態轉移。
- 權限檢查。
- Retry 上限。
- Artifact 保存。
- Gate 判定。
- HITL Pause／Resume。

原因很直接：

> 專業判斷需要模型彈性；流程安全需要確定性。

## 13. 最小參考流程

```text
1. Coordinator 產生 coordinator_result
2. Host 驗證並保存 Artifact
3. Host 更新 current-state
4. 建立 Coordinator → Implementer Handoff
5. Implementer 讀取被引用的最小上下文
6. Implementer 產生 implementation_result + evidence refs
7. Host 更新 State，建立 Implementer → Reviewer Handoff
8. Reviewer 產生 review_result
9. Host 根據 Verdict 路由：
   - APPROVE → Readiness
   - REQUEST_CHANGES → Implementer
   - DESIGN_CONFLICT → Coordinator
   - INCOMPLETE → 補 Evidence／人工處理
10. Readiness 通過後 Pause at Human Commit Gate
```

這個流程即使完全由人手工啟動，也已經能消除大部分上下文搬運。

## 14. 參考資料

- LangGraph：State 與 Persistence  
  <https://docs.langchain.com/oss/javascript/langgraph/thinking-in-langgraph>  
  <https://docs.langchain.com/oss/javascript/langgraph/persistence>
- OpenAI Agents SDK：Handoffs 與 Agent Orchestration  
  <https://openai.github.io/openai-agents-js/guides/handoffs/>  
  <https://openai.github.io/openai-agents-js/guides/multi-agent/>
- A2A Protocol：Artifacts 與 Task Lifecycle  
  <https://a2a-protocol.org/latest/topics/key-concepts/>  
  <https://a2a-protocol.org/latest/topics/life-of-a-task/>
- Azure Architecture Center：Claim Check  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check>
