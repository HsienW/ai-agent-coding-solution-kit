# 10｜Provenance Audit 與 Incident Replay

[English](./10-provenance-audit-and-incident-replay.md) | [繁體中文](./10-provenance-audit-and-incident-replay-zh-TW.md)

本文件規範多 Agent Workflow 如何記錄 Provenance、重建 Audit 因果鏈，並在事故發生後執行受控的 Incident Replay。適用對象包括 Agent、Coordinator、Orchestrator、Reviewer、Runtime Store 維護者與 Human Approver。

Provenance 應讓調查者回答以下問題：哪個執行者在什麼授權下，讀取哪些固定版本的輸入，執行什麼 Action，產生哪個 Artifact 與 Evidence，最後由誰依哪個 Policy 推動 State Transition。Incident Replay 使用這條因果鏈重算決策或重跑驗證，同時保護原始 Run 與外部系統。

本文件使用三項原則：

- Audit 保存足以重建決策的穩定事實，避免收錄完整 Prompt、私有推理與未篩選的輸出。
- Replay 預設在隔離環境執行，並阻擋 Commit、Push、Deploy、Delete 與外部 API Write。
- 現行 Schema 維持不變。額外紀錄放入 Trace、Host Journal 或獨立 Artifact。

## 1. 適用範圍

本文件接續既有規範：

- [`03-runtime-storage-and-retention-zh-TW.md`](./03-runtime-storage-and-retention-zh-TW.md) 定義 Durable Knowledge、Runtime State、Evidence 與 Trace 的保存層級。
- [`05-security-and-maintainability-zh-TW.md`](./05-security-and-maintainability-zh-TW.md) 定義 Evidence Provenance、Artifact Integrity、Human Gate 與 Threat Model。
- [`06-context-authority-and-retrieval-policy-zh-TW.md`](./06-context-authority-and-retrieval-policy-zh-TW.md) 定義 Context Authority、Version、Freshness、Integrity 與 Binding。
- [`07-role-capability-and-scope-control-zh-TW.md`](./07-role-capability-and-scope-control-zh-TW.md) 定義 Runtime Identity、Capability、Policy Version 與 Authorization Trace。
- [`08-mutation-concurrency-and-conflict-resolution-zh-TW.md`](./08-mutation-concurrency-and-conflict-resolution-zh-TW.md) 定義 Mutation Identity、Receipt、Idempotency 與 Side-effect Reconciliation。
- [`09-context-promotion-and-rollback-zh-TW.md`](./09-context-promotion-and-rollback-zh-TW.md) 定義 Promotion、Rollback、Terminal Run 與 Cross-run Binding。

本文件處理：

- Agent Invocation、Context Retrieval、Tool Action、Artifact Publication、Validation、Review、Approval 與 State Transition 的 Provenance。
- 授權、接受、拒絕、Promotion、Rollback 與外部副作用的 Audit Reconstruction。
- Decision Replay、Validation Replay、Simulation Replay 與 Effect Reconciliation。
- Incident Evidence Freeze、Replay Isolation、Result Comparison、Retention 與 Redaction。
- 本機 `.agent-runtime` 的最小導入方式，以及 Durable Store 的升級條件。

下列內容維持在其他規範：

- Current State 的合法 Phase 與 Transition Table。
- Role、Capability、Path 與 Tool Scope 的授權模型。
- Accepted Pointer 的 CAS 與 Mutation Critical Section。
- Production Deployment、Database Recovery、Traffic Shift 與組織層級的 Incident Management 流程。

## 2. 名詞與操作邊界

### 2.1 Provenance

Provenance 是一組可驗證的來源、身分與因果關係。它連結輸入、執行、輸出、Evidence、Decision 與 State Mutation。

一筆 Artifact 的 Provenance 至少要能指出：

```text
producer identity
changeId / runId / stage
input references and checksums
source base or state revision
output reference and checksum
verification evidence
policy and authorization decision
parent event or decision reference
```

按時間排列的 Log 只能表示「先後看起來如此」。Revision、Parent Reference 與明確的因果 Edge 才能證明某個 Decision 使用哪份輸入。

### 2.2 Audit Record

Audit Record 保存重建決策所需的穩定事實。它應支援機器查詢與完整性驗證，並採取比 Full Trace 更嚴格的存取與保存規則。

Audit Record 可以保存正規化後的 Tool Action、Policy Version、State Revision、Outcome、Reason Code 與 Reference。它不能保存 Secret、Credential、Session Cookie、Approval Proof 原文或模型私有推理。

### 2.3 Trace

Trace 保存診斷細節，例如 Adapter Timing、Retry、截斷後的 Tool Output、Token 使用量與內部錯誤。Trace 可以採用較短 Retention，也可以在保留 Audit Record 與必要 Evidence 後獨立清除。

### 2.4 Evidence

Evidence 支持一項可驗證聲明，例如：

- 某個 Command 在指定 Working Directory 執行成功。
- 某份 Diff 通過指定版本的測試。
- Reviewer 針對固定 Checksum 的 Candidate 給出 Verdict。
- System of Record 顯示外部副作用已完成。

Evidence 必須綁定被驗證的 Target。只有測試名稱或 `passed` 字樣，無法建立有效 Binding。

### 2.5 Incident Replay

Incident Replay 依照固定的原始輸入、Policy、State Revision 與環境條件，重算決策或重跑允許的驗證。Replay 產生新的 Replay Result，原始 Artifact、Audit Record 與 Terminal Run 維持不變。

### 2.6 Re-execution 與 Reconciliation

Re-execution 會再次執行具有真實效果的 Action。它需要新的 Run、Capability、Gate、Idempotency Key 與必要的 Human Approval。

Reconciliation 查詢 System of Record，確認原始 Side-effect 是否完成。Timeout 或連線中斷後，Orchestrator 先執行 Reconciliation，再判斷後續路由。

## 3. 角色與責任

| 角色 | 責任 | 禁止事項 |
|---|---|---|
| Coordinator | 定義 Incident 問題、Scope、Replay Mode、Comparison Policy 與修正方向 | 不修改原始 Audit Record；不以猜測補齊缺失 Provenance |
| Orchestrator | 固定 Reference、建立 Replay Manifest、驗證完整性、配置隔離環境、套用 Policy 與寫入 Host Journal | 不覆寫 Terminal Run；不把 Replay 當成既有 Invocation 的 Retry |
| Agent | 依 Handoff 讀取固定輸入，執行授權內的分析或驗證，產生新 Result | 不擴大 Context／Tool Scope；不自行啟用外部 Write |
| Reviewer／Auditor | 唯讀檢查 Provenance Chain、Evidence Binding 與 Replay 差異 | 不更新 Current State、Accepted Pointer 或 Approval |
| Runtime Store | 保存 Immutable Artifact、Audit Record、Receipt 與 Retention Metadata | 不允許 Create-only Record 被原地改寫 |
| Human／Incident Operator | 批准敏感資料存取、Live Re-execution、外部 Side-effect 與高風險修正 | 不以口頭同意取代可驗證 Approval Record |

`Incident Operator` 是事故期間的職責名稱，不代表現行 Current State Schema 新增 Role。實際 Runtime Identity 仍要通過 07 的 Policy 與 Capability 檢查。

## 4. Provenance Invariant

每個 Workflow Host 必須遵守以下條件：

1. Stable Provenance Record 採 Append-only，修正時建立新 Record 並引用被取代的 Record。
2. Artifact Reference 必須固定 URI 與 Checksum；內容不同時使用新 URI。
3. 每個 Agent Result 必須能追溯 Producer、Change、Run、Stage 與必要 Input Reference。
4. Validation Evidence 必須綁定 Candidate、Source Base、Command 與執行環境。
5. Review Verdict 必須綁定已審查 Candidate 與 Evidence。
6. State Transition 必須引用 Authorization Decision、Gate Decision 或 Mutation Receipt。
7. `current-state.json` 保存目前投影，不承擔完整事件歷史。
8. Timestamp 只提供輔助排序；因果順序由 Revision、Parent Reference 與明確 Binding 決定。
9. Replay 產生新的 Artifact 與 Trace，不修改原始 Run。
10. Replay 預設拒絕具有外部副作用的 Tool Action。
11. Provenance 缺口、Checksum 不符或 Policy Version 遺失時，流程 fail closed。
12. Audit Record 不保存 Secret、Credential、完整敏感 Arguments 與模型私有推理。
13. Terminal Run 維持封閉；後續調查使用新的 Incident／Replay Identity。
14. Retention 到期造成必要輸入消失時，系統要回報缺口，不能捏造替代資料。

Last-write-wins 不適用於 Provenance Record、Audit Decision、Mutation Receipt、Approval 或 Replay Result。

## 5. Provenance Graph

### 5.1 節點

Provenance Graph 可以包含下列節點：

| 節點 | 內容 |
|---|---|
| Contract | Requirement、Proposal、Design、Task 或 Acceptance Criteria |
| State Snapshot | 指定 Revision 的 Current State |
| Handoff | 固定角色、Stage、Input Reference 與 Transition |
| Invocation | 一次受 Orchestrator 管控的 Agent 執行 |
| Context Resolution | 實際載入的 Context、Authority、Version 與 Binding |
| Tool Action | Tool、Action、Resource、Arguments Hash 與 Side-effect Class |
| Artifact | Plan、Implementation、Review、Readiness 或 Archive Result |
| Evidence | Command、Diff、測試、掃描或外部觀測結果 |
| Decision | Authorization、Review、Gate、Promotion、Rollback 或 Arbitration |
| Mutation Receipt | State、Pointer、Source 或外部 Side-effect 的提交結果 |
| Approval | Human 對固定 Target、Scope、Revision 與期限的授權 |
| External Observation | 從 System of Record 取得的外部狀態 |

### 5.2 因果 Edge

Host 應使用固定 Vocabulary，避免以自由文字表達關係：

| Edge | 語意 |
|---|---|
| `derivedFrom` | Output 由指定 Input 推導 |
| `consumed` | Invocation 實際讀取指定 Context／Artifact |
| `produced` | Invocation 或 Tool Action 產生指定 Output |
| `validated` | Evidence 驗證指定 Target |
| `reviewed` | Verdict 審查指定 Candidate |
| `authorizedBy` | Action 由指定 Policy／Approval 授權 |
| `applied` | Decision 已透過 Mutation 提交 |
| `observed` | Record 來自 System of Record 查詢 |
| `superseded` | 新 Record 取代舊 Record 的有效地位 |
| `compensated` | 新 Mutation 補償先前已提交的效果 |
| `replayOf` | Replay Record 指向原始 Record |

Edge 必須保存來源與目標 Reference。`createdAt` 接近無法證明兩者具有因果關係。

### 5.3 最小閉合鏈

一項 Accepted Implementation 至少要形成：

```text
approved requirement / design
  → handoff
  → authorization decision
  → source base + resolved context
  → implementation candidate
  → validation evidence
  → review verdict
  → gate decision
  → accepted pointer mutation receipt
  → current state revision
```

任何必要節點缺失時，Audit 要回報具體缺口。Auditor 不能因最終 Current State 顯示成功，就推定中間 Gate 合法。

## 6. 最小 Provenance Record

下列 YAML 是概念格式，用於說明 Host Journal 的最小資料。它不是現行 JSON Schema：

```yaml
schemaVersion: 0.1.0-draft
eventId: evt-review-004
eventType: review-decision
changeId: feature-example
runId: run-004
stage: review-result

actor:
  actorId: reviewer-worker-02
  role: reviewer

binding:
  stateRevision: 8
  policyVersion: reviewer-policy-1.2.0
  handoffRef: runtime://feature-example/run-004/handoffs/review-004.json
  parentEventIds:
    - evt-validation-004

inputs:
  - uri: artifact://feature-example/run-004/implementation-v2.json
    sha256: xxxx
  - uri: evidence://feature-example/run-004/validation-v2.json
    sha256: xxxx

decision:
  outcome: approve
  reasonCode: REVIEW_CRITERIA_SATISFIED

outputRef: artifact://feature-example/run-004/review-v2.json
resultHash: sha256:xxxx
occurredAt: 2026-08-14T10:12:03Z
recordedAt: 2026-08-14T10:12:04Z
traceId: trace-review-004
```

### 6.1 Identity

下列 Identity 必須分開：

- `eventId`：單一 Provenance Event。
- `traceId`：跨多個 Event 的診斷關聯。
- `changeId`：工程變更範圍。
- `runId`：一次 Workflow Run。
- `invocationId`：一次 Agent Invocation。
- `mutationId`：一次受控 Mutation。
- `incidentId`：一次事故調查。
- `replayId`：一次受控 Replay。

共用一個 ID 會使 Retry、Cross-run 與多次 Replay 無法區分。

### 6.2 Tool Action Record

Tool Action 應額外保存：

```text
toolId
toolVersion
action
normalizedResource
normalizedArgumentsHash
capability
policyVersion
environmentRef
sideEffectClass
idempotencyKey, when required
beforeRef / afterRef, when applicable
decision and reasonCode
```

敏感 Arguments 經正規化與 Redaction 後再計算 Hash。Audit Store 不保存可還原 Secret 的內容。

## 7. Lifecycle Capture Point

Orchestrator 應在下列位置留下 Provenance：

| Capture Point | 必要紀錄 |
|---|---|
| Handoff Issued | From、To、Stage、固定 Input Ref、Transition、期限 |
| Invocation Authorized | Runtime Identity、Capability、Resource、Policy Version、State Revision、Decision |
| Context Resolved | 實際載入的 URI、Checksum、Authority、Freshness、Binding |
| Tool Action Started | Tool、Action、Arguments Hash、Side-effect Class、Idempotency Key |
| Tool Action Completed | Status、Output Ref、Before／After Ref、System of Record Ref |
| Artifact Published | Producer、Input Ref、Output URI、Checksum、Schema Version |
| Validation Completed | Command、Working Directory、Target、Exit Code、Evidence Ref |
| Review Decided | Candidate Ref、Evidence Ref、Verdict、Reason Code |
| Gate Decided | Gate Type、Target Binding、Decision Owner、Approval Ref |
| State Committed | Expected Revision、Applied Revision、Mutation Receipt、Accepted Pointer |
| External Effect Observed | Query Target、Observation Time、Observed Identity、Result Ref |

Host 應先保存輸出 Artifact，再提交指向該 Artifact 的 State Mutation。Crash 發生於兩者之間時，08 的 Recovery 流程負責辨識 Orphan Candidate 或重建 State。

## 8. Audit Reconstruction

### 8.1 調查問題

Audit Bundle 應讓 Reviewer 或 Incident Operator 回答：

1. 哪個 Runtime Identity 發起 Action？
2. 它當時具備哪個 Role、Capability 與 Effective Scope？
3. Orchestrator 使用哪個 Policy Version 與 State Revision 判定授權？
4. Agent 實際讀取哪些 Context，而非 Handoff 原先列出但未讀取的項目？
5. Candidate 綁定哪個 Source Base、Diff 與 Input Checksum？
6. Validation 與 Review 是否指向同一 Candidate？
7. 哪個 Gate 或 Approval 允許 Accepted Pointer／外部狀態改變？
8. Mutation 是否成功提交？System of Record 顯示什麼結果？
9. 原始執行與 Replay 的輸入、環境及結果有哪些差異？

### 8.2 Audit Bundle

Audit Bundle 是事故範圍內的 Reference Manifest。它不複製所有內容：

```yaml
schemaVersion: 0.1.0-draft
bundleId: audit-bundle-REG-204-01
incidentId: REG-204
sourceChangeId: feature-example
sourceRunId: run-004

scope:
  fromEventId: evt-implementation-004
  toEventId: evt-pointer-accept-004

records:
  - ref: runtime://feature-example/host-journal/provenance/evt-implementation-004.json
    sha256: xxxx
  - ref: runtime://feature-example/host-journal/provenance/evt-review-004.json
    sha256: xxxx

missingRecords: []
integrityStatus: verified
createdAt: 2026-08-14T11:00:00Z
```

Bundle 只收錄與調查問題有因果關係的 Record。完整 Trace 只有在定位缺口、重建 Tool Failure 或釐清時間線時才加入。

### 8.3 Audit 完成條件

Audit 結論必須包含：

- 調查範圍與已固定的原始 Reference。
- Provenance Chain 是否閉合。
- Integrity 與 Retention 缺口。
- 已確認事實、尚未確認項目與證據來源。
- 造成錯誤 Decision 或 Mutation 的最早可證明節點。
- 後續 Replay、Rollback、Correction 或人工處理建議。

缺少 Evidence 時，結論只能標示 `insufficient-evidence`。Auditor 不得把推測寫成已確認事實。

## 9. Integrity、Ordering 與 Clock

### 9.1 Content Integrity

Stable Record 與 Artifact 應使用 Checksum。Host 驗證：

```text
reference exists
schema or record format is readable
checksum matches
changeId / runId binding matches
producer identity is allowed
parent references exist
retention state permits use
```

Checksum 只能證明內容未變。它不能單獨證明 Producer 身分、授權或紀錄時間。

### 9.2 Ordering

排序依據採以下優先順序：

1. Current State `revision` 與 Mutation `committedRevision`。
2. `parentEventIds` 與明確 Provenance Edge。
3. Tool／Host 提供的 monotonic sequence 或 Fencing Token。
4. `occurredAt` 與 `recordedAt`。

不同 Host 的 Clock 可能偏移。Incident Timeline 可以顯示 Timestamp，但不能只靠 Timestamp 宣告因果順序。

### 9.3 Tamper Evidence

本機版本可採 Create-only File、Checksum 與嚴格 Directory Permission。下列需求出現時，再升級 Hash Chain、Digital Signature、Merkle Tree 或 WORM Storage：

- 必須證明 Host Journal 沒有遭到刪除或重排。
- Audit Record 需要跨組織交換。
- 法遵要求獨立時間戳或簽章。
- 多 Host 寫入需要全域 Sequence。

## 10. Retention、Redaction 與存取控制

### 10.1 Retention Class

| 資料 | 建議保存原則 |
|---|---|
| Current State | 保存 Latest Snapshot；必要時由 Journal 重建 |
| Minimal Audit Record | 依工程風險、法遵與事故調查需求保存 |
| Approval／Mutation Receipt | 至少涵蓋對應 Change 與外部效果的追溯期間 |
| Evidence | 涵蓋 Review、Readiness、Rollback 與事故觀察期 |
| Full Trace | 採較短期限；保留必要 Audit Reference 後可清除 |
| Replay Artifact | 保存至 Incident 結案及修正驗證完成 |

Retention Policy 必須記錄 Version。Incident 宣告後，Orchestrator 可以對相關 Record 設定 Legal Hold 或 Incident Hold；Hold 只能由具備授權的 Operator 解除。

### 10.2 Redaction

Audit、Trace 與 Replay Bundle 必須排除：

- API Key、Token、Cookie、Password 與 Private Key。
- `.env`、Credential Store 或 Secret Manager 的完整內容。
- Approval Proof 的可重放原文。
- 未遮蔽的個人資料與任務無關檔案。
- 模型私有推理與供應商內部推理資料。

需要證明某個敏感值參與 Action 時，保存 Redacted Identifier、Secret Version 或不可逆 Hash，並限制 Hash 的可推測輸入空間。

### 10.3 Access Scope

Audit 權限不等於 Runtime 全目錄讀取權。Orchestrator 仍要套用 07 的 Effective Scope：

```text
platform policy
∩ adapter policy
∩ role policy
∩ incident scope
∩ handoff scope
∩ approval scope
```

Reviewer 可以取得調查所需 Evidence，不能列舉其他 Change、Secret Store 或無關的 Full Trace。

## 11. Incident Declaration 與 Evidence Freeze

### 11.1 宣告 Incident

Coordinator 或具授權的 Incident Operator 建立新的 `incidentId`，並記錄：

```text
incident question
affected changeId / runId
observed symptom
observation source
suspected time range
required retention hold
investigation owner
```

Incident Record 不得寫回原始 Terminal Run。它透過 Cross-run Reference 指向原始 Change、Run、Artifact 與 State Revision。

### 11.2 Freeze Reference

Orchestrator 在 Replay 前固定：

- 原始 Current State Revision 或可驗證 Snapshot。
- Handoff、Agent Result、Evidence、Review 與 Approval Reference。
- Source Commit、Worktree Base、Diff Hash 與 Accepted Pointer。
- Policy、Adapter、Tool Registry 與環境版本。
- 外部觀測的 System of Record Reference。

Freeze 保護 Reference 與 Retention，不暫停整個 Repository。業務流程需要繼續時，Coordinator 可以開新 Change；兩者必須使用明確 Cross-run Binding。

### 11.3 Freeze 失敗

必要 Record 已遺失、過期或 Checksum 不符時，Orchestrator 停止 Replay，回傳固定 Failure Code。Operator 可以調整調查範圍或接受 `insufficient-evidence` 結論，不能用 Latest Artifact 取代缺失的原始版本。

## 12. Replay Mode

### 12.1 Decision Replay

Decision Replay 以固定的 State Revision、Policy Version、Context Manifest 與 Artifact Reference，重新計算：

- Authorization Decision。
- Context Authority／Freshness 判定。
- Review／Gate 的機械式 Eligibility。
- Promotion 或 Rollback Precondition。
- State Transition 是否符合 Transition Table。

此模式不呼叫具有副作用的 Tool，也不更新 Current State。它適合調查錯誤路由、過期 Evidence、Target Binding Mismatch 與 Policy Drift。

### 12.2 Validation Replay

Validation Replay 在隔離 Worktree 或 Sandbox 重新執行測試、Lint、Build、Scanner 或唯讀查詢。Orchestrator 必須固定 Source Base、Dependency Lock、Command、Working Directory 與環境版本。

Validation Replay 產生新的 Evidence，並以 `replayOf` 關係指向原始 Evidence。新 Evidence 不能直接取代原始 Run 的 Validation Result。

### 12.3 Simulation Replay

Simulation Replay 使用 Stub、Recorded Response、Read-only Mirror 或 Dry-run Adapter 取代外部 Write。適用於：

- 重建 Tool Routing 與 Arguments Normalization。
- 驗證 Idempotency Key、Scope 與 Approval 判定。
- 比較外部 Response 變動造成的 Agent 行為差異。

Recorded Response 必須保留取得時間、來源、完整性與 Redaction 狀態。過期資料只能用來重建當時決策，不能當成目前的 authoritative Context。

### 12.4 Effect Reconciliation

Effect Reconciliation 對 System of Record 執行受控查詢：

```text
query by idempotency key or external reference
verify target identity
compare before / after reference
classify committed, absent, partial, or unknown
record observation time and source
```

查詢結果為 `unknown` 時，Orchestrator 回傳 `SIDE_EFFECT_OUTCOME_UNKNOWN` 並停止自動處理。

### 12.5 Live Re-execution

需要再次執行 Commit、Push、Deploy、Delete 或外部 Write 時，Coordinator 建立新的 Change／Run 或受治理的補償流程。Orchestrator 重新檢查 Capability、Scope、Gate、Approval、Idempotency 與 Current State Revision。

Live Re-execution 不使用 Replay 身分提交外部效果，也不能沿用已失效的 Approval。

## 13. Replay Manifest

Replay Manifest 固定本次調查的輸入與執行限制。下列 YAML 是概念格式，不屬於現行 Schema：

```yaml
schemaVersion: 0.1.0-draft
incidentId: REG-204
replayId: replay-REG-204-01
sourceChangeId: feature-example
sourceRunId: run-004
mode: decision-replay

sourceBinding:
  stateRevision: 9
  sourceCommit: git:def456
  policyVersion: promotion-policy-1.0.0
  adapterVersion: local-agent-adapter-0.8.0
  environmentRef: runtime://feature-example/incidents/REG-204/environment.json

requiredRefs:
  - uri: artifact://feature-example/run-004/implementation-v2.json
    sha256: xxxx
  - uri: evidence://feature-example/run-004/validation-v2.json
    sha256: xxxx
  - uri: artifact://feature-example/run-004/review-v2.json
    sha256: xxxx

blockedSideEffects:
  - git.commit
  - git.push
  - deployment.write
  - external.delete

comparisonPolicy:
  mode: semantic-and-binding
  allowedDrift:
    - timestamp
    - traceId

createdBy: incident-coordinator-01
createdAt: 2026-08-14T11:20:00Z
traceId: trace-replay-REG-204-01
```

### 13.1 Manifest 驗證

Orchestrator 必須在 Invocation 前檢查：

1. `incidentId`、Source Change 與 Source Run Binding。
2. Required Reference 是否存在且 Checksum 相符。
3. Policy、Adapter、Tool 與環境版本是否可取得。
4. Replay Mode 是否允許 Handoff 指定的 Action。
5. Side-effect Denylist 是否由 Sandbox 與 Adapter 落實。
6. 執行者是否具備 Incident Scope 與資料讀取權限。

任一檢查失敗時，Agent Invocation 不得開始。

## 14. Environment Binding

### 14.1 必要 Binding

Replay 的可重現程度取決於已固定的環境資料：

| 維度 | 建議紀錄 |
|---|---|
| Source | Commit、Branch／Worktree Base、Diff Hash、Submodule Revision |
| Dependency | Lockfile Hash、Package／Container Digest、Runtime Version |
| Agent | Adapter Version、Role、Stage、Prompt／Template Hash |
| Model | Provider、Model Identifier、可取得的 Version 與推論設定 |
| Tool | Tool ID、Schema Version、Binary／Service Version |
| Policy | Authorization、Retrieval、Mutation、Promotion Policy Version |
| Runtime | OS、Architecture、Locale、Timezone、Environment Allowlist |
| External Context | Source、Query、Observed At、Freshness、Recorded Response Ref |

無法取得精確版本時，Manifest 要標示缺口及其影響。Orchestrator 不能以目前版本冒充原始版本。

### 14.2 Determinism 邊界

相同 Prompt、Model Identifier 與參數不保證 LLM 逐字輸出一致。外部 API、搜尋索引、套件 Registry、時鐘與隨機來源也會產生差異。

Replay 的 Comparison Policy 應依問題選擇：

- Byte-level：適合 Checksum、純函式輸出與固定 Fixture。
- Structural：比較 Schema、欄位、Reference 與 Reason Code。
- Semantic：比較 Verdict、風險分類與建議路由。
- Evidence-based：比較測試、Source Identity 與 System of Record 結果。

需要逐位元重現時，Host 必須固定所有非決定性來源；無法固定的來源應記入 Reproducibility Limitation。

## 15. Replay 執行流程

```text
declare incident
  → freeze source references
  → build audit bundle
  → verify integrity and retention
  → select replay mode
  → create replay manifest
  → authorize replay invocation
  → prepare isolated environment
  → execute replay
  → publish replay result
  → compare original and replay
  → classify divergence
  → route correction / rollback / closure
```

### 15.1 Prepare

Orchestrator：

1. 建立 `incidentId` 與 `replayId`。
2. 固定 Source Run、State Revision、Policy Version 與 Required Reference。
3. 驗證 Audit Chain、Checksum、Retention 與 Access Scope。
4. 選擇 Replay Mode 與 Comparison Policy。
5. 建立隔離 Worktree、Container 或 Read-only Sandbox。
6. 在 Adapter 與 Tool Gateway 套用 Side-effect Denylist。

### 15.2 Execute

Agent 只接收 Replay Handoff 指定的 Input Reference。它可以產生分析、驗證 Evidence 與 Replay Result，不能：

- 修改原始 Artifact、Current State 或 Host Journal。
- 讀取 Manifest 以外的 Context。
- 使用替代 Tool 規避被拒絕的 Action。
- 將 Replay Evidence 宣告為原始 Run 的 Accepted Evidence。
- 呼叫 Commit、Push、Deploy、Delete 或外部 Write。

### 15.3 Compare

Reviewer 或 Auditor 依 Comparison Policy 比較：

```text
input identity
policy and environment identity
authorization outcome
tool routing and normalized arguments
artifact structure and checksum
validation and review binding
state transition eligibility
external observation
```

比較結果必須指出差異發生在哪個節點，以及該差異是否足以改變原始 Decision。

### 15.4 Route

| 結果 | 路由 |
|---|---|
| 原始決策可重建且結果相符 | 關閉 Replay，保留 Audit Summary |
| 結果相符但存在無影響 Drift | 記錄 Drift 與 Reproducibility Limitation |
| Evidence Binding 錯誤 | `INCOMPLETE` 或 `FAILED`，交由 Coordinator 建立修正流程 |
| Requirement／Design／Policy 衝突 | `NEEDS_COORDINATOR_ARBITRATION` |
| 需要 Source／Accepted Pointer 修正 | 建立新的 Change／Run，依 09 執行 Promotion／Rollback |
| 需要外部 Write | 暫停並要求新的 Human Approval |
| Provenance 無法閉合 | 回傳固定 Failure Code，不宣告根因 |

Replay 結果不直接更新原始 Current State。新的修正 Run 依合法 Transition Table 更新自己的 State。

## 16. Result Comparison 與 Drift

### 16.1 結果分類

| 分類 | 定義 | 後續處理 |
|---|---|---|
| `exact-match` | 固定欄位與內容完全一致 | 記錄成功 |
| `semantically-equivalent` | 文字或非關鍵欄位不同，Decision 與 Binding 相同 | 記錄允許差異 |
| `acceptable-drift` | Comparison Policy 已列出的環境或時間差異 | 保留 Drift Detail |
| `outcome-diverged` | Verdict、Evidence、Routing 或 Eligibility 改變 | Coordinator 判定修正範圍 |
| `non-reproducible` | 必要版本或非決定性來源無法重建 | 記錄限制，不宣告一致 |
| `integrity-failure` | Checksum、Schema 或 Identity 不符 | 隔離資料並停止 |
| `insufficient-evidence` | Provenance Chain 缺少必要 Record | 補資料或縮小結論 |

### 16.2 Replay Result

概念性 Replay Result：

```yaml
schemaVersion: 0.1.0-draft
replayId: replay-REG-204-01
replayOf:
  changeId: feature-example
  runId: run-004
  decisionRef: runtime://feature-example/host-journal/provenance/evt-review-004.json

mode: decision-replay
status: completed
classification: outcome-diverged

comparison:
  inputIdentity: mismatch
  policyIdentity: match
  environmentIdentity: match
  originalOutcome: approve
  replayOutcome: incomplete

findings:
  - code: PROMOTION_BINDING_MISMATCH
    originalEvidenceRef: evidence://feature-example/run-004/validation-v2.json
    expectedTargetRef: artifact://feature-example/run-004/implementation-v2.json
    actualTargetRef: artifact://feature-example/run-003/implementation-v1.json

nextAllowedAction: coordinator-arbitration
createdAt: 2026-08-14T11:42:00Z
traceId: trace-replay-REG-204-01
```

`completed` 與 `classification` 屬概念性 Replay Record。若以現行 Agent Result 表達，Producer 必須使用 Schema 允許的 Status，並把分類放入對應 Payload、Summary、Risk 或 Trace。

## 17. External Side-effect Reconciliation

### 17.1 禁止盲目重送

下列 Action 不得由 Replay 自動執行：

- Git Commit、Push、Force Push、Tag 與 Release Publication。
- Deployment、Rollback、Traffic Shift 與 Feature Flag Write。
- Database Write、Migration、Delete 與資料修復。
- Issue、Message、Email、Payment 或第三方 API Write。
- Credential Rotation、Permission Change 與 Secret Update。

Tool 名稱不影響判定。Agent 改用 Shell、Script Runner 或另一個 Adapter 呼叫同一外部效果時，Policy 仍要拒絕。

### 17.2 Unknown Outcome

原始 Action 發生 Timeout 或連線中斷時：

1. 讀取 Mutation Receipt、Idempotency Key 與 External Reference。
2. 使用唯讀 Query 查詢 System of Record。
3. 比較 Target Identity、Before Ref 與 After Ref。
4. 將結果分類為 `committed`、`absent`、`partial` 或 `unknown`。
5. 保存新的 External Observation Record。

`unknown` 必須回傳 `SIDE_EFFECT_OUTCOME_UNKNOWN`。Orchestrator 停止 Retry 與 Replay，等待 Coordinator 或 Human 決策。

### 17.3 Compensating Action

已提交的外部效果需要撤銷時，Coordinator 建立 Compensating Action。它使用新的 Mutation Identity、Idempotency Key、Approval 與 Receipt，並以 `compensated` Edge 指向原始 Mutation。

Compensating Action 屬新的受治理變更，不屬於 Replay 的唯讀執行範圍。

## 18. Terminal Run 與 Cross-run

### 18.1 Terminal Run

`terminal: true` 的 Run 維持封閉。Incident 發生後，系統只能：

- 讀取原始 Artifact、Evidence、Trace、State Snapshot 與 Receipt。
- 建立外部 Incident Record、Audit Bundle 與 Replay Result。
- 以 Reference 指向原始 Run。

系統不能修改原始 Result、Gate、Accepted Pointer、Handoff、Revision History 或 Execution Summary。

### 18.2 Cross-run Binding

Replay 或修正 Run 引用原始 Run 時，至少固定：

```text
source changeId / runId
target incidentId / replayId / runId
purpose
artifact or decision URI / checksum
source state revision
source commit
authority level
retention availability
```

Cross-run Reference 在新 Run 中屬 Diagnostic 或 Incident Input。它不會自動取得 Accepted Authority，也不能沿用原始 Approval。

### 18.3 後續修正

Replay 確認錯誤後，Coordinator 依問題建立：

- 新的 Implementation／Validation Run。
- 新的 Context Promotion 或 Compensating Rollback。
- Durable Knowledge Correction。
- 需要 Human Approval 的外部修正。

新 Run 使用目前有效的 Requirement、Policy 與 State。原始事故資料只提供診斷與修正依據。

## 19. 固定失敗代碼與路由

### 19.1 Provenance Failure

| Code | 判定條件 | 安全路由 |
|---|---|---|
| `PROVENANCE_RECORD_MISSING` | 必要 Event、Decision、Receipt 或 Evidence 不存在 | `INCOMPLETE`，列出缺失 Reference |
| `PROVENANCE_CHAIN_BROKEN` | Parent Reference 或必要 Edge 無法閉合 | `INCOMPLETE`，停止根因判定 |
| `PROVENANCE_INTEGRITY_FAILED` | Checksum、Schema、Producer 或 Binding 不符 | `FAILED`，隔離 Record |
| `PROVENANCE_SCOPE_DENIED` | 執行者無權讀取必要資料 | 暫停，要求具 Scope 的 Operator |
| `PROVENANCE_RETENTION_EXPIRED` | 必要 Trace／Evidence 已依 Policy 清除 | `INCOMPLETE`，標示調查限制 |
| `PROVENANCE_POLICY_VERSION_MISSING` | 原始 Authorization／Retrieval Policy 無法取得 | 停止 Decision Replay |

### 19.2 Replay Failure

| Code | 判定條件 | 安全路由 |
|---|---|---|
| `REPLAY_INPUT_UNAVAILABLE` | Manifest 指定輸入遺失或不可讀 | `INCOMPLETE` |
| `REPLAY_ENVIRONMENT_MISMATCH` | Source、Dependency、Tool 或 Runtime 無法建立相容環境 | 停止並記錄差異 |
| `REPLAY_POLICY_VERSION_UNAVAILABLE` | 原始 Policy 無法載入 | 停止 Decision Replay |
| `REPLAY_SIDE_EFFECT_BLOCKED` | Replay 嘗試執行外部 Write | `FAILED`，終止 Invocation |
| `REPLAY_OUTCOME_DIVERGED` | 結果超過 Comparison Policy 的允許差異 | 交由 Coordinator 判定 |
| `REPLAY_AUTHORIZATION_REQUIRED` | 需要敏感資料或 Live Re-execution | 暫停並要求 Approval |
| `REPLAY_MANIFEST_INVALID` | Manifest Binding、Mode 或 Reference 不完整 | `INCOMPLETE`，重新建立 Manifest |

### 19.3 重用既有代碼

下列狀況沿用既有規範：

| Code | 來源與用途 |
|---|---|
| `SIDE_EFFECT_OUTCOME_UNKNOWN` | 08：外部效果無法確認，先 Reconcile |
| `STATE_REVISION_CONFLICT` | 08：Replay／修正使用的 State 前提過期 |
| `TERMINAL_RUN_IMMUTABLE` | 09：要求修改已封閉 Run |
| `ROLLBACK_TARGET_INTEGRITY_FAILED` | 09：Rollback Basis 完整性失敗 |
| Context Integrity／Binding Code | 06：Context URI、Checksum、Authority 或 Target 不符 |

### 19.4 Failure Result

Failure Result 或 Trace 至少保存：

```text
code
incidentId
replayId
source changeId / runId
stage
actorId
recordRef or inputRef
expectedIdentity
actualIdentity
comparisonPolicy
nextAllowedAction
traceId
```

現行 Agent Result 應使用 Schema 已允許的 `blocked`、`failed` 或 `incomplete` Status，並把固定 Code 放入可驗證的 Payload、Blocker、Risk、Summary 或 Trace。Current State 只使用現行 Phase Enum。

## 20. Handoff 與現行 Schema 相容

### 20.1 現有 Handoff 的使用方式

[`handoff-envelope.schema.json`](../schemas/handoff-envelope.schema.json) 沒有 `incidentId`、`replayId`、`replayMode`、`sourceStateRevision` 或 `blockedSideEffects` 欄位，且使用 `additionalProperties: false`。現行 Handoff 不得直接加入這些欄位。

本機流程使用：

- `requiredInputRefs` 固定 Replay Manifest、Audit Bundle、Source Artifact、Evidence 與 Policy Reference。
- Artifact Reference 的 `uri` 與 `sha256` 固定內容。
- `description` 說明 Reference 是 Incident Input、Replay Manifest、Original Evidence 或 Recorded Response。
- `reason` 說明調查問題、Replay Mode 與禁止的 Side-effect 類型。
- `acceptanceCriteriaRefs` 指向 Comparison Policy 與 Audit 完成標準。
- `onSuccess`、`onFailure`、`onConflict` 只使用合法 Phase 與 Owner。
- Orchestrator Invocation Metadata 攜帶 Incident Scope、Replay ID 與 Policy Version。
- Trace／Host Journal 保存 Provenance Event、Audit Bundle 與 Replay Result。

### 20.2 Handoff 片段

以下片段只使用現行 Handoff Schema 已有欄位：

```json
{
  "from": "coordinator",
  "to": "reviewer",
  "stage": "review-result",
  "reason": "Decision replay for REG-204; read-only investigation; external writes are denied by the adapter policy",
  "requiredInputRefs": [
    {
      "refId": "replay-manifest-REG-204-01",
      "type": "other",
      "uri": "runtime://feature-example/incidents/REG-204/replay-manifest-01.json",
      "sha256": "1111111111111111111111111111111111111111111111111111111111111111",
      "description": "Incident replay manifest; host-controlled input"
    },
    {
      "refId": "audit-bundle-REG-204-01",
      "type": "evidence",
      "uri": "evidence://feature-example/incidents/REG-204/audit-bundle-01.json",
      "sha256": "2222222222222222222222222222222222222222222222222222222222222222",
      "description": "Verified provenance references for the source run"
    }
  ]
}
```

`reason` 提供人類可讀說明。Adapter 與 Policy 負責實際阻擋外部 Write；Agent 不能把自然語言限制當成唯一安全邊界。

### 20.3 Agent Result

[`agent-result.schema.json`](../schemas/agent-result.schema.json) 已提供：

- Producer、Change ID、Run ID、Stage 與 Created Time。
- `inputRefs`、`outputRefs` 與 Artifact Checksum。
- Verification、Risk、Summary、Payload 與 Next Handoff。

它沒有 `traceId`、`parentEventIds`、`replayOf`、`policyVersion` 或 `environmentRef`。這些資料應存入 Host Journal／Trace，或建立獨立 Artifact，再透過現行 Reference 指向它。

### 20.4 Current State

[`current-state.schema.json`](../schemas/current-state.schema.json) 保存目前 Phase、Owner、Revision、Accepted Artifact、Gate、Blocker 與 Next Action。它不保存完整 Provenance Graph 或 Replay History。

現行 Phase 沒有 `INCIDENT_REPLAYING`、`AUDITING` 或 `RECONCILING`。事故工作在獨立 Host Workflow 中執行；需要反映阻塞時，Orchestrator 使用合法的 `INCOMPLETE`、`FAILED` 或 `NEEDS_COORDINATOR_ARBITRATION`，並附上 Incident Reference。

### 20.5 Conceptual Record

本文的 Provenance Record、Audit Bundle、Replay Manifest 與 Replay Result 都是概念格式。若要正式成為可交換資產，團隊應先建立：

```text
schemas/provenance-event.schema.json
schemas/audit-bundle.schema.json
schemas/replay-manifest.schema.json
schemas/replay-result.schema.json
templates/provenance-policy.template.yaml
examples/incident-decision-replay/
examples/validation-replay/
```

新增資產前要先定義 Schema Version、Migration、Retention、Signature 與 Compatibility Policy。

## 21. 本機完整範例

### 21.1 原始 Run

`feature-example` 的 `run-004` 產生 implementation v2：

```text
implementation v2
  → validation reported pass
  → reviewer approved
  → readiness passed
  → accepted pointer updated to v2 at revision 9
  → human approved deployment
```

數日後，Production 出現 Regression `REG-204`。外部觀測顯示 v2 在特定輸入下回傳錯誤結果。

### 21.2 Evidence Freeze

Coordinator 宣告 `REG-204`。Orchestrator 固定：

```text
source run: run-004
state revision: 9
source commit: def456
implementation v2 ref / checksum
validation v2 ref / checksum
review v2 ref / checksum
readiness and approval refs
accepted pointer mutation receipt
deployment external reference
```

原始 Run 已是 Terminal。Orchestrator 不修改其中任何檔案。

### 21.3 Audit Reconstruction

Auditor 沿 Provenance Chain 檢查 Binding：

```text
implementation v2
  sha256: aaaa...

validation-v2.json
  targetRef: implementation v1
  targetSha256: 1111...

review-v2.json
  candidateRef: implementation v2
  validationRef: validation-v2.json
```

Validation Evidence 綁定 v1，Review 卻把它當成 v2 的 Evidence。Current State 顯示 v2 已 Accepted，無法補足中間的 Binding 缺口。

### 21.4 Decision Replay

Orchestrator 建立 `replay-REG-204-01`，載入原始 Policy 與 Revision，並阻擋所有外部 Write。Decision Replay 重新檢查 Promotion Eligibility：

```text
expected validation target: implementation v2 / aaaa...
actual validation target:   implementation v1 / 1111...
outcome: incomplete
code: PROMOTION_BINDING_MISMATCH
```

Replay Result 分類為 `outcome-diverged`。它指向原始 Review Decision，不會改寫原 Verdict。

### 21.5 Validation Replay

Coordinator 要求第二次 Replay。Orchestrator 在固定 `def456` 的隔離 Worktree 執行原始 Validation Command，並產生：

```text
replay evidence target: implementation v2 / aaaa...
regression test: failed
side effects: none
classification: outcome-diverged
```

新 Evidence 只服務 Incident 調查。它不能直接替代 `run-004` 的原始 Evidence。

### 21.6 修正與 Rollback

Coordinator 根據 Audit 與 Replay Result 建立新的修正 Change。若團隊選擇回退 Accepted Context，流程依 09 建立 Rollback Basis、Rollback Candidate、Validation、Review、Gate 與新 Mutation Receipt。

Production Rollback 需要新的 Human Approval 與外部 Side-effect Receipt。Incident Replay 不執行該操作。

### 21.7 結案

Incident Summary 保存：

- Provenance 缺口與最早可證明的錯誤 Decision。
- Decision／Validation Replay 的 Reference 與比較結果。
- 修正 Change、Rollback 或 Durable Knowledge Correction 的 Reference。
- Retention Hold 的解除條件。
- 後續 Policy、Schema、測試或監控改善。

## 22. 最小本機導入

### 22.1 可選目錄

第一版可以在 Git-ignored Runtime 中加入 Host 專用區：

```text
.agent-runtime/<change-id>/
├─ current-state.json
├─ artifacts/
├─ evidence/
├─ host-journal/
│  ├─ provenance/
│  │  └─ <event-id>.json
│  └─ mutation-receipts/
└─ incidents/
   └─ <incident-id>/
      ├─ replay-manifest.json
      ├─ audit-bundle.json
      ├─ environment.json
      └─ replay-results/
```

`host-journal` 與 `incidents` 由 Orchestrator／Runtime Host 管理。Agent 只透過 Handoff Reference 讀取授權內容，不能列舉或修改整個目錄。

### 22.2 最小 Host 介面

本機 Orchestrator 至少提供：

```text
appendProvenanceEvent(record)
verifyProvenanceChain(scope)
buildAuditBundle(incidentId, refs)
createReplayManifest(mode, sourceRefs, policy)
prepareIsolatedReplay(manifest)
publishReplayResult(result)
reconcileExternalEffect(idempotencyKey, targetRef)
applyRetentionHold(incidentId, refs)
```

所有寫入採 Create-only 或 Atomic Replace。原始 Artifact、Terminal Run 與既有 Journal Record 不提供 Update 介面。

### 22.3 最小落地順序

1. 先讓 Agent Result、Handoff 與 Evidence 固定 URI、Checksum、Change ID 與 Run ID。
2. Host Journal 保存 Authorization Decision、Tool Action、Mutation Receipt 與 State Revision。
3. 建立能依 Reference 匯出 Audit Bundle 的腳本。
4. Replay 先支援 Decision Replay 與 Validation Replay。
5. Adapter 對 Replay Invocation 套用外部 Write Denylist。
6. 為 Audit、Trace、Evidence 與 Incident Hold 設定 Retention Policy。
7. 使用一個已知 Binding Error 的 Fixture 驗證完整流程。

## 23. Durable Store 升級判準

符合任一條件時，團隊應評估將 Provenance 與 Replay Record 移到支援 Append-only、Index、CAS 或 Tamper Evidence 的 Durable Store：

- 多個 Orchestrator Process 或多台 Worker 共同處理同一 Change。
- 需要跨 Host、跨 Repository 或跨環境查詢完整 Lineage。
- 稽核要求重建每一次 Authorization、Gate 與 State Transition。
- 法遵要求 WORM、Digital Signature、獨立 Timestamp 或 Legal Hold。
- 外部 Side-effect 需要可靠 Outbox／Inbox 與交易式 Receipt。
- Incident 數量使本機檔案難以索引、保留與權限分區。
- Runtime 位於 Network Filesystem，無法可靠提供 Create-only 與 Atomicity。

升級後仍保留以下語意：

```text
current state is a projection
artifact is immutable
provenance uses explicit references
decision binds policy and state revision
replay writes new records
external effects require reconciliation and approval
terminal run remains immutable
```

單機、Single Writer、低事故量且沒有法遵需求時，本機 Host Journal 已能支援基本調查。團隊應先驗證因果鏈與 Replay 邊界，再評估額外基礎設施。

## 24. 驗收清單

### Provenance Coverage

- [ ] Agent Result 可追溯 Producer、Change ID、Run ID 與 Stage。
- [ ] Input／Output Reference 使用固定 URI 與 Checksum。
- [ ] Validation、Review、Gate 與 Accepted Pointer 指向同一 Target Identity。
- [ ] State Transition 可追溯 Decision、Expected Revision 與 Mutation Receipt。
- [ ] Parent Reference 或 Provenance Edge 能建立因果順序。
- [ ] Current State 只保存目前投影，不承擔完整歷史。

### Audit Integrity

- [ ] Stable Audit Record 採 Append-only。
- [ ] Audit Bundle 只包含調查範圍內的 Reference 與 Integrity Status。
- [ ] Timestamp 不作為唯一因果判定依據。
- [ ] 缺失 Record 會回傳固定 Code。
- [ ] Auditor 將已確認事實、推測與資料缺口分開。
- [ ] Producer、Policy Version、State Revision 與 Decision Owner 可被重建。

### Replay Isolation

- [ ] 每次 Replay 有獨立 `incidentId` 與 `replayId`。
- [ ] Replay Manifest 固定 Source、Policy、Environment 與 Required Reference。
- [ ] Replay 使用隔離 Worktree、Container 或 Read-only Sandbox。
- [ ] Replay Result 使用新 URI，並指向原始 Record。
- [ ] Replay 不修改原始 Artifact、Current State 或 Terminal Run。
- [ ] Comparison Policy 區分 Exact、Semantic、Drift 與 Non-reproducible。

### Side-effect Safety

- [ ] Adapter／Tool Gateway 阻擋 Replay 的 Commit、Push、Deploy、Delete 與外部 Write。
- [ ] Tool 替代路徑不能繞過 Action 授權。
- [ ] Unknown Side-effect Outcome 先查詢 System of Record。
- [ ] Side-effect Mutation 有 Idempotency Key、Before／After Ref 與 Receipt。
- [ ] Live Re-execution 使用新的 Run、Gate 與 Human Approval。
- [ ] Compensating Action 指向原始 Mutation，並產生新 Receipt。

### Privacy 與 Retention

- [ ] Audit Record 不保存 Secret、Credential、Approval Proof 原文或模型私有推理。
- [ ] 敏感 Arguments 經正規化與 Redaction。
- [ ] Audit、Evidence、Trace 與 Replay Artifact 有不同 Retention Class。
- [ ] Incident Hold 有授權、期限與解除條件。
- [ ] Audit Scope 不授予整個 Runtime Store 的列舉權。
- [ ] 已過期資料會回報限制，不以 Latest Artifact 替代。

### Lifecycle 與 Schema 相容性

- [ ] Terminal Run 維持封閉，事故調查使用 Cross-run Reference。
- [ ] Replay Evidence 不直接成為原始 Run 的 Accepted Evidence。
- [ ] 修正、Rollback 與外部 Write 建立新的 Change／Run。
- [ ] 現行 Handoff 沒有加入未定義的 Incident／Replay 欄位。
- [ ] Current State 沒有加入未定義的 Audit／Replay Phase。
- [ ] 概念性 Record 在正式採用前具備 Schema Version 與 Migration Policy。

## 25. 參考文件

- [Artifact-based Shared State + Structured Handoff 參考架構](./02-reference-architecture-zh-TW.md)
- [Repository Knowledge、Runtime State、Evidence 與 Trace 分層](./03-runtime-storage-and-retention-zh-TW.md)
- [安全、治理與長期維護](./05-security-and-maintainability-zh-TW.md)
- [Context Authority 與 Retrieval Policy](./06-context-authority-and-retrieval-policy-zh-TW.md)
- [Role Capability 與 Scope Control](./07-role-capability-and-scope-control-zh-TW.md)
- [Mutation、Concurrency 與 Conflict Resolution](./08-mutation-concurrency-and-conflict-resolution-zh-TW.md)
- [Context Promotion 與 Rollback](./09-context-promotion-and-rollback-zh-TW.md)
- [Handoff Envelope Schema](../schemas/handoff-envelope.schema.json)
- [Current State Schema](../schemas/current-state.schema.json)
- [Agent Result Schema](../schemas/agent-result.schema.json)
- [Workflow Policy Template](../templates/workflow-policy.template.yaml)
