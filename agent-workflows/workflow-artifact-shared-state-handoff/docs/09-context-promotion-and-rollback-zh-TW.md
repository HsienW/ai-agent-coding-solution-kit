# 09｜Context Promotion 與 Rollback

[English](./09-context-promotion-and-rollback.md) | [繁體中文](./09-context-promotion-and-rollback-zh-TW.md)

本文件規範 Coordinator、Orchestrator、Agent、Archive、Runtime Store 與 Human Approver 如何將 Candidate Context 提升為 Accepted Context、將 Runtime 結論整理為 Durable Knowledge，以及在已接受內容失效時執行可追溯的 Rollback。Promotion 與 Rollback 都會改變後續 Agent 採信的 Context，必須綁定固定版本、Gate、State Revision、Capability 與 Evidence。

本文件使用以下規範用語：

- 「必須」表示流程不得省略的要求。
- 「應」表示預設做法；偏離時需要留下原因。
- 「可以」表示依團隊規模與風險選用的能力。

## 1. 適用範圍

[`06-context-authority-and-retrieval-policy-zh-TW.md`](./06-context-authority-and-retrieval-policy-zh-TW.md) 定義 Agent 應採信哪些 Context；[`07-role-capability-and-scope-control-zh-TW.md`](./07-role-capability-and-scope-control-zh-TW.md) 限制誰能提出或套用變更；[`08-mutation-concurrency-and-conflict-resolution-zh-TW.md`](./08-mutation-concurrency-and-conflict-resolution-zh-TW.md) 規範 Accepted Pointer 的 CAS 與並行寫入。本文件接續處理：

- Candidate 需要通過哪些條件才能成為 Accepted Context？
- Accepted Pointer、Gate、Phase 與 Handoff 應如何一起更新？
- 哪些 Runtime 結論適合提升到 OpenSpec／Git？
- 已接受的 Artifact 發現 Regression 時，能否直接指回舊版本？
- Rollback Target 如何證明仍適用於目前 Requirement 與 Source Base？
- 哪些 Review、Readiness 與 Approval 會因 Rollback 失效？
- Active Run 與 Terminal Run 應採用什麼恢復路徑？
- 哪些動作可以自動化，哪些必須等待 Coordinator 或 Human？

本文件處理 Accepted Context Promotion、Runtime-to-Durable Knowledge Promotion、Pointer Re-selection、Compensating Rollback、Gate Invalidation、Cross-run Recovery 與 Rollback Evidence。以下內容由其他文件負責：

- Current State、Agent Result、Handoff 與 Artifact Reference 的基本結構，見 [`02-reference-architecture-zh-TW.md`](./02-reference-architecture-zh-TW.md)。
- Runtime、Evidence、Trace、Retention 與 Archive 分層，見 [`03-runtime-storage-and-retention-zh-TW.md`](./03-runtime-storage-and-retention-zh-TW.md)。
- Artifact Integrity、Human Gate 與 State Machine Governance，見 [`05-security-and-maintainability-zh-TW.md`](./05-security-and-maintainability-zh-TW.md)。
- Context Authority、Version Pinning、Freshness 與 Binding，見 [`06-context-authority-and-retrieval-policy-zh-TW.md`](./06-context-authority-and-retrieval-policy-zh-TW.md)。
- Role、Capability、Path、Tool 與 Approval Scope，見 [`07-role-capability-and-scope-control-zh-TW.md`](./07-role-capability-and-scope-control-zh-TW.md)。
- CAS、Atomic Replace、Idempotency 與 Conflict Routing，見 [`08-mutation-concurrency-and-conflict-resolution-zh-TW.md`](./08-mutation-concurrency-and-conflict-resolution-zh-TW.md)。

本文件不定義 Production Deployment、Database Rollback、Feature Flag、Traffic Shift 或業務資料補償的實作，也不修改現有 JSON Schema。

## 2. 名詞與操作邊界

### 2.1 Promotion

Promotion 表示受控流程提高一份 Context 的可採信等級或保存期限。本文件區分兩種 Promotion：

| 類型 | 來源 | 目標 | 主要結果 |
|---|---|---|---|
| Accepted Context Promotion | Candidate Artifact | Accepted Context | Current State 更新 Accepted Pointer |
| Runtime-to-Durable Knowledge Promotion | Accepted Runtime Artifact 與 Evidence | OpenSpec／Git | 建立可長期理解的 Execution Summary、ADR 或 Spec 修正 |

Accepted Context Promotion 改變本 Run 下游 Stage 的預設輸入。Durable Knowledge Promotion 改變後續 Change 與半年後仍需理解的工程事實。兩者需要不同 Gate，不能用同一個「已完成」布林值代替。

### 2.2 Rollback、Revert、Restore 與 Re-run

| 名稱 | 語意 | 本文件規則 |
|---|---|---|
| Rollback | 將有效行為或採信 Context 恢復到已知可接受狀態 | 以新決策與新 Revision 完成 |
| Revert | 對目前 Source 產生反向或補償 Diff | 屬於新的 Source Mutation 與 Candidate |
| Restore | 從備份或 Durable Store 恢復損壞資料 | 不得使 Current State Revision 倒退 |
| Re-run | 使用相同或修正後輸入再次執行 Stage | 產生新的 Attempt、Artifact 或 Run |
| Invalidate | 撤銷 Pointer、Gate、Handoff 或 Approval 的適用性 | 保留原紀錄並建立失效原因 |
| Supersede | 新 Artifact 或 Durable Decision 取代舊版本的現行效力 | 舊版本保持 Immutable 與可追溯 |

直接以舊 `current-state.json` 覆蓋目前 State 不屬於合法 Rollback。這會讓 `revision` 倒退，也可能復活已撤銷的 Gate、Approval 與 Handoff。

### 2.3 Promotion Target

Promotion Target 是 Promotion 後成為下游權威輸入或長期工程知識的資源。Target 必須具有明確的 Fact Type 與版本身分，例如：

```text
implementation fact → accepted implementation artifact
review verdict      → accepted review result
validation fact     → evidence bound to accepted implementation
execution outcome   → Git-tracked execution summary
design deviation    → approved OpenSpec / ADR update
```

一筆 Promotion 只能宣告它實際驗證的 Fact Type。通過 Implementation Review 不會自動批准 Requirement 變更，Archive 完成也不會自動授權 Commit 或 Push。

## 3. 角色與責任

| 角色 | Promotion／Rollback 責任 | 固定限制 |
|---|---|---|
| Coordinator | 判斷語意適用性、處理 Scope／Design Conflict、批准 Rollback Plan | 不直接修改 Current State，不代替 Reviewer 或 Human Gate |
| Implementer | 建立 Candidate、補償 Diff 與 Validation Evidence | 不接受自己的輸出，不覆寫舊 Candidate，不直接切換 Pointer |
| Reviewer | 審查固定 Candidate、Diff 與對應 Evidence | 不修改 Source、Gate、Accepted Pointer 或上游 Artifact |
| Readiness | 確認 Accepted Artifact 與 Gate 是否足以進入 Archive | 不自行恢復舊 Pointer，不補做 Implementer 工作 |
| Archive | 建立 Archive Result 與受控 Execution Summary | 不直接 Commit，不把未接受的 Runtime 內容提升進 Git |
| Orchestrator | 驗證版本、Gate、Capability 與 CAS，套用合法 State Mutation | 不做 Requirement 仲裁，不產生品質 Verdict，不代替 Human Approval |
| Human | 批准 Git、Deploy、Delete、外部 Rollback 與其他高風險 Action | Approval 必須綁定固定 Target、Diff、Scope、Revision 與有效期 |

Artifact Producer 可以在 Result 中提出 Promotion 或 Rollback 建議。建議本身沒有 Accept、Apply 或 Approve 權限。

## 4. Context Lifecycle Invariant

每個實作必須維持下列限制：

1. Candidate 只能經由明確 Gate 成為 Accepted Context。
2. Producer 不能更新自己輸出的 Accepted Pointer。
3. Promotion Decision 必須綁定固定 URI、Checksum、Change ID、Run ID 與 Target Identity。
4. Accepted Pointer、Gate、Phase、Owner 與 Handoff 使用同一筆 State CAS。
5. Promotion 失敗時保留原 Accepted Pointer。
6. 被 Supersede 或 Rollback 的 Artifact 保持 Immutable。
7. Rollback 不會讓 Current State `revision` 倒退。
8. Source 已改變時，Pointer Re-selection 不能代替 Source Revert。
9. Rollback 產生的新 Source 內容必須形成新的 Candidate。
10. 新 Candidate 必須取得自己的 Diff、Validation、Review 與 Approval Binding。
11. Terminal Run 不因後續問題重新開啟；後續處理建立新的 Change／Run。
12. Durable Knowledge 修正保留 Git 歷史，不原地抹除已發布決策。
13. Handoff、Review、Readiness 與 Approval 的適用性隨 Target Binding 改變而重新判定。
14. 外部副作用結果不明時，先查詢 System of Record。

Last-write-wins 禁止用於 Promotion Decision、Accepted Pointer、Gate、Approval 與 Rollback Target 選擇。

## 5. Promotion 類型與 Gate

不同 Artifact Kind 需要不同的 Promotion 條件：

| Promotion Target | 必要條件 | Decision Owner | 寫入者 |
|---|---|---|---|
| Plan／Design Candidate | Requirement Binding、Plan Review、Scope Approval | Coordinator／Human Policy | Orchestrator |
| Implementation Candidate | Schema、Diff Binding、Validation、Review Verdict | Reviewer 提供 Verdict；Policy 判定 Gate | Orchestrator |
| Review Result | 審查對象與 Accepted Candidate 相同、Result 合法 | Orchestrator 驗證 | Orchestrator |
| Readiness Result | Accepted Implementation、Review、Evidence 與 Task 狀態一致 | Readiness 提供 Verdict | Orchestrator |
| Archive Result | Readiness Passed、Summary 完整、Human Gate 狀態明確 | Archive 提出；Orchestrator 驗證 | Orchestrator |
| Execution Summary | 只包含 Accepted Outcome、Deviation、Risk 與 Reference | Archive／Coordinator | 受控 Git Executor |
| ADR／Spec 修正 | Durable Decision 經批准且 Scope 明確 | Coordinator／Human | 受控 Spec Writer／Git Executor |

Reviewer Verdict、Gate Transition、Pointer Update、Human Approval 與外部 Action 是分開的事件。任一事件存在，都不能推定其他事件已完成。

## 6. Accepted Context Promotion

### 6.1 Promotion Eligibility

Orchestrator 接受 Candidate 前必須確認：

- Candidate 通過對應 Agent Result Schema。
- `changeId`、`runId`、`producer`、`stage` 與 `kind` 符合 Handoff。
- Candidate URI 與 Checksum 已固定，內容完整且 Immutable。
- Candidate 的 Source Base、Diff Hash 與目前受審 Target 相同。
- Validation Evidence 綁定同一份 Candidate／Diff。
- Review Result 明確審查同一份 Candidate。
- Requirement、Acceptance Criteria 與 Scope 未在審查期間變更。
- Role、Capability、Path 與 Environment 仍在 Effective Scope。
- 必要 Gate 已通過，Blocker 與 Major Finding 已依 Policy 處理。
- Current State 仍符合讀取時的 Expected Revision。

任何一項無法證明時，Promotion 必須停止。較新的檔名、較大的 Attempt 或較晚的建立時間不能補足缺少的 Gate。

### 6.2 Promotion Decision

Promotion Decision 概念上至少包含：

```text
promotionId
promotionType
changeId
runId
actorId
candidateRef
targetFactType
sourceAuthority
targetAuthority
evidenceRefs
gateRefs
expectedRevision
policyVersion
decision
reasonCode
decidedAt
traceId
```

現行 Schema 沒有 Promotion Decision 欄位。本機第一版應由 Trace／Host Journal 或獨立 Artifact 保存，不能把它直接加入 Current State、Agent Result 或 Handoff。

### 6.3 Promotion 提交順序

Accepted Context Promotion 應依下列順序執行：

```text
load Current State at expectedRevision
→ resolve Candidate and required Evidence
→ validate Schema / checksum / target binding
→ evaluate Review and Promotion Gates
→ create Promotion Decision
→ build next State Snapshot
→ CAS Accepted Pointer + Gate + Phase + Owner + Handoff
→ record Mutation Receipt and Trace
```

Promotion Decision 必須在 CAS 前完成驗證，Receipt 則記錄實際提交結果。CAS 失敗時，Decision 只能視為尚未套用的提案；Orchestrator 重新讀取 State 後，依新 Revision 重新判斷。

### 6.4 Promotion 後的 Context

Promotion 成功後：

- `latestArtifacts` 指向新的 Accepted Artifact。
- 下游 Handoff 固定新的 URI 與 Checksum。
- 舊 Accepted Artifact 轉為 Superseded Context，仍可供稽核與 Rollback 分析。
- 未被接受的其他 Candidate 保持 Candidate 身分。
- 舊 Handoff 若綁定不同 Target，必須失效。
- 下游 Agent 不得自行選取目錄中的更新檔案替換 Pointer。

現行 Schema 尚未提供 `supersededBy`。Supersede 關係由 Trace、Host Journal、Artifact `description` 或獨立索引保存。

## 7. Runtime-to-Durable Knowledge Promotion

### 7.1 可提升內容

Change 完成或進入 Archive 時，Archive 應從 Accepted Artifact 與 Evidence 產生精簡的 Durable Output：

- 實際完成範圍與主要修改檔案。
- 最終 Validation 與 Review 結果。
- 與原 Proposal／Design 的差異。
- 已接受風險、未完成項與後續工作。
- 影響後續維護的重要決策。
- Accepted Artifact、Commit、CI 或 Release Reference。

Requirement、Design 或 Architecture Decision 發生持久變更時，應更新對應 OpenSpec 或 ADR。Execution Summary 不能代替原本應修正的 Durable Contract。

### 7.2 不提升內容

下列內容預設留在 Runtime、Evidence 或 Trace：

- 完整聊天與私有推理。
- 每次 Retry 的原始回覆。
- 全量 stdout、Debug Log 與 Token 明細。
- 未通過 Validation 的 Candidate 全文。
- 已被 Supersede 的暫時摘要。
- Secret、Credential、個人資料或未經批准的外部內容。
- 與最終 Decision 無關的診斷資料。

若稽核要求保留其中項目，Retention Policy 應指定受控 Store、存取權與期限，不應把內容直接提交進 Git。

### 7.3 Durable Promotion Gate

Archive 或 Git Executor 執行 Durable Promotion 前必須確認：

- Readiness Result 指向目前 Accepted Artifact。
- Execution Summary 只引用固定且可解析的 Reference。
- Summary 中的 Validation、Review 與 Risk 與原 Artifact 一致。
- Durable Contract 的修正已通過適用的 Coordinator／Human Gate。
- Commit Scope 沒有混入 Runtime Trace、Secret 或未接受 Candidate。
- Human Approval 綁定實際 Diff、Target Branch、Action 與有效期。

Archive Agent 可以建立 `archive_result` 與 Execution Summary Candidate。只有具有對應 Capability 的受控 Executor 能寫入 Git；Commit 與 Push 仍是兩個獨立 Action。

### 7.4 Durable Promotion 完成條件

Promotion 只有在 System of Record 可以查到結果後才算完成：

```text
prepare Execution Summary / Spec update
→ validate content and references
→ obtain Human Approval when required
→ commit through controlled Executor
→ verify Commit Identity in Git
→ record Commit Reference in Archive Evidence / Trace
```

Client Timeout 或工具回應遺失時，Executor 必須先查詢 Git／CI 等 System of Record。無法確認結果時回傳 `DURABLE_PROMOTION_OUTCOME_UNKNOWN`，不得盲目重送 Commit 或 Push。

## 8. Promotion 的 CAS 與並行決策

兩個 Candidate 同時要求 Promotion 時，兩者都必須以讀取時的 Revision 提交：

```text
candidate v2 reads revision 8
candidate v3 reads revision 8

v2 passes gates and commits revision 9
v3 submits expectedRevision 8
→ STATE_REVISION_CONFLICT
→ accepted pointer remains v2
```

v3 保持 Candidate。Orchestrator 不能只把 `expectedRevision` 改成 9 後重送；它必須重新檢查 v3 是否仍符合目前 Scope、Gate、Owner、Review Target 與 Promotion Policy。

若 v2 與 v3 都合法但代表互斥的工程決定，流程回傳 `PROMOTION_AUTHORITY_CONFLICT` 並進入 `NEEDS_COORDINATOR_ARBITRATION`。建立時間不能作為仲裁規則。

## 9. Rollback 類型

### 9.1 Accepted Pointer Re-selection

Pointer Re-selection 將下游 Context 重新指向先前已接受的 Artifact。只有以下條件全部成立時才可以使用：

- 實際 Source、Config 與執行基線仍與舊 Artifact 描述相同。
- 舊 Artifact 通過 Integrity、Retention 與可讀性檢查。
- Requirement、Schema、Policy 與 Environment 未使舊版本失效。
- 所需 Validation 仍在 Freshness Policy 允許範圍。
- 新 State Mutation 重新計算 Gate、Owner 與 Handoff。

只要 Source 已包含後續版本，Pointer Re-selection 就不能宣告系統已恢復。此時必須建立 Compensating Candidate。

### 9.2 Compensating Source Rollback

Compensating Rollback 在目前 Source Base 上建立新的修正內容，使行為回到可接受狀態：

```text
current source with v2
+ accepted behavior from v1
+ current requirement / schema / dependency constraints
                    ↓
           rollback candidate v3
```

v3 是新的 Candidate。它可能使用 Revert Commit、反向 Patch、手動修正或替代實作，但都必須產生新的 Diff Hash、Changed Files、Validation Evidence 與 Review Result。

### 9.3 Durable Knowledge Correction

已提交的 Execution Summary、ADR、Spec 或 Task 狀態有誤時，Coordinator 應建立新的 OpenSpec／Git Change 修正。修正內容必須：

- 指出被修正的 Commit、Document Version 或 Decision Reference。
- 說明哪些事實仍有效，哪些已被 Supersede。
- 提供新的 Requirement、Design 或 Execution Evidence。
- 經過原文件類型要求的 Review 與 Human Git Gate。

歷史 Commit 不應因內容過時而被原地改寫。需要重寫公開歷史的特殊情況由 Repository Governance 處理，不在 Agent 自動化權限內。

### 9.4 External Environment Rollback

Deploy、Migration、Feature Flag、Remote Config 與業務資料 Rollback 由各自的操作規範執行。本文件只要求：

- Handoff 固定 Environment、Target Version、Action 與 Evidence。
- Executor 具有明確 Capability 與有效 Human Approval。
- 操作使用 Idempotency Key 或平台等價機制。
- 執行前後都讀取 System of Record。
- 結果以 External Reference 回寫 Trace／Archive Evidence。

Agent Result、Reviewer Verdict 或 `requiresHumanApproval: true` 都不是外部 Rollback 的 Approval Proof。

## 10. Rollback Target 選擇

### 10.1 Target Eligibility

Coordinator 與 Orchestrator 選擇 Rollback Target 時必須檢查：

| 檢查 | 問題 |
|---|---|
| Identity | Target 的 URI、Checksum、Commit、Run 與 Producer 是否可驗證？ |
| Prior Acceptance | Target 是否曾通過當時必要的 Validation 與 Review？ |
| Current Requirement | Target 行為是否仍符合目前批准的 Requirement？ |
| Current Base | Source、Dependency、Schema 與 Environment 是否仍相容？ |
| Security | Target 是否包含已知 Vulnerability、撤銷 Credential 或禁止設定？ |
| Retention | 必要 Artifact、Diff 與 Evidence 是否仍可取得？ |
| Scope | Rollback 會影響哪些 Path、Component、Data 與外部系統？ |
| Approval | 目前 Action 是否需要新的 Human Approval？ |

「以前可用」只證明 Target 曾經成立，不能證明它仍適用於現在。

### 10.2 Rollback Basis 與 Rollback Candidate

Rollback Basis 是用來恢復行為的舊 Artifact、Commit 或 Decision；Rollback Candidate 是在目前基線上產生的新結果。兩者必須分開記錄：

```text
rollbackBasisRef: accepted-v1
currentBaseRef: source-at-v2
rollbackCandidateRef: candidate-v3
```

Reviewer 審查 v3，不能把 v1 的舊 Verdict 直接套用到 v3。v1 的 Evidence 可作為比較基準，但不會取代目前環境的 Validation。

### 10.3 Target 無法使用

符合下列任一條件時，流程不能自動 Rollback：

- Target Artifact 已遺失或 Checksum 不符。
- Requirement、Schema、Dependency 或資料格式已不相容。
- 舊版本含有已知 Security Issue。
- Rollback 需要超出批准 Scope 的資料或權限。
- 無法確認目前 Source／Environment Identity。
- 多個 Rollback Target 都合法，但行為互斥。

Coordinator 應改為建立新的修正 Plan；涉及高風險外部狀態時交由 Human 決定。

## 11. 補償式 Rollback 流程

### 11.1 Prepare

Coordinator 建立 Rollback Plan，至少固定：

```text
incident / regression reference
current accepted artifact
current source base
rollback basis
affected scope
expected restored behavior
required validation
required review
human-gated actions
```

若 Regression 顯示原 Requirement 或 Design 本身錯誤，Coordinator 必須先更新 Durable Contract，再交給 Implementer。

### 11.2 Implement

Implementer 在獨立 Worktree 或受控工作樹上：

1. 驗證 Current Base 與 Handoff 指定版本相同。
2. 讀取 Rollback Basis、目前 Requirement 與必要 Source。
3. 建立補償 Diff，不修改舊 Artifact。
4. 執行指定 Validation，產生新的 Evidence。
5. 發布新的 Implementation Candidate。
6. 提出 Review Handoff，不自行更新 Pointer。

Base Divergence 時沿用 08 的 `SOURCE_BASE_DIVERGED`，不能默默套用舊 Patch。

### 11.3 Review 與 Readiness

Reviewer 必須檢查：

- 補償 Diff 是否只涵蓋批准 Scope。
- Candidate 是否恢復預期行為。
- 後續 Requirement、Schema 或 Security Fix 是否被意外移除。
- Validation 是否覆蓋原 Regression 與目前整合行為。
- Residual Risk 是否需要 Coordinator 或 Human 接受。

Review 通過後，Readiness 仍需針對 Rollback Candidate 檢查 Gate。舊 Candidate 的 Readiness Result 不適用。

### 11.4 Accept

Orchestrator 使用新的 Promotion Decision 與 Expected Revision 提交：

```text
accepted implementation: candidate-v3
review result: review-v3
readiness result: readiness-v3
next phase: policy-defined legal phase
next owner: policy-defined owner
```

Accepted Pointer、相關 Gate、Phase、Owner 與 Handoff 必須同時更新。CAS 失敗時保留目前 State，重新載入後再判斷 v3 是否仍可接受。

### 11.5 Durable Record

Rollback 完成後，Execution Summary 或後續 Change 應保存：

- Regression／Incident Reference。
- 被取代的 Accepted Artifact。
- Rollback Basis 與新 Candidate。
- 實際 Source／Environment 變更。
- Validation、Review 與 Human Approval Reference。
- 未恢復項目與後續工作。

原本的 Promotion Decision 與 Artifact 不刪除；Retention 到期時，至少保留可重建決策關係的 Metadata。

## 12. Gate、Handoff 與 Approval 失效

### 12.1 Gate Invalidation Matrix

Rollback 或 Promotion Target 改變後，Orchestrator 應依影響重新計算 Gate：

| 變更 | 預設失效項目 |
|---|---|
| Implementation URI／Checksum 改變 | `implementationComplete`、`reviewPassed`、`readinessPassed`、`humanApproved` |
| Diff Hash／Changed Files 改變 | Review、Readiness、Commit／Deploy Approval |
| Requirement／Scope 改變 | Plan Approval 之後的所有下游 Gate |
| Validation Target 改變 | Review 與 Readiness |
| Review Result 改變 | Readiness 與後續 Approval |
| Target Branch／Environment 改變 | 對應 Git／Deploy Approval |
| Policy Version 或 Capability Scope 改變 | Authorization Decision 與尚未執行的 Approval |

實際清除範圍由版本化 Transition Policy 定義。Orchestrator 不能為了維持流程進度而保留已失去 Binding 的 `true`。

### 12.2 Handoff Invalidation

下列情況會使既有 Handoff 失效：

- Current State Revision 已改變，且原 Handoff 不再符合目前 Owner 或 Phase。
- Required Input Ref 指向被 Rollback 或 Supersede 的 Target。
- Checksum、Run ID、Change ID 或 Source Base 不符。
- Acceptance Criteria、Scope、Policy 或 Permission 已改變。
- `expiresAt` 已到期。

失效後，Orchestrator 應依最新 State 建立新 Handoff。Agent 不得自行替換 Reference 後繼續執行。

### 12.3 Approval Invalidation

Human Approval 必須綁定固定 Action 與內容。以下變動需要新的 Approval：

- Diff、Payload、Commit、Target Branch 或 Environment 改變。
- Rollback 從本機 Source 擴張到外部 Deploy／Data Mutation。
- State Revision 或 Policy Version 改變後，原批准條件不再成立。
- Approval 已到期或被撤銷。
- Executor、Tool 或 Capability Scope 改變。

Reviewer 的 `approve` 不能替代 Human Approval；舊 Human Approval 也不能替代新 Candidate 的 Review。

## 13. Active Run、Terminal Run 與 Cross-run

### 13.1 Active Run

Run 尚未 Terminal 時，如果 Promotion 後發現問題，Orchestrator 應依合法 Transition 路由：

- 需要修正實作時，進入 `CHANGES_REQUESTED` 並建立 `fix-from-review` 或 Policy 允許的實作 Handoff。
- Requirement、Design 或 Scope 有衝突時，進入 `NEEDS_COORDINATOR_ARBITRATION`。
- 缺少必要 Context 或 Evidence 時，使用 `INCOMPLETE` 或安全 Phase 加 Blocker。
- Integrity 無法證明時，使用 `FAILED` 或安全 Phase 加 Blocker。

現行 Phase Enum 沒有 `ROLLING_BACK`。實作不能自行新增 Phase；Rollback 意圖由 Handoff Reason、Blocker、Trace 與 Artifact Reference 表達。

### 13.2 Terminal Run

`terminal: true` 的 Run 保持封閉。後續 Regression、Requirement 修正或 Rollback 應建立新的 Change／Run，並引用原 Run 的 Accepted Artifact、Execution Summary、Commit 與 Incident Reference。

新 Run 可以使用原 Artifact 作為 Diagnostic 或 Rollback Basis，但不能修改原 Run 的 Result、Gate、Handoff 或 Revision History。

### 13.3 Cross-run Binding

跨 Run 引用必須明確允許，且至少驗證：

```text
source changeId / runId
target changeId / runId
fact type and purpose
artifact URI / checksum
source base / commit
authority level
retention availability
```

舊 Run 的 Accepted Artifact 在新 Run 中預設是 Diagnostic／Rollback Basis。只有新 Run 完成必要 Gate 後，新 Candidate 才能成為目前 Accepted Context。

## 14. State Recovery 與 Rollback 的分界

Runtime Store 損壞時，Host 可以從 Durable Knowledge、Git、Artifact 與 Mutation Receipt 重建 Current State。重建後仍須建立較新的 `revision`，並保存 Recovery Trace。

下列做法必須禁止：

- 把備份中的舊 `current-state.json` 直接覆蓋到目前檔案。
- 從檔案修改時間猜測哪一份 State 應被接受。
- 在 Artifact 缺失時保留指向它的 Accepted Pointer。
- 將舊 Gate 與 Human Approval 無條件複製到重建 State。
- 為了通過 Schema 而刪除無法解釋的 Blocker 或 History Reference。

Recovery 解決資料完整性；Rollback 解決工程行為或採信 Context。兩者可以發生在同一事故中，但必須留下不同 Reason Code 與 Evidence。

## 15. 固定失敗代碼與路由

### 15.1 Promotion Failure

| Code | 判定條件 | 預設處理 |
|---|---|---|
| `PROMOTION_PRECONDITION_FAILED` | 必要 Gate、Evidence 或 Policy 條件未成立 | 保留原 Pointer，補齊前置條件 |
| `PROMOTION_TARGET_STALE` | Candidate、Validation 或 Source Base 已過期 | 建立新 Handoff／重新 Validation |
| `PROMOTION_BINDING_MISMATCH` | Candidate、Diff、Validation、Review 指向不同 Target | `INCOMPLETE`，拒絕 Promotion |
| `PROMOTION_GATE_INVALID` | Gate 內容與 Target 或 Policy 不相容 | 清除無效 Gate，回到合法 Stage |
| `PROMOTION_AUTHORITY_CONFLICT` | 多個合法 Candidate 代表互斥決定 | `NEEDS_COORDINATOR_ARBITRATION` |
| `DURABLE_PROMOTION_INCOMPLETE` | Summary／Spec 修正或必要 Reference 不完整 | 停止 Archive／Git Action |
| `DURABLE_PROMOTION_OUTCOME_UNKNOWN` | 無法確認 Commit、Push 或外部寫入結果 | 查詢 System of Record，停止重送 |

### 15.2 Rollback Failure

| Code | 判定條件 | 預設處理 |
|---|---|---|
| `ROLLBACK_TARGET_MISSING` | Rollback Basis 已遺失或 Retention 不可用 | 停止，自 Coordinator 取得替代 Plan |
| `ROLLBACK_TARGET_INTEGRITY_FAILED` | Checksum、Schema 或來源身分不符 | `FAILED`，隔離 Target |
| `ROLLBACK_BASE_DIVERGED` | Target 與目前 Source／Environment 不相容 | 建立補償 Plan，不直接套用 |
| `ROLLBACK_REVALIDATION_REQUIRED` | 舊 Evidence 無法涵蓋目前 Candidate | 建立 Validation Handoff |
| `ROLLBACK_SCOPE_EXPANDED` | 實際影響超過批准 Scope | 暫停並要求 Coordinator／Human |
| `ROLLBACK_GATE_INVALIDATED` | Target 改變使既有 Gate 失效 | 清除適用 Gate，重新 Review／Readiness |
| `TERMINAL_RUN_IMMUTABLE` | 要求修改已封閉 Run | 建立新的 Change／Run |
| `EXTERNAL_STATE_DIVERGED` | System of Record 與預期狀態不同 | 停止自動化並交由受控操作流程 |

CAS 衝突、Base Divergence、Idempotency 與 Unknown Side-effect Outcome 沿用 08 的既有代碼。Context Integrity 與 Authority 問題可以沿用 06 的固定代碼，避免為相同條件建立多組名稱。

### 15.3 Failure Result

Promotion／Rollback Failure 的 Trace 或 Result 至少應保存：

```text
code
changeId
runId
stage
actorId
currentAcceptedRef
candidateRef / rollbackBasisRef
expectedIdentity
actualIdentity
invalidatedBindings
nextAllowedAction
traceId
```

這是概念格式。使用現有 Agent Result 時，角色應選擇合法的 `blocked`、`failed` 或 `incomplete` Status，並把固定 Code 放入既有 Payload、Blocker、Risk、Summary 或 Trace；不得加入 Schema 未定義欄位。

## 16. Handoff 與現行 Schema 相容

### 16.1 現有 Handoff 的使用方式

[`handoff-envelope.schema.json`](../schemas/handoff-envelope.schema.json) 沒有 `promotionId`、`rollbackBasisRef`、`rollbackCandidateRef`、`invalidatedGates` 或 `targetAuthority` 欄位，而且使用 `additionalProperties: false`。現行 Handoff 不能直接加入這些欄位。

第一版可以使用：

- `requiredInputRefs` 固定 Current Accepted Artifact、Rollback Basis、Current Source、Requirement 與 Evidence。
- Artifact Reference 的 `uri` 與 `sha256` 固定版本。
- `description` 說明 Reference 是 Candidate、Accepted、Rollback Basis 或 Diagnostic Input。
- `reason` 說明 Promotion／Rollback 任務與觸發原因。
- `acceptanceCriteriaRefs` 指向 Regression Test、Requirement 與 Gate Criteria。
- `onSuccess`、`onFailure`、`onConflict` 指向現有合法 Phase 與 Owner。
- Orchestrator Metadata 攜帶 Expected Revision、Idempotency Key 與 Policy Version。
- Trace／Host Journal 保存 Promotion Decision、Rollback Record 與失效細節。

### 16.2 Handoff 片段

以下為閱讀片段，省略 Handoff Schema 的其他必要欄位：

```json
{
  "from": "coordinator",
  "to": "implementer",
  "stage": "fix-from-review",
  "reason": "Create a compensating rollback candidate for regression REG-204",
  "requiredInputRefs": [
    {
      "refId": "current-accepted-v2",
      "type": "agent-result",
      "uri": "artifact://feature-example/run-004/implementation-v2.json",
      "sha256": "2222222222222222222222222222222222222222222222222222222222222222",
      "description": "Current accepted implementation; regression target"
    },
    {
      "refId": "rollback-basis-v1",
      "type": "agent-result",
      "uri": "artifact://feature-example/run-003/implementation-v1.json",
      "sha256": "1111111111111111111111111111111111111111111111111111111111111111",
      "description": "Previously accepted implementation; rollback basis only"
    }
  ]
}
```

`description` 提供用途說明，不會自行提高 Authority。Orchestrator 仍要驗證 Rollback Basis 的來源、Integrity、Retention 與目前適用性。

### 16.3 Current State 相容方式

現行 [`current-state.schema.json`](../schemas/current-state.schema.json) 可以表達：

- 目前 Phase、Owner 與 Revision。
- `latestArtifacts` 中的 Accepted Pointer。
- Promotion／Rollback 後重新計算的 Gate。
- 固定 Failure Code 的 Blocker。
- 新 Handoff 與 Next Action。

它無法完整表達 Promotion History、Supersede Graph 或 Rollback Basis。這些資料留在 Trace、Host Journal、Execution Summary 或獨立 Artifact；不能把自訂欄位塞入 Current State。

## 17. 概念性 Promotion／Rollback Record

以下 YAML 說明 Decision Record，尚未對應本庫正式 Schema：

```yaml
schemaVersion: 0.1.0-draft
recordId: promotion-record-004
recordType: accepted-context-promotion
changeId: feature-example
runId: run-004

from:
  authority: candidate
  artifactRef: artifact://feature-example/run-004/implementation-v3.json

to:
  authority: accepted
  factType: implementation

bindings:
  sourceBase: git:def456
  diffHash: sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
  validationRef: evidence://feature-example/run-004/validation-v3.json
  reviewRef: artifact://feature-example/run-004/review-v3.json
  expectedRevision: 11

decision:
  outcome: accepted
  appliedRevision: 12
  policyVersion: 1.0.0
  traceId: trace-promotion-004
```

Rollback Record 可以另存：

```yaml
schemaVersion: 0.1.0-draft
recordId: rollback-record-001
recordType: compensating-rollback
changeId: feature-example
runId: run-004

currentAcceptedRef: artifact://feature-example/run-004/implementation-v2.json
rollbackBasisRef: artifact://feature-example/run-003/implementation-v1.json
rollbackCandidateRef: artifact://feature-example/run-004/implementation-v3.json

reason:
  code: REGRESSION_CONFIRMED
  incidentRef: repo://incidents/REG-204.md

invalidatedBindings:
  - review-v2
  - readiness-v2
  - approval-v2

result:
  acceptedRef: artifact://feature-example/run-004/implementation-v3.json
  appliedRevision: 12
  traceId: trace-rollback-001
```

正式採用前，至少需要：

```text
schemas/promotion-decision.schema.json
schemas/rollback-record.schema.json
templates/promotion-policy.template.yaml
examples/accepted-context-promotion/
examples/compensating-rollback/
```

這些資產屬於後續契約升級。本文件的本機導入不依賴它們。

## 18. 本機完整範例

### 18.1 v2 Promotion

目前狀態如下：

```text
revision: 8
accepted implementation: v1
source base: abc123

candidate v2
  source base: abc123
  validation: passed
  review: approved
```

Orchestrator 驗證 Binding 與 Gate 後，以 `expectedRevision: 8` 提交 v2。CAS 成功產生 Revision 9，`latestArtifacts.implementationResult` 指向 v2，下游 Handoff 固定 v2 與其 Evidence。

### 18.2 發現 Regression

Readiness 或後續人工檢查發現 v2 破壞既有行為：

```text
regression: REG-204
current accepted: v2
known good basis: v1
current source: includes v2
```

因 Source 已包含 v2，Orchestrator 不能只把 Pointer 指回 v1。Coordinator 建立 Rollback Plan，把 v1 列為 Rollback Basis，並將原 Regression Test、目前 Requirement 與 Source Base 交給 Implementer。

### 18.3 建立 v3

Implementer 在目前 Source Base 上產生 v3：

```text
candidate v3
  rollback basis: v1
  current base: source-with-v2
  changed files:
    - src/payment/service.ts
    - tests/payment/service.test.ts
  validation:
    regression-test: passed
    current-suite: passed
```

v3 恢復 v1 的可接受行為，同時保留 v2 之後仍有效的 Dependency、Schema 與 Security Change。它取得新的 Diff Hash、Validation、Review 與 Readiness Result。

### 18.4 Gate Invalidation 與重新接受

建立 v3 時，v2 的 Review、Readiness 與 Human Approval 失效。Orchestrator 以新 Handoff 路由 Review 與 Readiness；完成後使用新的 Expected Revision 更新：

```text
accepted implementation: v3
review result: review-v3
readiness result: readiness-v3
revision: 9 → 10 → ... → 12
```

State Revision 持續向前。v1、v2 與 v3 都保留原 URI；Accepted Pointer 最終指向 v3。

### 18.5 Archive 與 Git

Archive 的 Execution Summary 記錄 v2 Regression、v1 Rollback Basis、v3 補償內容與驗證結果。若需要 Commit，Human Approval 綁定 v3 的 Diff Hash 與 Commit Scope；受控 Executor 執行後查詢 Git，保存 Commit Reference。

若 Regression 在原 Run 已 `COMPLETED` 後才發現，整段處理改在新的 Change／Run 執行，原 Run 保持不變。

## 19. 最小本機導入

不修改現有 Schema，也能採用本文件的大部分規則：

1. 將 `latestArtifacts` 解讀為目前 Accepted Pointer。
2. Producer 只建立 Candidate；Orchestrator 更新 Pointer。
3. Promotion 前驗證 URI、Checksum、Change ID、Run ID、Diff 與 Evidence Binding。
4. Accepted Pointer、Gate、Phase、Owner 與 Handoff 使用同一筆 CAS。
5. Promotion 或 Rollback 後，重新建立綁定新 Revision 的 Handoff。
6. Source 已改變時，建立 Compensating Candidate，不只切換 Pointer。
7. Rollback Candidate 取得新的 Diff、Validation、Review 與 Readiness。
8. Current State `revision` 只向前增加。
9. Terminal Run 保持 Immutable；後續處理建立新 Change／Run。
10. Promotion／Rollback Decision 先放在 Trace 或 Host Journal。
11. Archive 只提升 Accepted Outcome 與必要 Engineering Decision。
12. Runtime Trace、失敗全文與 Secret 不進 Git。
13. Commit、Push、Deploy、Delete 與 External Rollback 保留 Human Gate。
14. 外部寫入結果不明時先查詢 System of Record。

### 19.1 可選本機目錄

Git-ignored Runtime 可以加入 Host 專用紀錄：

```text
.agent-runtime/<change-id>/
├─ current-state.json
├─ runs/
│  └─ <run-id>/
│     ├─ artifacts/
│     └─ evidence/
└─ host-journal/
   ├─ promotion-decisions/
   └─ rollback-records/
```

`host-journal/` 是 Runtime Implementation Detail。Agent 不應直接修改，也不應把整個目錄當成一般 Context 載入。

### 19.2 最小 Orchestrator 介面

本機 Orchestrator 可以封裝兩個受控介面：

```text
promoteAcceptedContext(
  changeId,
  expectedRevision,
  candidateRef,
  evidenceRefs,
  gateDecision,
  idempotencyKey
)

applyRollbackPromotion(
  changeId,
  expectedRevision,
  rollbackBasisRef,
  rollbackCandidateRef,
  evidenceRefs,
  invalidatedBindings,
  idempotencyKey
)
```

兩個介面都負責 Capability、Schema、Integrity、Binding、Transition、CAS 與 Receipt。它們不能替代 Coordinator 的語意決策、Reviewer Verdict 或 Human Approval。

## 20. 驗收清單

### Promotion

- [ ] Candidate 通過 Schema、Integrity、Target 與 Source Base 檢查。
- [ ] Validation、Review 與 Candidate 綁定同一份 Diff。
- [ ] Producer 不能接受自己的輸出。
- [ ] Promotion 使用讀取時的 Expected Revision。
- [ ] Pointer、Gate、Phase、Owner 與 Handoff 使用同一筆 CAS。
- [ ] CAS 失敗時保留原 Accepted Pointer。
- [ ] Superseded Artifact 保持 Immutable。

### Durable Knowledge

- [ ] Archive 只提升 Accepted Outcome、Deviation、Risk 與必要 Reference。
- [ ] Requirement／Design 改變時更新 OpenSpec 或 ADR。
- [ ] 完整 Trace、失敗全文、Token 與 Secret 不進 Git。
- [ ] Execution Summary 中的 Artifact 與 Evidence Reference 可解析。
- [ ] Commit 與 Push 分別取得所需 Human Approval。
- [ ] Git 寫入完成後可查到 Commit Identity。

### Rollback Target

- [ ] Rollback Basis 的 URI、Checksum、Run 與 prior acceptance 可驗證。
- [ ] Target 仍符合目前 Requirement、Schema、Dependency 與 Security Policy。
- [ ] Rollback Basis 與 Rollback Candidate 分開記錄。
- [ ] Source 已改變時不只執行 Pointer Re-selection。
- [ ] 無法證明 Target 可用時停止自動化。

### Compensating Rollback

- [ ] Rollback Candidate 建立在目前 Source Base。
- [ ] 新 Candidate 有自己的 Diff Hash、Changed Files 與 Evidence。
- [ ] Validation 涵蓋原 Regression 與目前整合行為。
- [ ] Reviewer 重新審查 Rollback Candidate。
- [ ] Readiness 與 Human Approval 綁定新 Target。
- [ ] Current State Revision 沒有倒退。

### Invalidation 與 Lifecycle

- [ ] Target 改變後重新判定 Gate、Handoff 與 Approval。
- [ ] 失效的 Handoff 不會由 Agent 自行替換 Reference 後繼續使用。
- [ ] 現行 Phase 不會加入未定義的 `ROLLING_BACK`。
- [ ] Terminal Run 不會被重新開啟或原地修改。
- [ ] Cross-run Reference 有明確 Purpose、Identity 與 Integrity Binding。
- [ ] State Recovery 與工程 Rollback 使用不同 Reason Code。

### Schema 相容性

- [ ] 現行 Current State 沒有加入 `promotionRecord`、`rollbackTarget` 或 `supersededBy`。
- [ ] 現行 Handoff 沒有加入未定義的 Promotion／Rollback 欄位。
- [ ] Decision 與 History 先保存在 Trace、Host Journal 或獨立 Artifact。
- [ ] Failure Code 使用既有 Blocker、Result 或 Trace 承載。
- [ ] 未來 Schema 在正式採用前有 Version、Fixture 與 Migration 規則。

## 21. 參考文件

- [Artifact-based Shared State + Structured Handoff 參考架構](./02-reference-architecture-zh-TW.md)
- [Repository Knowledge、Runtime State、Evidence 與 Trace 分層](./03-runtime-storage-and-retention-zh-TW.md)
- [安全、治理與長期維護](./05-security-and-maintainability-zh-TW.md)
- [Context Authority 與 Retrieval Policy](./06-context-authority-and-retrieval-policy-zh-TW.md)
- [Role Capability 與 Scope Control](./07-role-capability-and-scope-control-zh-TW.md)
- [Mutation、Concurrency 與 Conflict Resolution](./08-mutation-concurrency-and-conflict-resolution-zh-TW.md)
- [Agent Platform Operations](../../../context-engineering/docs/03-agent-platform-operations-zh-TW.md)
- [Tool Governance and Evaluation](../../../agent-design/tool-schema-routing/docs/03-tool-governance-and-evaluation-zh-TW.md)
- [Current State Schema](../schemas/current-state.schema.json)
- [Agent Result Schema](../schemas/agent-result.schema.json)
- [Handoff Envelope Schema](../schemas/handoff-envelope.schema.json)
- [Execution Summary Template](../templates/execution-summary.template.md)
- [Workflow Policy Template](../templates/workflow-policy.template.yaml)
