# 設計模式映射：多 Agent Shared State 與 Handoff 並不是全新問題

## 1. 不是單一 GoF Pattern

Artifact-based Shared State + Structured Handoff 並沒有一個完全一對一的 GoF 設計模式。

它更像是將多個既有架構模式組合，用來處理：

- 多個專業角色共享問題狀態。
- 工作所有權轉移。
- 長流程路由與失敗回路。
- 狀態持久化。
- 大型 Artifact 引用。
- 歷史、快照與查詢效率。
- 人工批准與不可逆操作。

因此，較合理的做法是理解每個 Pattern 在整體架構中的責任，不必強行宣稱「它就是某個 Pattern」。

## 2. 模式對照總表

| 多 Agent 元件 | 對應模式 | 解決問題 |
|---|---|---|
| Shared State | Blackboard | 多個專家基於共同問題空間工作 |
| State Store Interface | Repository | 隔離 Domain 與 JSON／DB／Checkpointer |
| Handoff Envelope | Command Message / Envelope Wrapper | 將工作交接包成明確訊息 |
| Artifact Reference | Claim Check | 傳引用而非大型內容 |
| Workflow Routing | State Machine | 定義合法階段與 Transition |
| Planner → Coder → Reviewer | Pipes and Filters | 每階段消費 Artifact 並產生新 Artifact |
| Orchestrator | Process Manager | 根據中間結果決定下一個步驟 |
| Complete History | Event Sourcing | 以事件保存完整狀態變更歷史 |
| Current State | Snapshot / Materialized View | 提供高效、最新的操作視圖 |
| Retry-safe Consumer | Idempotent Receiver | 避免重試重複產生副作用 |
| Human Approval | Approval Gate / Interrupt-Resume | 在高風險操作前暫停 |

## 3. Blackboard Pattern

### 核心思想

多個 Knowledge Sources 不必互相直接知道對方，而是透過共同 Blackboard 讀取目前狀態並寫入新事實。

```text
Planner ───────┐
Implementer ───┼── Shared Blackboard
Reviewer ──────┘
```

映射到多 Agent Coding Workflow：

- Planner 寫入核准 Scope 與 Acceptance Criteria。
- Implementer 寫入 Implementation Result 與 Evidence。
- Reviewer 寫入 Findings 與 Verdict。
- Coordinator 根據 Blackboard 做仲裁與 Readiness。

### 適用價值

- 降低 Agent 之間點對點耦合。
- 允許角色由不同模型或工具實作。
- 共同狀態可以被人與程式檢查。

### 風險

- 所有人都能任意寫 Blackboard 時，容易污染狀態。
- Blackboard 變成無結構大雜燴。
- 缺少 Control Plane 時，不知道何時輪到誰。

因此需要搭配：

- Schema。
- Ownership。
- State Machine。
- Structured Handoff。

## 4. Repository Pattern

Martin Fowler 對 Repository 的描述是：在 Domain 與資料映射層之間提供類似集合的介面，隔離底層持久化細節。

多 Agent Workflow 可以定義：

```ts
interface WorkflowStateRepository {
  getCurrentState(changeId: string): Promise<CurrentState>;
  saveArtifact(artifact: AgentArtifact): Promise<ArtifactRef>;
  updateState(expectedVersion: number, next: CurrentState): Promise<void>;
}
```

第一版底層可以是：

- 本地 JSON。

未來可以替換成：

- SQLite。
- PostgreSQL。
- Redis。
- Object Storage Metadata。
- LangGraph Checkpointer。

Agent 與 Routing Logic 不需要知道儲存細節。

### 適用價值

- 支援漸進式演進。
- 易於測試。
- 避免 Orchestrator 與檔案路徑緊耦合。

### 風險

- 過度泛化會產生複雜抽象。
- 不同 Store 的交易與一致性能力不同，不能假裝完全等價。

## 5. Command Message 與 Envelope Wrapper

Structured Handoff 接近 Command Message：

> 請指定角色，基於指定輸入，執行指定階段，產生指定結果。

Envelope 則包含：

- Routing Metadata。
- Correlation ID。
- Schema Version。
- Security Context。
- Required References。
- Expected Output。

真正的工作內容放在被引用的 Artifact 中。

### 適用價值

- Handoff 可被驗證。
- 容易建立 Adapter。
- 可加入 Trace、Timeout、Retry 等 metadata。

### 風險

- 把完整 Artifact 重複塞進 Envelope，會失去分離價值。
- Envelope 欄位過多時會變成另一份 Current State。

## 6. Claim Check Pattern

Claim Check Pattern 的精神是：

- 大型或敏感 Payload 放在專門 Store。
- 流程訊息只攜帶可受控的引用。

映射：

```text
Handoff
  └─ diffRef: runtime://evidence/git-diff.patch
```

而不是：

```text
Handoff
  └─ diff: "完整十萬字 patch"
```

### 適用價值

- 降低 Token 與訊息大小。
- 可獨立設定 Artifact 權限與保留期限。
- 容易使用 Checksum 驗證。

### 風險

- Reference 失效。
- Path Traversal。
- Artifact Store 權限錯誤。
- Reference 與 State 版本不一致。

## 7. State Machine

State Machine 是 Structured Handoff 的秩序來源。

```text
READY_FOR_REVIEW
    ↓ review.requested
REVIEWING
    ├─ approve → READY_FOR_READINESS_CHECK
    ├─ changes → CHANGES_REQUESTED
    └─ conflict → NEEDS_COORDINATOR_ARBITRATION
```

### 適用價值

- 非法 Transition 可被拒絕。
- 容易測試。
- 流程可視化。
- 不依賴模型自由生成下一步。

### 風險

- 狀態數量無控制地膨脹。
- 將所有細節都設計成 State，造成複雜度。

建議只把影響路由、權限與恢復的狀態顯性化。

## 8. Pipes and Filters

規劃、實作、審查都可視為 Filter：

```text
Requirement
→ Plan Filter
→ Implementation Filter
→ Review Filter
→ Readiness Filter
```

每個 Filter：

- 接受明確輸入。
- 進行專業轉換。
- 產生 Artifact。

### 適用價值

- 角色可替換。
- 每階段可獨立測試。
- 容易新增 Validation Filter。

### 限制

真正 Coding Workflow 通常不是純線性：

```text
Reviewer → Implementer → Reviewer
```

因此需要搭配 State Machine 與 Process Manager。

## 9. Process Manager

Enterprise Integration Patterns 將 Process Manager 描述為：中央處理單元保存流程狀態，並根據中間結果決定下一步。

這幾乎就是 Node Orchestrator 的責任：

- 維持 Current State。
- 呼叫下一個 Agent。
- 根據 Result 路由。
- 管理 Retry 與 Human Gate。

### 適用價值

- 支援非線性流程。
- 支援條件分支與平行節點。
- 集中管理流程安全。

### 風險

- 中央瓶頸。
- 變成什麼都管的 God Object。
- 所有 Domain Logic 被塞進 Orchestrator。

控制方式：

- Orchestrator 只管理流程，不承擔 Agent 專業判斷。
- 將 Storage、Adapter、Policy、Validation 分層。
- 保持 State Contract 穩定。

## 10. Event Sourcing

Event Sourcing 以 append-only Event 保存完整變更歷史：

```text
PlanApproved
ImplementationCompleted
ReviewRequested
ChangesRequested
ReviewApproved
HumanCommitApproved
```

### 適用價值

- 完整稽核。
- 可重建 State。
- 容易分析流程歷史。

### 風險

- Event Versioning。
- Replay 成本。
- Ordering。
- Idempotency。
- Snapshot 與 Migration。

第一版通常不需要完整導入。可先使用：

```text
Current State + Latest Artifact + Important Approval Record
```

## 11. Snapshot / Materialized View

Current State 可以視為完整歷史的操作快照：

```text
Full Events / Artifacts
          ↓ projection
Current State
```

Agent 正常執行讀 Snapshot，而不是每次重播全部歷史。

### 適用價值

- 降低讀取與 Token 成本。
- 清楚識別目前可接受版本。
- 快速恢復。

### 風險

- Snapshot 與來源不一致。
- 更新失敗造成 Stale State。

控制方式：

- State Version。
- Atomic Update。
- Checksum。
- 必要時可從事件重建。

## 12. Idempotent Receiver

Agent 或 CLI 可能因 Timeout 被重試。若同一 Result 被處理兩次，不能重複：

- 修改 State。
- 建立相同 Artifact。
- 發起 Commit。
- 觸發 Deploy。

可以使用：

- `runId`
- `handoffId`
- `artifactId`
- `idempotencyKey`
- Expected State Version

判斷是否已處理。

## 13. 模式組合

完整架構可以表示為：

```text
                         ┌──────────────────┐
                         │ Durable Knowledge│
                         └────────┬─────────┘
                                  │
                                  ▼
┌───────────┐   Command/Handoff  ┌────────────────┐
│   Agent   │ ◄────────────────► │ Process Manager│
└─────┬─────┘                    └───────┬────────┘
      │                                  │
      │ Artifact                         │ State Transition
      ▼                                  ▼
┌────────────────────────────────────────────┐
│ Blackboard / Shared State                  │
│ Current Snapshot + Artifact References     │
└───────────────────┬────────────────────────┘
                    │ Repository
                    ▼
       ┌────────────────────────────┐
       │ Runtime / Artifact Storage │
       └────────────────────────────┘
```

模式責任：

- Blackboard：共享問題空間。
- Repository：隔離儲存。
- Handoff／Command：傳遞工作。
- State Machine：限制流程。
- Process Manager：執行路由。
- Claim Check：引用大型 Artifact。
- Snapshot：高效讀取當前狀態。
- HITL Gate：保護不可逆操作。

## 14. 不要過度模式化

設計模式是幫助思考，不是要求第一版實作所有元件。

最小可用組合：

```text
Blackboard-like Current State
+ Structured Command/Handoff
+ Explicit State Machine
+ Local Repository Adapter
+ Human Approval Gate
```

只有當需求出現時，再增加：

- Process Manager 自動化。
- Event Sourcing。
- Queue。
- Distributed Lock。
- Full Trace Platform。

## 15. 參考資料

- Martin Fowler：Repository  
  <https://martinfowler.com/eaaCatalog/repository.html>
- Enterprise Integration Patterns：Process Manager  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html>
- Enterprise Integration Patterns：Message / Pipes and Filters / Claim Check  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/Message.html>  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/PipesAndFilters.html>  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/StoreInLibrary.html>
- Azure Architecture Center：Event Sourcing / Materialized View / Claim Check  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing>  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view>  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check>
