# 04｜分階段導入路線：從人工交接到 Node Orchestrator

## 1. 導入原則

最穩定的導入順序是：

```text
先觀察人工流程
→ 固化角色與 Gate
→ 定義資料契約
→ 分離儲存生命週期
→ 最後自動編排
```

原因是：

> 自動化只會放大既有流程。若角色、輸入、輸出與失敗路由仍不清楚，Orchestrator 只會把混亂跑得更快。

本章提供五個導入階段。每一階段都有明確 Scope、完成標準與不要提前做的事情。

## 2. Phase 0：建立人工流程基線

### 目標

先確認多角色分工確實有價值，並觀察真實 Handoff 內容。

### 建議角色

- Coordinator / Planner
- Implementer
- Reviewer
- Human Approver

### 需要記錄

每次人工交接實際搬運了什麼：

- 規格摘要。
- Scope。
- 修改檔案。
- Diff。
- Validation。
- Findings。
- Verdict。
- 下一步。

### 完成標準

- 每個角色責任穩定。
- Reviewer 是否唯讀已決定。
- 哪些操作必須 HITL 已決定。
- 流程至少成功跑過數次。
- 能指出最常重複複製的資料。

### 不要做

- 不要急著導入 Queue。
- 不要建立十幾種 Agent。
- 不要將所有聊天永久保存。
- 不要允許多 Agent 同時寫同一工作樹。

## 3. Phase 1：Contract Layer

### 目標

讓不同 Agent 能用相同資料語言工作，即使仍由人手工啟動。

### 改造內容

1. Agent Result Schema。
2. Handoff Envelope Schema。
3. Current State Schema。
4. State Transition Table。
5. Artifact Reference Convention。
6. Reviewer Read-only Contract。
7. HITL Git Gate。
8. Execution Summary Template。
9. Git-ignored Runtime Directory。

### 最小目錄

```text
.agent-runtime/<change-id>/
├─ current-state.json
├─ artifacts/
└─ evidence/
```

### 第一版可不安裝任何套件

可以先透過：

- JSON Schema 文件。
- CLI 原生 Structured Output。
- 人工檢查。
- 簡單 JSON Parse。

驗證流程。

等到 Orchestrator 階段，再引入 Ajv、Zod 或其他 Runtime Validator。

### 完成標準

- 下一個 Agent 不需要使用者貼入長摘要。
- 只靠 Change ID、Current State 與 Artifact Refs 即可開始。
- Schema 不合法時不允許交接。
- Reviewer 不需要寫入權限。
- Commit／Push 仍由人工控制。
- Runtime Artifact 不進 Git。

### Dry Run

建議挑一個中等規模、低風險 Change：

```text
Coordinator
→ coordinator_result
→ Implementer
→ implementation_result + evidence
→ Reviewer
→ review_result
→ Coordinator readiness
→ Human commit
```

衡量人工貼入內容是否明顯下降。

## 4. Phase 2：Storage and Lifecycle Separation

### 目標

讓 State、Evidence、Trace 與 Durable Knowledge 有清楚邊界。

### 改造內容

- Repo／Runtime／Trace 分離。
- Latest Accepted Artifact 概念。
- Artifact Version 與 Checksum。
- Retention Policy。
- Secret Redaction。
- Cross-run Isolation。
- Runtime Cleanup。
- Failure Recovery Strategy。

### 完成標準

- Git 不會隨 Agent Retry 無限增加文件。
- Current State 不含大型 Diff 或完整 Trace。
- Artifact 可由 ID、URI、Version、Checksum 追蹤。
- 能清理 Trace 而不破壞 Durable Knowledge。
- 能區分最新產生與最新可接受版本。

### 可選升級

- CI Artifact Store。
- SQLite。
- Object Storage。
- Checkpointer。

選擇應由使用場景驅動；外觀完整不是目標。

## 5. Phase 3：Node.js Orchestrator

### 目標

把已穩定的人工狀態流轉機械化。

### 建議工作流結構

```text
tools/agent-orchestrator/
├─ src/
│  ├─ state/
│  ├─ contracts/
│  ├─ adapters/
│  ├─ routing/
│  ├─ approvals/
│  └─ cli/
├─ tests/
└─ package.json
```

### 核心責任

```text
1. Load Current State
2. Validate State Schema
3. Check Transition Preconditions
4. Build Handoff
5. Invoke Agent Adapter
6. Capture Structured Result
7. Validate Result Schema
8. Persist Artifact and Evidence
9. Apply Transition
10. Pause at Human Gate
11. Resume after Approval
```

### Adapter 抽象

不要把 Orchestrator 綁死某個 CLI：

```ts
interface AgentAdapter {
  invoke(input: AgentInvocation): Promise<AgentInvocationResult>;
}
```

不同工具只需各自處理：

- 命令列參數。
- Structured Output。
- stdout／stderr。
- Exit Code。
- Timeout。
- Sandbox／Permission。

### Validator

Node.js 可使用：

- Ajv：偏 JSON Schema 標準與跨語言契約。
- Zod：偏 TypeScript 型別與程式內使用。

若公開契約需要被不同語言、CLI 與平台共用，JSON Schema 通常更適合作為 Canonical Contract；TypeScript 可以從 Schema 生成型別，或在程式內建立 Adapter。

### 狀態推進規則

Orchestrator 不應盲目相信 Agent 的 `nextHandoff`。它應：

1. 驗證 Agent 是否有權提出該 Transition。
2. 檢查 Gate 是否滿足。
3. 驗證 Artifact Reference。
4. 更新 State。
5. 記錄 Transition 結果。

### 完成標準

- 能自動完成 Plan → Implement → Review 的正常路徑。
- 能處理 REQUEST_CHANGES 回路。
- 能處理 INCOMPLETE。
- 能將 Design Conflict 升級給 Coordinator。
- 能在 Commit Gate 暫停。
- 重新執行不會重複產生不安全副作用。

## 6. Phase 4：Production Hardening

### 目標

讓系統可安全支援多使用者、多 Run 與長時間執行。

### 能力

- Durable Store。
- Queue。
- Lease／Lock。
- Concurrency Control。
- Timeout／Retry Policy。
- Dead-letter／Manual Recovery。
- Trace、Metrics、Alert。
- Cost Budget。
- Policy Engine。
- Evaluation。
- Release／Rollback。

### 何時才需要？

當出現以下需求：

- 多個 Change 併發。
- 多人共用同一 Agent Platform。
- Agent 執行超過單一程序生命週期。
- 需要嚴格稽核。
- 需要 SLA。
- 需要對 Token、延遲與成功率做治理。

## 7. 推薦的施工切片

Phase 1 可以拆成以下階段性 Commit，由人執行 Commit：

```text
Commit 1
docs: define artifact shared state and handoff contracts

Commit 2
feat: add agent result, handoff, and current state schemas

Commit 3
refactor: route agent stages through artifact references

Commit 4
chore: define local runtime boundary and gitignore rules

Commit 5
test: validate artifact handoff with a dry run
```

Phase 3 再拆：

```text
Commit 1
feat(orchestrator): add contract validation and state repository

Commit 2
feat(orchestrator): add CLI agent adapters

Commit 3
feat(orchestrator): add workflow routing and retry loop

Commit 4
feat(orchestrator): add human approval pause and resume

Commit 5
test(orchestrator): cover failure and recovery paths
```

## 8. Rollout 策略

不要一次切換所有任務。

### Shadow Mode

Orchestrator 只計算下一步，不實際呼叫 Agent。與人工判斷比較。

### Assisted Mode

Orchestrator 建立 Handoff 與命令，使用者確認後執行。

### Semi-automatic Mode

自動執行可逆階段，在 Review、Commit、Push、Deploy 暫停。

### Controlled Automatic Mode

對低風險任務自動流轉，高風險操作仍 HITL。

## 9. 回退策略

自動編排出現問題時，應能退回人工模式：

- Current State 可被人閱讀。
- Artifact 是標準 JSON／檔案，不只存在框架內部。
- Handoff 可轉成可讀摘要。
- CLI Adapter 可單獨執行。
- Durable Knowledge 不依賴 Runtime 才能理解。

這正是採用開放契約的價值：流程資料不被封閉框架狀態鎖住。

## 10. 量化導入效果

建議導入前後比較：

| 指標 | 目標方向 |
|---|---|
| 每次 Handoff 人工貼入字數 | 降低 |
| 每個 Change 人工操作次數 | 降低 |
| 過期 Context 造成的返工 | 降低 |
| Schema Error 被提前發現比例 | 提高 |
| CLI 中斷後恢復時間 | 降低 |
| Reviewer Evidence 完整率 | 提高 |
| 不可逆操作人工覆核率 | 維持 100% 或依政策 |

不要把「完全無人操作」當作唯一 KPI。對 Coding Workflow 而言，可靠性與可追溯性往往比自治程度更重要。

## 11. 常見提前過度設計

### 一開始就上完整 Event Sourcing

若尚未有穩定 Event Model，只會增加遷移與 Replay 成本。

### 一開始就上分散式 Queue

單人、單工作樹流程通常沒有必要。

### 讓 LLM 自由生成所有 Transition

容易出現未知狀態、跳過 Gate 與不可重現路由。

### Reviewer 直接修程式碼

破壞角色獨立與 Review Evidence。

### Runtime 全部提交 Git

會造成文件爆炸與敏感資訊風險。

### Orchestrator 綁死單一 CLI

工具升級或替換時，整個流程都必須重寫。

## 12. 導入決策樹

```text
是否已有穩定多角色流程？
├─ 否 → 先做 Phase 0
└─ 是
   ↓
是否仍靠人工貼入摘要／Diff／Findings？
├─ 是 → 做 Phase 1
└─ 否
   ↓
是否出現文件爆炸、跨 Run、恢復與保留需求？
├─ 是 → 做 Phase 2
└─ 否
   ↓
流程契約是否已穩定且重複執行？
├─ 是 → 做 Phase 3
└─ 否 → 繼續人工 Dry Run
   ↓
是否有多人、多機、SLA、併發與稽核需求？
├─ 是 → 做 Phase 4
└─ 否 → 保持輕量
```

## 13. 參考資料

- OpenAI Agents SDK：Agent Orchestration  
  <https://openai.github.io/openai-agents-js/guides/multi-agent/>
- OpenAI Agents SDK：Human-in-the-loop  
  <https://openai.github.io/openai-agents-js/guides/human-in-the-loop/>
- LangGraph：Overview / Persistence  
  <https://docs.langchain.com/oss/javascript/langgraph/overview>  
  <https://docs.langchain.com/oss/javascript/langgraph/persistence>
- Enterprise Integration Patterns：Process Manager  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html>
