# 08｜Mutation、Concurrency 與 Conflict Resolution

[English](./08-mutation-concurrency-and-conflict-resolution.md) | [繁體中文](./08-mutation-concurrency-and-conflict-resolution-zh-TW.md)

本文件規範 Coordinator、Orchestrator、Agent Adapter、Runtime Store 與受控 Executor 如何提交 Mutation、處理並行寫入，以及在 State、Source、Artifact 或外部副作用發生衝突時恢復。每一筆 Mutation 都必須綁定 Actor、Capability、Target、Base Identity 與 Idempotency Identity；系統只在前置版本、Gate 與資源狀態仍符合預期時接受寫入。

本文件使用以下規範用語：

- 「必須」表示流程不得省略的要求。
- 「應」表示預設做法；偏離時需要留下原因。
- 「可以」表示依團隊規模與風險選用的能力。

## 1. 適用範圍

[`06-context-authority-and-retrieval-policy-zh-TW.md`](./06-context-authority-and-retrieval-policy-zh-TW.md) 固定 Agent 讀取的 State Revision、Candidate 與 Accepted Artifact；[`07-role-capability-and-scope-control-zh-TW.md`](./07-role-capability-and-scope-control-zh-TW.md) 判斷 Actor 是否有權對指定 Resource 執行 Action。本文件接續處理實際寫入：

- Current State 在讀取後被其他 Writer 更新時，如何拒絕舊寫入？
- Artifact、Handoff 與 State 應依什麼順序發布？
- 多個 Implementer 如何隔離 Source Mutation？
- Git 可以完成文字合併時，是否仍需重新 Validation 與 Review？
- Lock、Lease、CAS 與 Fencing Token 各自解決哪一種競爭？
- Retry 如何避免重複 Commit、Push 或其他外部副作用？
- 哪些 Conflict 可以自動處理，哪些必須交回 Coordinator 或 Human？

本文件處理 State CAS、Atomic Replace、Crash Consistency、Artifact Immutability、Worktree Integration、Idempotency、Lock／Lease、Conflict Classification 與 Recovery Routing。以下內容由其他文件負責：

- Current State、Agent Result、Handoff 與 Artifact Reference 的基本結構，見 [`02-reference-architecture-zh-TW.md`](./02-reference-architecture-zh-TW.md)。
- Runtime Storage、Retention 與 Event Sourcing 升級時機，見 [`03-runtime-storage-and-retention-zh-TW.md`](./03-runtime-storage-and-retention-zh-TW.md)。
- Concurrency Race、Artifact Tampering 與 Single Writer 的安全風險，見 [`05-security-and-maintainability-zh-TW.md`](./05-security-and-maintainability-zh-TW.md)。
- Context Version、Checksum 與 Candidate／Accepted 選擇，見 [`06-context-authority-and-retrieval-policy-zh-TW.md`](./06-context-authority-and-retrieval-policy-zh-TW.md)。
- Role、Capability、Path 與 Tool Scope，見 [`07-role-capability-and-scope-control-zh-TW.md`](./07-role-capability-and-scope-control-zh-TW.md)。

本文件不定義 Production Deployment、Release Promotion、Database Transaction 或業務補償流程，也不修改現有 JSON Schema。

## 2. Mutation 類型與寫入責任

Mutation 指任何會改變 Repository、Runtime Store、Artifact Registry 或外部 System of Record 的操作。不同 Target 需要不同的隔離與提交方式。

| Mutation 類型 | Target | 預設 Writer | 主要控制 |
|---|---|---|---|
| Specification Mutation | Requirement、Proposal、Design、Tasks、ADR | Coordinator／受控 Human | Change Scope、Single Writer、Git Identity |
| Source Mutation | Source、Test、Config、Generated File | Implementer | Worktree、Base Commit、Path Ownership |
| Artifact Publication | Agent Result、Evidence、Handoff | 對應 Producer／Orchestrator | Unique URI、Create-only、Checksum、Immutable |
| Runtime State Mutation | Current State、Gate、Accepted Pointer | Orchestrator／受控 Automation | Expected Revision、CAS、Atomic Replace |
| External Side Effect | Commit、Push、Issue Update、Deploy、Delete | 受控 Executor | Human Gate、Idempotency、Before／After Reference |

Reviewer、Readiness 與 Archive 可以建立自己的 Stage Output，不能修改 Source、上游 Artifact、Gate 或 Current State。Producer 建立 Candidate；Orchestrator 驗證後才可以更新 Accepted Pointer。

### 2.1 Mutation Identity

一筆 Mutation 概念上至少包含：

```text
mutationId
idempotencyKey
actorId
changeId
runId
stage
capability
resource
baseRevision / baseCommit
payloadHash
```

這些欄位用於判定「誰要修改什麼、根據哪個版本、是否已執行過」。現有 Schema 尚未定義完整 Mutation Envelope；本機第一版由 Orchestrator Invocation Metadata 與 Trace 保存，不能直接塞入 `additionalProperties: false` 的 Current State、Agent Result 或 Handoff。

### 2.2 Prepare、Publish 與 Accept

三個動作必須分開：

| 動作 | 說明 | 執行者 |
|---|---|---|
| Prepare | 在隔離區建立 Source Change、Candidate Artifact 或下一版 State | Agent／Orchestrator |
| Publish | 將完整且通過基本驗證的 Candidate 放到唯一 URI | Producer Host／Orchestrator |
| Accept | 通過 Gate 後更新 Current State Pointer | Orchestrator |

Publish 不會自動帶來 Accepted 身分。Accept 必須再次確認 State Revision、Gate、Artifact Binding 與 Capability。

## 3. Concurrency Invariant

每個實作必須維持下列限制：

1. 同一 Change 在同一時間只有一個 Current State Writer。
2. Agent Invocation、測試、Build 或 Review 期間不持有 State Lock。
3. Lock 只保護讀取最新版、驗證、寫入與替換的短 Critical Section。
4. Current State 不接受局部 In-place 修改。
5. State 引用的 Artifact 必須已完整發布並通過 Schema 與 Integrity 檢查。
6. Artifact 使用唯一 URI，發布後保持 Immutable。
7. Producer 不能更新自己的 Accepted Pointer。
8. State CAS 失敗不能覆寫較新的 Revision。
9. `revision`、`attempt` 與 Artifact Version 各自具有單一用途。
10. Merge 後的整合結果形成新的 Candidate，舊 Validation 與 Review 不直接沿用。
11. Human Approval 綁定固定 Diff／Payload；內容改變後必須重新評估。
12. 外部副作用結果不明時，先向 System of Record 查證。

Last-write-wins 禁止用於 Current State、Gate、Accepted Pointer、Permission 與 Approval。Trace、Metric 或可重建的暫存 Cache 可以依其資料政策採用其他策略。

## 4. Current State Revision 與 CAS

### 4.1 `revision` 的語意

[`current-state.schema.json`](../schemas/current-state.schema.json) 已要求整數 `revision`。本文件將其定義為 Current State Snapshot 的單調遞增版本：

- 建立第一份合法 State 時，`revision` 至少為 1。
- 每次成功的邏輯 State Mutation 增加 1。
- CAS 拒絕、Schema 驗證失敗或 Atomic Replace 失敗時不增加。
- Writer 不能跳號預留 Revision。
- Restore 或人工修復也必須產生新的 Revision，不能倒退。

`attempt` 表示 Agent 工作嘗試次數。Retry Agent 可能增加 `attempt`，但只有成功更新 Current State 才增加 `revision`。

### 4.2 Expected Revision

Writer 必須根據讀取時的 Revision 提交：

```text
expectedRevision = 5
proposedRevision = 6
```

提交當下若 Current State 已是 Revision 6，Store 必須回傳 `STATE_REVISION_CONFLICT`，保留現有內容。Writer 接著重新載入 State、Handoff 與必要 Artifact，再判斷原 Mutation 是否仍成立。

### 4.3 CAS 流程

概念性流程如下：

```text
applyTransition(changeId, expectedRevision, proposedTransition):
  acquireShortStateLock(changeId)

  current = loadAndValidateState(changeId)

  if current.revision != expectedRevision:
    releaseStateLock()
    return STATE_REVISION_CONFLICT

  validateTransition(current, proposedTransition)
  validateGates(current, proposedTransition)
  validateArtifactRefs(proposedTransition)

  next = buildNextState(current, proposedTransition)
  next.revision = current.revision + 1

  atomicReplace(current-state.json, next)
  releaseStateLock()

  return committed(next.revision)
```

實作必須以 `finally` 或同等機制釋放 Lock。Timeout、Process Crash 或 Validation Error 都不能讓 Lock 永久占用。

### 4.4 重新計算條件

CAS 失敗後只有下列條件全部成立，Orchestrator 才可以有限重試：

- 新 State 仍由相同 Change 管理。
- Phase、Owner 與必要 Gate 沒有改變原 Mutation 的前提。
- Target Artifact、Diff 與 Approval Binding 仍相同。
- Retry 不會重複已成功的外部副作用。
- Retry 次數未超過 Policy 上限。

任一語意前提改變時，流程停止並產生 Conflict Result；不能只把 `expectedRevision` 改成新數字後再次送出。

## 5. 本機 Atomic Write 與 Crash Consistency

### 5.1 State 寫入

本機 Current State 應採整份 Snapshot 替換：

```text
1. 讀取並驗證 current-state.json
2. 在相同 Filesystem 建立暫存檔
3. 寫入完整 next state
4. 驗證 JSON 與 Current State Schema
5. Flush 必要資料
6. Atomic Replace current-state.json
7. 記錄 Mutation Receipt／Trace
```

暫存檔必須位於與目標相同的 Filesystem／Volume，避免 Rename 跨裝置後退化成 Copy。平台無法保證 Atomic Replace 時，應改用具備 Transaction 或 CAS 的 Durable Store。

### 5.2 Artifact 與 State 的提交順序

流程必須先發布 Artifact，再更新 State：

```text
write temporary artifact
→ validate schema / checksum / target binding
→ publish immutable artifact URI
→ publish immutable handoff
→ CAS current-state.json
→ record mutation receipt
```

CAS 若失敗，已發布 Artifact 仍是未被 Accepted Pointer 引用的 Candidate，可以由 Retention Job 清理。Current State 維持有效。

以下順序會造成 Dangling Reference，必須禁止：

```text
update current-state.json
→ write referenced artifact
```

Process 在兩步之間終止時，下游 Agent 會讀到不存在或不完整的 Artifact。

### 5.3 Partial Write Recovery

啟動或恢復時，Orchestrator 應檢查：

- `current-state.json` 是否通過 Schema。
- 暫存 State 是否殘留。
- State Reference 指向的 Artifact 是否存在且符合 Checksum。
- 已發布但未被任何 State／Handoff 引用的 Candidate。
- Mutation Receipt 是否顯示前次提交已完成。

無法證明 State 完整時，流程應進入 `FAILED` 或維持安全 Phase 並加入 `MUTATION_COMMIT_INCOMPLETE` Blocker。修復程序不能猜測哪一份暫存檔較新。

## 6. Immutable Candidate 與 Accepted Pointer

### 6.1 Candidate 使用唯一 URI

Producer 每次輸出都應使用唯一 Artifact ID 與 URI，例如：

```text
artifact://feature-example/run-003/implementation-result-attempt-02.json
evidence://feature-example/run-003/validation-attempt-02.json
```

同一 URI 已存在時：

- Checksum 相同且 Idempotency Key 相同，可以回傳既有 Mutation Receipt。
- Checksum 不同時回傳 `ARTIFACT_ID_COLLISION`。
- Producer 不得覆寫、刪除或修改既有 Artifact 來通過 Gate。

### 6.2 `latestArtifacts` 的相容語意

現有 Current State 使用 `latestArtifacts`。在 Schema 尚未加入獨立 `acceptedArtifacts` 前，Orchestrator 必須把 Pointer 解讀為「目前最新可用且已接受的 Artifact」。

```text
accepted implementation = v1

candidate v2
→ validation failed
→ remains candidate

latestArtifacts.implementationResult
→ remains v1
```

檔名、建立時間或 Attempt 較大不會自動更新 Pointer。

### 6.3 Accepted Pointer CAS

Pointer 更新與 Phase、Gate、Owner、Handoff 必須放在同一筆 State Mutation 中，並使用相同 `expectedRevision`。若兩個 Candidate 同時要求接受：

```text
v2 reads revision 8
v3 reads revision 8

v2 commits revision 9
v3 attempts expectedRevision 8
→ STATE_REVISION_CONFLICT
→ accepted pointer remains v2
```

v3 可以保留為 Candidate，等待 Coordinator 比較或建立新的 Review Handoff。Producer 不能自行把 v3 寫入 `latestArtifacts`。

### 6.4 Merge 會產生新 Candidate

兩個已驗證 Candidate 合併後，Diff、Target Binding 與風險都已改變。整合結果必須取得新的 Artifact ID、Diff Hash、Validation Evidence 與 Review Result：

```text
candidate-A + candidate-B
          ↓
integrated-candidate-C
          ↓
combined validation
          ↓
review
          ↓
accepted pointer CAS
```

A、B 的 Review Verdict 只能作為整合參考，不能直接批准 C。

## 7. Source Mutation 與 Worktree Isolation

### 7.1 Single Writer 預設

本機第一版應維持：

- 一個 Implementer 修改一個工作樹。
- Reviewer 對 Source 維持唯讀。
- Coordinator 管理 Scope 與 Path Ownership。
- Orchestrator 序列化 Runtime State Mutation。
- Human 控制 Git Gate。

單一 Writer 仍要保存 Base Commit 或 Source Snapshot Identity，避免 Agent 工作期間人工修改相同工作樹而未被偵測。

### 7.2 平行 Implementer

需要平行施工時，每個 Implementer 必須：

- 使用獨立 Worktree／Branch。
- 固定 `baseCommit` 或等價 Source Identity。
- 取得明確 Path Ownership。
- 只修改批准的 Scope。
- 產生自己的 Candidate、Changed Files、Diff Hash 與 Evidence。
- 不直接合併到 Integration Branch。

Coordinator 應把 Shared Generated Files、Lockfile、Migration、Schema 與共用 Config 指派給單一 Integration Owner。這些檔案即使位於不同 Feature Path，也常受到多個 Candidate 影響。

### 7.3 Base Divergence

提交 Candidate 時，Integration Host 必須比較：

```text
candidate.baseCommit
integration.currentBase
```

兩者不同時回傳 `SOURCE_BASE_DIVERGED`。流程可以建立受控 Rebase／Integration Handoff，但不能默默套用舊 Diff。Rebase 後的 Candidate 必須重新計算 Diff Hash 並執行相關 Validation。

## 8. Controlled Integration

### 8.1 Integration Owner

Integration Owner 可以是受控 Automation、指定 Implementer 或 Human。它負責：

- 驗證所有 Candidate 的 Base Identity。
- 檢查 Changed Files 與 Path Ownership。
- 套用候選變更到隔離的 Integration Worktree。
- 分類文字、結構與語意衝突。
- 產生 Integrated Candidate 與 Combined Evidence。

Integration Owner 不能跳過原 Role Policy、Review Gate 或 Human Gate。

### 8.2 自動整合的最低條件

只有下列條件全部成立，流程才可以嘗試機械式整合：

- Candidate 使用相同或明確相容的 Base。
- Changed Files 沒有未處理的 Ownership 重疊。
- Shared Generated Files 有唯一 Owner。
- 沒有 Requirement、Design、Schema 或 API Contract Conflict。
- Integration Policy 允許該檔案類型自動處理。
- 整合後會重新執行 Validation 與 Review。

Git Clean Merge 只表示文字套用成功。Combined Behavior 仍可能失敗。

### 8.3 衝突後的 Source 保護

Integration 遇到 Conflict 時：

- 不修改原 Candidate Artifact。
- 不在原 Implementer Worktree 直接嘗試多輪未知修正。
- 保存 Base、Candidate、Conflict Path 與 Tool Output Reference。
- 建立新的 Integration／Coordinator Handoff。
- 未完成的整合結果不能進入 Accepted Pointer。

Conflict Resolution 產生新內容時，必須建立新的 Candidate，不能把已審查版本原地改寫。

## 9. Lock、Lease 與 Fencing

### 9.1 依執行環境選擇控制

| 執行環境 | 建議控制 | 適用邊界 |
|---|---|---|
| 單 Process 本機 | In-process Mutex + Revision Check + Atomic Replace | 一個 Orchestrator Process |
| 同機多 Process | OS File Lock + Revision Check + Atomic Replace | 多個 Process 共用本機 Runtime |
| 多機共享 Runtime | Durable Store CAS + Lease + Fencing Token | Network Store 或分散式 Worker |
| 平行 Source Work | Worktree／Branch + Base Identity + Integration Owner | 多個 Implementer |

Lock、Lease 與 Worktree 解決不同問題。State Lock 序列化短期 State Commit；Worktree 隔離長時間 Source Mutation；Lease 管理跨 Process 或跨機的暫時所有權。

### 9.2 Lock 範圍

State Lock 只能覆蓋：

```text
load current state
→ compare revision
→ validate proposed transition
→ atomic replace
→ release lock
```

下列工作不得在 State Lock 內執行：

- LLM Invocation。
- Tool Retrieval。
- Test、Lint 或 Build。
- Review。
- Network Call。
- Human Approval 等待。

長時間持有 Lock 會把 Agent Timeout 變成整個 Change 的阻塞，也使 Crash Recovery 更困難。

### 9.3 Lease Record

跨 Process 或多機環境使用 Lease 時，概念上至少需要：

```text
leaseId
ownerId
resource
acquiredAt
expiresAt
fencingToken
```

Lease Holder 必須在到期前續約。續約失敗後立即停止寫入；恢復的舊 Process 也不能沿用原 Lease。

### 9.4 Fencing Token

只檢查 Lease 時間仍可能讓暫停後恢復的舊 Writer 寫入。Store 應為每次取得 Lease 配發單調遞增的 Fencing Token：

```text
writer A gets token 41
writer A pauses
lease expires

writer B gets token 42
writer B commits

writer A resumes with token 41
→ FENCING_TOKEN_REJECTED
```

Clock-based Lock File 無法單獨提供這項保證。若本機實作只使用 Lock File，必須限制在單機受控 Process，並搭配 Revision Check；跨機時升級到能驗證 Token 的 Store。

## 10. Idempotency 與 Safe Retry

### 10.1 Idempotency Key

每一筆可能重送的 Mutation 應具有 Idempotency Key。Key 應綁定：

```text
changeId
runId
stage
action
resource
baseRevision / baseCommit
payloadHash
```

Key 不應只使用 Agent 名稱或 Timestamp。兩筆語意不同的 Mutation 不能共用同一 Key。

### 10.2 Mutation Receipt

Mutation Host 應保存最小 Receipt：

```text
mutationId
idempotencyKey
requestHash
decision
target
beforeRef
afterRef
committedRevision
sideEffectRef
createdAt
traceId
```

同一 Idempotency Key 再次出現時：

- Request Hash 相同且前次已成功，回傳原 Receipt。
- 前次仍在執行，回傳既有狀態，不啟動第二份工作。
- Request Hash 不同，回傳 `IDEMPOTENCY_KEY_REUSED`。
- 前次結果不明，先執行 Reconciliation。

Receipt 可以先放在 Runtime Trace 或 Host Journal。現行 Schema 沒有相應欄位，不能把 Receipt 任意嵌入 Agent Result 或 Current State。

### 10.3 Retry 分類

| Failure 類型 | 是否可自動 Retry | 條件 |
|---|---:|---|
| 暫時 Lock Contention | 可以 | 有上限、Backoff，且未進入 Critical Section |
| State Revision Conflict | 有條件 | 重新讀取後語意前提仍相同 |
| Network Read Timeout | 可以 | Read Operation 可重複且有 Timeout |
| Validation Failure | 不可以 | 必須先產生修正 Handoff |
| Design／Semantic Conflict | 不可以 | 交回 Coordinator |
| Approval Changed／Expired | 不可以 | 建立新 Approval Request |
| External Write Outcome Unknown | 不可以直接重送 | 先向 System of Record Reconcile |

Retry Policy 必須設定最大次數、Backoff 與終止條件。無限重試會持續消耗資源，也會掩蓋真正的 State 或 Design Conflict。

### 10.4 Unknown Side-effect Outcome

Commit、Push、Issue Update 或外部 API Write 可能在對方成功後發生 Client Timeout。此時 Executor 必須：

1. 使用 Idempotency Key、Commit Hash、Request ID 或 Target Version 查詢 System of Record。
2. 已成功時保存 After Reference，回傳原結果。
3. 明確未執行時才允許 Retry。
4. 無法確認時回傳 `SIDE_EFFECT_OUTCOME_UNKNOWN`，停止自動化。

Agent 不能因為沒有收到成功回應，就假定副作用沒有發生。

## 11. Conflict 分類與固定代碼

### 11.1 Conflict Matrix

| Code | 判定條件 | 預設處理 |
|---|---|---|
| `STATE_REVISION_CONFLICT` | Current State 已不是讀取時的 Revision | 重新讀取並重建 Mutation |
| `STATE_LOCK_UNAVAILABLE` | 無法取得短期 State Lock | 有限重試，超限後 `incomplete` |
| `LEASE_EXPIRED` | Lease 已失效 | 停止寫入，重新取得 Lease |
| `FENCING_TOKEN_REJECTED` | 舊 Writer 使用過期 Token 提交 | 拒絕並保存 Trace |
| `MUTATION_DUPLICATE` | 相同 Mutation 已成功 | 回傳既有 Receipt |
| `IDEMPOTENCY_KEY_REUSED` | 同一 Key 對應不同 Request Hash | `blocked`，交由 Host／Human 檢查 |
| `ARTIFACT_ID_COLLISION` | 相同 URI 已存在不同內容 | 隔離新輸出，禁止覆寫 |
| `MUTATION_COMMIT_INCOMPLETE` | Crash、Partial Write 或 State Integrity 不明 | 維持原 State，啟動 Recovery |
| `SOURCE_BASE_DIVERGED` | Candidate Base 與 Integration Base 不同 | 建立受控 Rebase／Integration Handoff |
| `PATH_OWNERSHIP_CONFLICT` | 多個 Writer 修改同一 Ownership 範圍 | Coordinator 重新分工 |
| `SOURCE_MERGE_CONFLICT` | Git 無法完成文字合併 | Integration Owner 處理 |
| `SEMANTIC_CONFLICT` | 文字可合併但 Requirement、Schema、API 或行為互斥 | `NEEDS_COORDINATOR_ARBITRATION` |
| `ACCEPTED_POINTER_CONFLICT` | 多個 Candidate 同時要求更新 Accepted Pointer | Pointer 不變，重新比較 |
| `GATE_CHANGED` | Review、Approval 或 State Gate 已改變 | 重新執行 Gate 檢查 |
| `SIDE_EFFECT_OUTCOME_UNKNOWN` | Timeout 後無法確認外部寫入結果 | Reconcile，禁止盲目 Retry |

### 11.2 Conflict Result

Conflict Result 或 Trace 至少應保存：

```text
code
changeId
runId
stage
actorId
resource
expectedIdentity
actualIdentity
candidateRefs
safeReason
nextAllowedAction
traceId
```

這是概念格式。若使用現有 Agent Result，角色應採用 `blocked`、`failed` 或 `incomplete` Status，並把固定 Code 放入可驗證的 Payload、Blocker 或 Trace；不能新增 Schema 未定義欄位。

### 11.3 State Conflict 與 Design Conflict

State Revision Conflict 表示寫入前提過期，未必需要人工仲裁。重新載入後若 Transition 仍合法，可以建立新 Mutation。

Design Conflict 表示兩個權威工程決定互斥。Orchestrator 不得挑選其中一方，也不能用 Last-write-wins 處理；流程進入 `NEEDS_COORDINATOR_ARBITRATION`。

## 12. 自動處理與人工仲裁

### 12.1 可以自動處理

以下情況可以由 Orchestrator 或受控 Automation 處理：

- 相同 Idempotency Key 與 Request Hash 的 Duplicate Delivery。
- 短暫 State Lock Contention。
- CAS 失敗後重新讀取，且所有語意前提仍相同。
- 未被 State 或 Handoff 引用的暫存檔與 Orphan Candidate 清理。
- Policy 明確允許的機械式 Source Integration。
- Checksum 相同的重複 Artifact Publication。

自動處理仍要留下 Receipt 或 Trace，並受 Retry 上限與 Retention Policy 約束。

### 12.2 必須停止

以下情況不能由 Agent 自行選擇：

- Requirement、Design、Scope、Schema 或 API Contract 衝突。
- 同一路徑出現不同內容，且沒有明確 Integration Owner。
- Accepted Pointer 競爭需要比較多個合法 Candidate。
- Diff 改變後舊 Review 或 Approval 已不再適用。
- 外部副作用結果無法確認。
- State Integrity、Artifact Integrity 或 Fencing Token 驗證失敗。

Coordinator 負責工程語意仲裁；Human 處理高風險 Approval、Permission 與不可逆副作用。Orchestrator 只執行可重現的驗證與路由。

### 12.3 Conflict Resolution 也要版本化

人工或 Agent 解決 Conflict 後，不得原地修改已發布 Candidate。流程應：

```text
load conflict evidence
→ record resolution decision
→ create new candidate
→ recalculate diff / checksum
→ rerun required validation
→ rerun review
→ attempt accepted pointer CAS
```

Resolution Decision 若改變 Requirement、Design 或 Scope，必須先更新 Durable Knowledge。

## 13. Handoff `onConflict` 與 Schema 相容

### 13.1 現有 Handoff 的使用方式

[`handoff-envelope.schema.json`](../schemas/handoff-envelope.schema.json) 已提供 `onConflict`，但沒有 `expectedRevision`、`baseCommit`、`mutationId`、`leaseId` 或 `fencingToken` 欄位，而且使用 `additionalProperties: false`。現有 Handoff 不能直接加入這些自訂欄位。

第一版本可以使用：

- `requiredInputRefs` 固定 Source、Candidate、Evidence 與 State Reference。
- Artifact Reference 的 `uri` 與 `sha256` 固定內容。
- `description` 說明 Base Identity、Candidate／Accepted 身分與用途。
- `onConflict` 指向合法 Phase 與 Owner。
- Orchestrator Invocation Metadata 攜帶 Expected Revision 與 Idempotency Key。
- Trace／Host Journal 保存 Lock、Lease、Receipt 與 Conflict Detail。

### 13.2 `onConflict` 路由

Handoff 的 `onConflict` 是預先定義的安全路由，不能把所有 Conflict 都解讀為設計仲裁。Orchestrator 應先分類：

| Conflict 類型 | 路由 |
|---|---|
| 暫時 Lock／Revision Contention | 保持安全 Phase，有限重試或 `INCOMPLETE` |
| Source／Evidence 缺失 | `INCOMPLETE`，建立補件 Handoff |
| Design／Scope／Semantic Conflict | `NEEDS_COORDINATOR_ARBITRATION` |
| Integrity／Partial Commit | `FAILED` 或安全 Phase + Blocker |
| Human Approval Reject／Changed | 返回合法 Gate 前階段 |

`onConflict.phase` 在 Handoff Schema 中是一般字串，但寫入 Current State 前仍必須符合 Current State Phase Enum 與 Transition Invariant。

### 13.3 不使用 `FAILED_RETRYABLE`

[`05-security-and-maintainability-zh-TW.md`](./05-security-and-maintainability-zh-TW.md) 的 Failure Handling 曾提到 `FAILED_RETRYABLE`，現行 [`current-state.schema.json`](../schemas/current-state.schema.json) 並未列出該 Phase。採用本文件時不能將它寫入 Current State。

需要重試時使用：

- 保持原 Phase，並記錄 Blocker。
- `INCOMPLETE`。
- `FAILED`。
- `NEEDS_COORDINATOR_ARBITRATION`。

若未來要新增 Phase，必須先升級 Schema、Transition Table、Example Fixture 與 Migration。

## 14. 概念性 Mutation Policy

以下 YAML 用來說明 Mutation 控制。它不符合目前的 `workflow-policy.template.yaml`，也不能嵌入現行 Handoff：

```yaml
schemaVersion: 0.1.0-draft
policyId: local-mutation-policy
policyVersion: 0.1.0

state:
  writerRoles: [automation]
  requireExpectedRevision: true
  incrementRevisionBy: 1
  atomicReplaceRequired: true
  lockScope: commit-critical-section
  maxCasRetries: 1

artifacts:
  publishMode: create-only
  overwrite: deny
  checksumRequired: true
  producerMayAccept: false

source:
  defaultMode: single-writer
  parallelIsolation: worktree
  requireBaseCommit: true
  requirePathOwnership: true
  sharedFilesRequireIntegrationOwner: true

integration:
  createNewCandidate: true
  requireCombinedValidation: true
  requireReviewAfterMerge: true
  automaticMergeRiskMax: low

idempotency:
  requiredForSideEffects: true
  sameKeyDifferentPayload: deny
  reconcileUnknownOutcome: true

conflicts:
  semantic: coordinator-arbitration
  acceptedPointer: coordinator-arbitration
  integrity: fail-closed
```

正式採用此格式前，至少需要：

```text
schemas/mutation-request.schema.json
schemas/mutation-receipt.schema.json
schemas/lease-record.schema.json
templates/mutation-policy.template.yaml
examples/state-revision-conflict/
examples/parallel-worktree-integration/
```

這些資產屬於後續契約升級。本文件的本機導入不依賴它們。

## 15. 本機平行實作範例

### 15.1 分配工作

Coordinator 批准 Change，將兩項工作分配給不同 Implementer：

```text
baseCommit: abc123
currentStateRevision: 5

Implementer A
  worktree: worktrees/payment-service
  ownedPaths:
    - src/payment/**

Implementer B
  worktree: worktrees/payment-tests
  ownedPaths:
    - tests/payment/**
    - package-lock.json
```

`package-lock.json` 是 Shared Generated File，Coordinator 指定 Integration Owner 重新產生，Implementer B 的修改只能作為 Candidate Input。

### 15.2 各自產生 Candidate

```text
candidate-A
  baseCommit: abc123
  changedFiles:
    - src/payment/service.ts
  diffHash: hash-a
  validation: passed

candidate-B
  baseCommit: abc123
  changedFiles:
    - tests/payment/service.test.ts
    - package-lock.json
  diffHash: hash-b
  validation: passed
```

A、B 都是 Candidate。它們尚未進入 `latestArtifacts.implementationResult`。

### 15.3 Integration

Integration Owner：

1. 驗證兩個 Candidate 使用 `abc123`。
2. 套用 `src/payment/service.ts` 與 Test Change。
3. 根據整合後的 Dependency Graph 重新產生 `package-lock.json`。
4. 產生 `integrated-candidate-C`、新 Diff Hash 與 Changed Files。
5. 執行 Combined Test、Lint、Build 與必要 Validation。
6. 建立 Review Handoff，固定 C 與其 Evidence。

若 Lockfile 無法重建，流程回傳 `PATH_OWNERSHIP_CONFLICT` 或 `SOURCE_MERGE_CONFLICT`；A、B 的原 Candidate 保持不變。

### 15.4 Review 與 Pointer Update

Reviewer 核准 C 後，Orchestrator 準備從 Revision 5 更新：

```text
expectedRevision: 5
acceptedCandidate: integrated-candidate-C
nextPhase: READY_FOR_READINESS_CHECK
nextOwner: readiness
```

若另一個合法 Transition 已把 State 更新為 Revision 6，C 的提交回傳 `STATE_REVISION_CONFLICT`。Orchestrator 重新讀取 Revision 6；只有 Review、Gate、Phase、Owner 與 Target Binding 仍相容，才建立新的 Pointer Mutation。

### 15.5 Human Git Gate

Readiness 與 Archive 完成後，Approval Record 綁定 C 的 Diff Hash。若後續 Conflict Resolution 又產生 Candidate D，C 的 Approval 不適用於 D。Human 必須查看 D 的 Evidence 並建立新 Approval。

## 16. 最小本機導入

不修改現有 Schema，也能採用本文件的大部分規則：

1. 每個 Change 只允許一個 Orchestrator／Host 更新 `current-state.json`。
2. 將 `revision` 定義為每次成功 State Mutation 增加 1。
3. State Update API 要求呼叫端帶入 `expectedRevision`。
4. 使用同 Filesystem 暫存檔、Schema Validation 與 Atomic Replace。
5. Agent 執行期間不持有 State Lock。
6. Artifact 使用唯一 URI、Create-only 與 Checksum；禁止覆寫。
7. `latestArtifacts` 只指向 Accepted Artifact。
8. Producer 只建立 Candidate，由 Orchestrator 更新 Pointer。
9. 單一 Implementer 使用固定 Base Commit；平行時使用獨立 Worktree。
10. Path Ownership 與 Shared File Owner 寫入已批准的 Change Scope。
11. Merge 後建立 Integrated Candidate，重新執行 Validation 與 Review。
12. Side-effect Tool 使用 Idempotency Key 與 Before／After Reference。
13. Conflict Code 放入 Blocker 或 Trace，路由到現有合法 Phase。
14. Timeout 後的外部 Write 先 Reconcile，再決定是否 Retry。

### 16.1 可選本機目錄

第一版可以在 Git-ignored Runtime 中加入 Host 專用區：

```text
.agent-runtime/<change-id>/
├─ current-state.json
├─ runs/
│  └─ <run-id>/
│     ├─ artifacts/
│     └─ evidence/
├─ locks/
│  └─ current-state.lock
└─ host-journal/
   └─ mutation-receipts/
```

`locks/` 與 `host-journal/` 是 Runtime Implementation Detail，不是 Agent Context。Agent 不應直接修改或依賴其中內容。

### 16.2 本機 Commit Critical Section

最小 Orchestrator 應把下列操作封裝成一個受控函式：

```text
commitStateMutation(
  changeId,
  expectedRevision,
  proposedTransition,
  acceptedArtifactRefs,
  idempotencyKey
)
```

該函式負責 Lock、Revision Compare、Schema Validation、Transition Validation、Atomic Replace 與 Receipt。Agent Adapter 不能自行拼接或覆寫 `current-state.json`。

## 17. Durable Store 升級判準

符合任一條件時，應評估將 Current State 與 Mutation Receipt 移到支援 Transaction／CAS 的 Durable Store：

- 多個 Orchestrator Process 可能處理同一 Change。
- Worker 分布在多台機器。
- Runtime 位於 Network Filesystem。
- 需要 Lease、Fencing Token 或跨機 Failover。
- 必須保證 Mutation Receipt 與 State Update 的交易一致性。
- 外部副作用需要可靠 Outbox／Inbox。
- 稽核要求重建每一次 State Transition。
- Parallel Run 數量已使本機 Lock 與 Journal 難以管理。

升級後仍應保留相同語意：

```text
expected revision
conditional write
immutable candidate
accepted pointer
idempotency receipt
conflict classification
```

更換 Storage 不能改變 Agent 對 Candidate、Accepted、Gate 與 Role Boundary 的理解。

### 17.1 不需升級的情況

以下條件下，本機檔案模式通常足夠：

- 一個 Orchestrator Process。
- 一個 Change 只有一個 Source Writer。
- Runtime 不跨機共享。
- Commit、Push 與 Deploy 保留 Human Gate。
- 可接受 Crash 後由 Host 執行有限 Recovery。

此時 In-process Mutex、Revision Check、Atomic Replace 與 Immutable Artifact 已能提供清楚邊界。

## 18. 驗收清單

### State Mutation

- [ ] `revision` 每次成功 State Mutation 只增加 1。
- [ ] `attempt` 不作為 CAS Token。
- [ ] Writer 提交時帶入讀取時的 Expected Revision。
- [ ] CAS 失敗不覆寫較新的 State。
- [ ] Current State 採完整 Snapshot 與 Atomic Replace。
- [ ] Agent Invocation、Validation 與 Human Wait 不持有 State Lock。

### Artifact 與 Pointer

- [ ] Candidate 使用唯一 URI、Checksum 與 Create-only Publication。
- [ ] 已發布 Artifact 不接受 In-place 修改。
- [ ] State 只引用已完整發布且通過 Integrity 檢查的 Artifact。
- [ ] `latestArtifacts` 維持 Accepted Pointer 語意。
- [ ] Producer 不能更新 Accepted Pointer。
- [ ] Pointer、Gate、Phase、Owner 與 Handoff 使用同一 State CAS。

### Source 與 Integration

- [ ] 每個 Implementer 有固定 Base Identity 與 Path Ownership。
- [ ] 平行 Implementer 使用獨立 Worktree／Branch。
- [ ] Shared Generated File 有唯一 Integration Owner。
- [ ] Base Divergence 不會被默默忽略。
- [ ] Merge 後建立新的 Integrated Candidate。
- [ ] Integrated Candidate 重新執行 Combined Validation 與 Review。

### Lock、Lease 與 Retry

- [ ] Lock 只覆蓋短 Commit Critical Section。
- [ ] 多機 Lease 使用 Expiration 與 Fencing Token。
- [ ] Side-effect Mutation 有 Idempotency Key。
- [ ] 相同 Key 搭配不同 Payload 時拒絕執行。
- [ ] Retry 有上限、Backoff 與終止條件。
- [ ] Unknown Side-effect Outcome 先查詢 System of Record。

### Conflict 與 Recovery

- [ ] State、Source、Semantic、Pointer 與 Side-effect Conflict 分類處理。
- [ ] Conflict Result 有固定 Code 與 Trace ID。
- [ ] Design／Scope Conflict 交由 Coordinator 仲裁。
- [ ] Diff 改變後不沿用舊 Review 或 Approval。
- [ ] Partial Write 不會留下 Dangling State Reference。
- [ ] Orphan Candidate 有 Retention／Cleanup Policy。

### Schema 相容性

- [ ] 現行 Handoff 沒有加入未定義的 Mutation 欄位。
- [ ] Expected Revision 與 Idempotency Key 由 Orchestrator Metadata／Journal 管理。
- [ ] `onConflict` 目標通過 Current State Phase 與 Transition 驗證。
- [ ] 不把 `FAILED_RETRYABLE` 寫入現行 Current State。
- [ ] 未來 Schema 資產在正式採用前有 Version 與 Migration 規則。

## 19. 參考文件

- [Artifact-based Shared State + Structured Handoff 參考架構](./02-reference-architecture-zh-TW.md)
- [Repository Knowledge、Runtime State、Evidence 與 Trace 分層](./03-runtime-storage-and-retention-zh-TW.md)
- [安全、治理與長期維護](./05-security-and-maintainability-zh-TW.md)
- [Context Authority 與 Retrieval Policy](./06-context-authority-and-retrieval-policy-zh-TW.md)
- [Role Capability 與 Scope Control](./07-role-capability-and-scope-control-zh-TW.md)
- [Agent Platform Operations](../../../context-engineering/docs/03-agent-platform-operations-zh-TW.md)
- [Tool Governance and Evaluation](../../../agent-design/tool-schema-routing/docs/03-tool-governance-and-evaluation-zh-TW.md)
- [Handoff Envelope Schema](../schemas/handoff-envelope.schema.json)
- [Current State Schema](../schemas/current-state.schema.json)
- [Agent Result Schema](../schemas/agent-result.schema.json)
- [Workflow Policy Template](../templates/workflow-policy.template.yaml)
