# 03｜Repo、Runtime State、Evidence 與 Trace 的分離

## 1. 最大疑慮：Shared State 會不會讓文件比程式碼還多？

會，前提是把每一輪 Agent 輸出都當成永久 Markdown 提交進 Git。

錯誤做法通常長這樣：

```text
execution/
├─ planner-001.md
├─ coder-001.md
├─ reviewer-001.md
├─ coder-002.md
├─ reviewer-002.md
├─ coordinator-001.md
├─ test-output-001.md
└─ trace-001.md
```

長期後果：

- 使用者不知道哪份結果最新。
- Agent 為了保險而讀取全部歷史。
- Git Diff 被 Runtime 噪音污染。
- 短期失敗紀錄成為永久維護成本。
- 規格、狀態、證據與 Debug Log 混成同一層。
- 文件數量上升，卻沒有增加可理解性。

問題出在沒有區分資料的生命週期，而不是 Shared State 本身。

## 2. 四層分離模型

推薦將資訊分成四層。

## 2.1 Durable Knowledge：長期工程知識

保存位置通常是 Git Repository。

典型內容：

- Requirement／Proposal。
- Design。
- Acceptance Criteria。
- ADR／Decision。
- Task Plan。
- 最終 Execution Summary。
- 可複用 Schema、Template 與 Prompt。

判斷原則：

> 六個月後維護者仍需要知道，才應進入 Durable Knowledge。

## 2.2 Runtime State：目前操作快照

典型內容：

- Current phase。
- Current owner。
- Gate status。
- Latest artifact refs。
- Attempt。
- Blockers。
- Next action。

第一版可以保存在：

```text
.agent-runtime/<change-id>/current-state.json
```

並加入 `.gitignore`。

## 2.3 Execution Evidence：可驗證的施工證據

典型內容：

- Changed files。
- Git Diff／Diff Stat／Diff Check。
- Test／Lint／Build 命令與 Exit Code。
- Validation Summary。
- Review Result。
- 未執行項目。

Evidence 與 Trace 不同。Evidence 是做出工程判定所需的最小證據；Trace 是完整執行觀測。

## 2.4 Trace：除錯與觀測歷史

典型內容：

- 模型 Request／Response metadata。
- Token usage。
- Latency。
- Tool invocation。
- Retry。
- stdout／stderr。
- 完整事件序列。

Trace 通常放在：

- 專用 Trace Platform。
- CI Artifact Store。
- Object Storage。
- Database。
- 本地 Git-ignored 目錄。

不放進專案文件目錄。

## 3. 建議資料矩陣

| 資料 | 正常讀者 | 是否進 Git | 建議保存期限 | 主要用途 |
|---|---|---:|---|---|
| Requirements / Design | 人與 Agent | 是 | 長期 | 產品與工程契約 |
| Current State | Orchestrator / Agent | 否 | Run 存續期間 | 流程推進 |
| Latest Agent Results | 下一個 Agent | 否 | Change 完成後短期保留 | Handoff |
| Diff / Validation Evidence | Reviewer / Coordinator | 否 | 依稽核需求 | 驗證與 Review |
| Full Trace | 平台／維運 | 否 | 依成本與合規政策 | 除錯、觀測、成本分析 |
| Execution Summary | 未來維護者 | 是 | 長期 | 最終事實沉澱 |

## 4. 最小本地目錄

第一階段不需要資料庫。可以先使用：

```text
.agent-runtime/
└─ feature-example/
   ├─ current-state.json
   ├─ artifacts/
   │  ├─ coordinator-result.json
   │  ├─ implementation-result.json
   │  ├─ review-result.json
   │  └─ handoff.json
   └─ evidence/
      ├─ changed-files.txt
      ├─ git-diff.patch
      ├─ git-diff-stat.txt
      ├─ git-diff-check.txt
      └─ validation-summary.json
```

第一版採 Latest-only 策略即可：

- `implementation-result.json` 被新一輪結果覆蓋。
- `review-result.json` 保存目前最新 Review。
- Current State 指向最新可接受版本。

若需要歷史，可先依 Run 建立子目錄，但不要一開始就導入完整 Event Sourcing。

## 5. 為什麼 Agent 平時應讀 Snapshot？

如果每次都重播完整歷史：

```text
all planner messages
+ all implementation attempts
+ all reviews
+ all test logs
```

會造成：

- Token 成本持續增加。
- 過期資訊干擾判斷。
- 恢復速度變慢。
- 難以確定哪個結果已被接受。

更好的讀取模式是：

```text
Current State Snapshot
+ Required Artifact References
+ Directly Relevant Durable Knowledge
```

只有以下情況才查完整歷史：

- 發生設計仲裁。
- 追查錯誤來源。
- 安全稽核。
- 比較不同嘗試。
- 需要重建 State。

Azure 的 Materialized View Pattern 也反映相同思想：完整來源資料不一定適合每次查詢，應預先建立適合讀取需求的視圖。Current State 可以視為工作流的 Operational View。

## 6. Event Sourcing：有價值，但不要過早導入

完整事件可能像：

```text
ChangeCreated
PlanApproved
ImplementationStarted
ImplementationCompleted
ReviewRequested
ChangesRequested
ImplementationRevised
ReviewApproved
HumanCommitApproved
```

Event Sourcing 的優點：

- 完整稽核歷史。
- 可以重建 State。
- 容易分析流程瓶頸。
- 適合需要高可靠與補償操作的系統。

但成本也很高：

- Event Schema Versioning。
- Idempotency。
- Event Ordering。
- Replay。
- Snapshot。
- Migration。
- Compensating Action。

對大多數個人或小型團隊的第一版，建議：

```text
Current State
+ Latest Artifacts
+ 少量重要事件／人工批准紀錄
```

等到確實需要跨機恢復、併發 Run 或稽核，再升級。

## 7. Artifact Version 與「最新可接受版本」

「最新產生」不一定等於「最新可接受」。

例如：

```text
implementation v1 → review approved
implementation v2 → validation failed
```

系統不應因 v2 時間較新就自動取代 v1。

建議區分：

- `createdVersion`
- `candidateVersion`
- `acceptedVersion`
- `supersedes`
- `status`

A2A Task Lifecycle 也指出，Client／Orchestrator 比服務 Agent 更適合管理 Artifact 修訂關係與選擇最新可接受版本。這個責任不應完全交給 Artifact Producer。

## 8. Reference 而非 Embed

不要把完整 5 MB Diff 放入 `current-state.json`：

```json
{
  "diff": "...millions of characters..."
}
```

應保存：

```json
{
  "diffRef": {
    "artifactId": "diff-002",
    "uri": "runtime://evidence/git-diff.patch",
    "mediaType": "text/x-diff",
    "sha256": "...",
    "status": "accepted"
  }
}
```

優點：

- State 保持精簡。
- 大型 Artifact 可獨立保存與清理。
- 容易驗證內容雜湊。
- Handoff 不需複製全文。
- 可根據權限控制不同 Artifact。

這與 Claim Check Pattern 的思路相近：訊息傳遞受控引用，內容放在合適的 Store。

## 9. 從本地到生產環境的儲存演進

## Stage A：本地 Git-ignored Files

適合：

- 單人。
- 單工作樹。
- 一次只處理少量 Change。

優點：

- 零外部依賴。
- 易於檢查。
- 適合驗證 Schema 與流程。

限制：

- 不適合多機。
- 併發與鎖定能力弱。
- 本機損壞後難恢復。

## Stage B：CI Artifact Store

適合：

- Agent 在 CI 或遠端 Runner 執行。
- Evidence 需要跟某次 Run 綁定。

可保存：

- Diff。
- Test Reports。
- Structured Results。
- Trace Bundle。

## Stage C：Database + Object Storage

適合：

- 多使用者。
- 多 Run 併發。
- 需要查詢、稽核、保留政策。

常見分工：

```text
Database
= State、Metadata、Indexes、Approval

Object Storage
= Diff、Log、Report、Large Artifact
```

## Stage D：Workflow Runtime / Checkpointer

使用 LangGraph Checkpointer、Temporal、Step Functions 或其他 Durable Workflow 機制，提供：

- Checkpoint。
- Retry。
- Resume。
- Timeout。
- Workflow History。
- Human Approval。

選型應在契約穩定後進行，避免讓框架反過來定義尚未成熟的業務流程。

## 10. Retention Policy

Artifact 不需要全部永久保留。

可以依類型設定：

| 類型 | 建議策略 |
|---|---|
| Current State | Run 完成後保留短期，或轉為 Final Summary 後刪除 |
| Latest Agent Results | Change 完成後保留數天至數週 |
| Validation Evidence | 依 Review／稽核需求保留 |
| Full Trace | 依成本、隱私與除錯價值取捨 |
| Human Approval | 長期或依法規保存 |
| Execution Summary | 進 Git 長期保存 |

Retention 必須同時考慮：

- 儲存成本。
- 隱私與 Secret Exposure。
- 問題追蹤週期。
- 合規要求。
- 是否已有 Git Commit／CI Run 可追溯。

## 11. Promotion：哪些 Runtime 知識應提升進 Git？

Change 完成時，從 Runtime Artifact 產生一份精簡 Execution Summary：

- 實際完成內容。
- 與原 Design 的差異。
- 主要修改檔案。
- 驗證結果。
- 接受風險。
- 未完成項。
- 重要決策。
- Commit／Release Reference。

不應提升：

- 完整聊天。
- 私有推理。
- 每輪失敗原文。
- 全部 stdout。
- Token 明細。
- 被後續版本取代的暫時摘要。

這個流程可以稱為：

> Runtime-to-Durable Knowledge Promotion

## 12. 維護性檢查

一個健康的 Shared State 方案應滿足：

- Git 中的長期文件數量不隨 Agent Retry 無限增加。
- Agent 正常工作不需讀取完整歷史。
- 任一 Artifact 都能指出 Producer、Run 與 Scope。
- Current State 能指出最新可接受 Artifact。
- Trace 可以獨立刪除而不破壞專案知識。
- Runtime Store 損壞時，至少能從 Durable Knowledge 與 Git 狀態重新建立工作流。

## 13. 參考資料

- LangGraph Persistence / Checkpointers  
  <https://docs.langchain.com/oss/javascript/langgraph/persistence>  
  <https://docs.langchain.com/oss/javascript/langgraph/checkpointers>
- A2A：Life of a Task  
  <https://a2a-protocol.org/latest/topics/life-of-a-task/>
- Azure：Event Sourcing Pattern  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing>
- Azure：Materialized View Pattern  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view>
- Azure：Claim Check Pattern  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check>
