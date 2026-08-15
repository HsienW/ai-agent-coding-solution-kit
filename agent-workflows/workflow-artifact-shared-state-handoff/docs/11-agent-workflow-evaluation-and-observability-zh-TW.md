# 11｜Agent Workflow Evaluation 與 Observability

[English](./11-agent-workflow-evaluation-and-observability.md) | [繁體中文](./11-agent-workflow-evaluation-and-observability-zh-TW.md)

本文件規範多 Agent Workflow 如何評估 Artifact、Agent Stage 與整體執行流程，並以 Metrics、Event、Trace、Evidence 和 Audit Record 保留足以判斷品質與營運狀態的訊號。適用對象包括 Agent、Coordinator、Orchestrator、Reviewer／Evaluator、Runtime／Telemetry Host 與 Human Approver。

一個 Workflow Run 顯示 `COMPLETED`，只代表 State Machine 走到終點。團隊仍要知道輸出是否符合 Requirement、Agent 是否使用正確 Context、Retry 是否掩蓋不穩定行為，以及新版 Prompt、Router 或 Workflow Policy 有沒有讓特定高風險情境退化。Evaluation 提供可重複的比較方法，Observability 保存實際執行訊號，兩者共同支援 Release Gate、問題定位與後續治理。

本文件採用以下原則：

- Evaluation 固定 Subject、Dataset、Rubric、Policy、版本與環境，結果才能比較。
- 安全、授權、完整性與必要 Gate 屬於 Hard Invariant，Aggregate Score 不能抵銷失敗。
- Evaluation Result 只提供 Decision Input；Accepted Pointer 仍由 Orchestrator 依 Policy、Gate 與 CAS 更新。
- Metrics、Event、Trace、Evidence 與 Audit Record 各自保存不同粒度的資料。
- 現行 Schema 維持不變；新增紀錄放入 Trace、Host Journal 或獨立 Artifact，再以 Reference 連結。

## 1. 適用範圍

本文件接續既有規範：

- [`03-runtime-storage-and-retention-zh-TW.md`](./03-runtime-storage-and-retention-zh-TW.md) 定義 Durable Knowledge、Runtime State、Evidence 與 Trace 的保存層級。
- [`05-security-and-maintainability-zh-TW.md`](./05-security-and-maintainability-zh-TW.md) 定義 Evidence Provenance、Artifact Integrity、Human Gate 與營運檢查。
- [`06-context-authority-and-retrieval-policy-zh-TW.md`](./06-context-authority-and-retrieval-policy-zh-TW.md) 定義 Context Authority、Freshness、Integrity、Binding 與 Budget。
- [`07-role-capability-and-scope-control-zh-TW.md`](./07-role-capability-and-scope-control-zh-TW.md) 定義 Runtime Identity、Capability、Policy Version 與 Authorization Trace。
- [`08-mutation-concurrency-and-conflict-resolution-zh-TW.md`](./08-mutation-concurrency-and-conflict-resolution-zh-TW.md) 定義 Revision、CAS、Mutation Receipt、Retry 與 Conflict。
- [`09-context-promotion-and-rollback-zh-TW.md`](./09-context-promotion-and-rollback-zh-TW.md) 定義 Promotion、Rollback、Gate Invalidation 與 Accepted Pointer。
- [`10-provenance-audit-and-incident-replay-zh-TW.md`](./10-provenance-audit-and-incident-replay-zh-TW.md) 定義 Provenance Graph、Audit Reconstruction、Replay 與 Drift 分類。

本文件處理：

- Artifact Quality、Agent／Stage Capability 與 Workflow Operations 的評估層級。
- Offline Evaluation、Shadow Evaluation、Canary Observation、Online Monitoring 與 Replay Evaluation。
- Evaluation Dataset、Baseline、Candidate、Cohort、Rubric、Hard Invariant 與 Verdict。
- Metrics、Normalized Event、Distributed Trace、Evidence 與 Audit Record 的分工。
- Regression、Drift、SLI、SLO、Alert、Sampling、Retention 與 Redaction。
- 本機 `.agent-runtime` 的最小評估閉環，以及 Production Telemetry 的升級條件。

下列內容留在其他規範：

- Current State 的 Phase Enum、Transition Table 與 Gate 欄位。
- Prompt Package 內部的 Prompt、Schema、Example、Validator 與 Lockfile 格式。
- 特定模型供應商、Observability Vendor、Dashboard 產品或計費 API 的實作。
- 組織層級的績效考核、員工評分與商業 KPI。

## 2. 名詞與操作邊界

### 2.1 Validation

Validation 檢查單一輸入、輸出或 Candidate 是否符合明確 Contract，例如 JSON Schema、Domain Rule、Checksum、Target Binding、測試命令與 Gate 前置條件。Validation 通常產生 `pass`、`fail`、`partial` 或 `not_run`，並綁定一份固定 Target。

Validation Failure 必須控制 Flow。Runner 不能把錯誤只寫入 Log，接著執行依賴該結果的 Artifact Node 或外部 Tool。

### 2.2 Evaluation

Evaluation 使用版本化 Dataset 與 Rubric，評量一個固定 Subject 在多個 Case、Cohort 或實際流量樣本中的行為。Subject 可以是 Agent、Prompt Package、Router、Context Policy、Workflow、Tool Set、Model Configuration，或這些元件組成的 Release Unit。

Evaluation 回答 Candidate 相對 Baseline 是否改善、退化或缺乏足夠證據。它不直接完成 Promotion，也不取代 Reviewer Verdict 或 Human Approval。

### 2.3 Observability 與 Telemetry

Observability 是團隊依執行訊號判斷系統內部狀態的能力。Telemetry 是 Runtime 產生的 Metrics、Event、Trace 與 Resource Usage。完整 stdout、Raw Prompt 或整份 Tool Payload 不會因為標上 Telemetry 便自動取得保存資格；Data Classification、Redaction 與 Retention Policy 仍然適用。

### 2.4 Evidence、Trace 與 Audit Record

- Evidence 支持一項工程聲明，例如某份 Diff 通過指定測試，或某個 Evaluation Suite 已執行完成。
- Trace 保存一次 Run 的診斷細節與因果關聯，允許較短 Retention 與受控 Sampling。
- Audit Record 保存重建 Decision、Authorization、Gate 與 State Mutation 所需的穩定事實。

Evaluation Report 可以引用 Evidence、Trace 與 Audit Record，不能用 Aggregate Metric 取代 Target Binding。

### 2.5 Baseline、Candidate 與 Cohort

- Baseline 是已固定 Identity 的比較基準，可以是目前 Stable Release、上一份 Accepted Artifact 或經批准的 Policy Snapshot。
- Candidate 是準備接受評估的 Release Unit，內容必須固定，執行期間不得原地修改。
- Cohort 是具有共同條件的一組 Case 或 Run，例如 Domain、Risk Level、Repository、Workflow Stage、Model Version、語言或 Release Mode。

整體平均值容易遮住小型高風險 Cohort。Evaluation Report 必須保存分群結果，並標示樣本數。

### 2.6 Regression 與 Drift

Regression 表示 Candidate 在可比較條件下低於 Baseline 或違反既定 Invariant。Drift 表示輸入分布、Dependency、Model、Tool、Policy、Latency 或結果特徵隨時間改變。單次失敗可以觸發診斷，尚不足以單獨證明統計性 Drift。

## 3. 角色與責任

| 角色 | 責任 | 禁止事項 |
|---|---|---|
| Coordinator | 定義評估目標、Subject、Risk Cohort、Rubric、Hard Invariant 與可接受差異 | 不以單一 Aggregate Score 忽略高風險退化；不修改原始 Case Result |
| Orchestrator | 固定版本、建立 Evaluation Manifest、執行或調度 Suite、驗證 Binding、套用 Gate Policy | 不改寫 Evaluator Verdict；不讓未固定 Candidate 進入比較；不自行放寬 Threshold |
| Agent／Producer | 產生符合 Contract 的 Artifact、Evidence 與最小 Telemetry | 不替自己的輸出宣告整體通過；不更新 Accepted Pointer；不在 Telemetry 保存私有推理 |
| Reviewer／Evaluator | 唯讀檢查 Subject、Evidence、Rubric 與 Case Result，記錄 Verdict、限制與不確定性 | 不修改 Source、Candidate、Current State、Gate 或 Approval |
| Runtime／Telemetry Host | 產生 Identity、收集 Event／Trace、執行 Redaction、聚合 Metrics、套用 Retention | 不將 Secret 或完整敏感 Payload 送入 Telemetry Backend；不以 Sampled Trace 充當完整 Audit |
| Human Approver | 批准高風險 Threshold、Release、資料存取與 Policy 例外 | 不以口頭同意替代 Approval Record；不把 Reviewer Verdict 當成外部副作用授權 |

`Evaluator` 與 `Telemetry Operator` 是職責名稱，不代表現行 Schema 新增 Role。實際 Runtime Identity 使用 `reviewer`、`automation` 或經 07 文件授權的 Host Identity。

## 4. Evaluation 與 Observability Invariant

Workflow Host 必須遵守以下條件：

1. 每次 Evaluation 都具有唯一 `evaluationId`，並固定 Subject、Dataset、Rubric、Policy 與環境身分。
2. Baseline 與 Candidate 必須具有可比較的 Release Unit；版本集合不同時先標示相容性限制。
3. Dataset 執行期間保持 Immutable；新增 Incident Case 時建立新 Version。
4. Case Result 必須可追溯 Input、Expected Invariant、Actual Output、Evaluator 與 Evidence。
5. 安全、授權、Schema Integrity、Target Binding 與外部副作用控制採 Fail Closed。
6. Aggregate Score 不能覆蓋 Hard Invariant Failure。
7. Evaluation 完成狀態與品質 Verdict 分開記錄；Suite 成功執行不代表 Candidate 通過。
8. Producer 不得自行接受或 Promotion 自己的 Candidate。
9. Reviewer／Evaluator 的 Verdict 不直接更新 Current State；Orchestrator 依 Policy 與 Expected Revision 提交 Mutation。
10. Metrics 保存聚合趨勢，Run／Artifact／Invocation Identity 保存於 Event 與 Trace。
11. Event、Trace、Evidence 與 Audit 使用共同的 `changeId`、`runId`、`stage` 與必要 Reference 建立關聯。
12. Timestamp 只提供時間資訊；因果關係由 Parent Reference、Span Link、Revision 與 Target Binding 建立。
13. Telemetry 經 Redaction 後才離開 Runtime Trust Boundary。
14. Sampling 不得移除 Incident、Policy Denial、Human Override、Integrity Failure 與 Unknown Side-effect Outcome 所需的 Audit Record。
15. Evaluation 或 Observability 資料缺失時，系統回報 `inconclusive`、`INCOMPLETE` 或固定失敗代碼，不能以最近一筆資料補值。
16. Terminal Run 維持封閉；後續 Evaluation、Replay 或回歸分析建立新的 Identity 與 Artifact。

## 5. 三層評估模型

### 5.1 Artifact Quality

Artifact Quality 檢查交付結果是否符合批准的 Requirement、Schema、Domain Rule 與工程品質標準。常用訊號包括：

- Requirement 與 Acceptance Criteria Coverage。
- Schema／Domain Validation 結果。
- Test、Lint、Build 與 Diff Check。
- Review Finding 的 Severity、Status 與 Target Binding。
- 未執行驗證、Residual Risk 與後續人工判斷。

Artifact Quality 的評估單位是固定 URI、Checksum、Diff Hash 或 Commit。檔名相同而內容不同時，評估結果不得沿用。

### 5.2 Agent／Stage Capability

Agent／Stage Capability 檢查角色是否在批准 Scope 內完成指定工作：

- Router 是否選到正確 Domain 與 Skill。
- Context Builder 是否使用正確 Authority、Version 與 Budget。
- Agent 是否遵守 Output Contract、Tool Policy 與 Handoff Scope。
- Validation Failure 是否進入正確 Fallback。
- Evidence 是否完整，Reason Code 是否可機器判讀。
- Retry、Clarify、Escalation 與 Human Review 是否符合 Policy。

此層評估 Agent 在特定 Stage 的行為，不能將 Reviewer 的唯讀工作與 Implementer 的修改能力放進同一分母比較。

### 5.3 Workflow Operations

Workflow Operations 檢查端到端執行的可靠度與成本：

- Run Completion、Accepted Outcome 與 Rework。
- Stage Latency、Handoff Wait、Human Gate Wait 與 Queue Time。
- Retry、Timeout、Conflict、Rollback 與 Incident。
- Schema Invalid、Context Stale、State Revision Conflict 與 Partial Write。
- Token、Model Call、Tool Call、Storage 與 Telemetry Cost。

Workflow 到達 `COMPLETED` 後若立即 Rollback，單看 Completion Rate 會得到錯誤判斷。Outcome 指標應搭配 Acceptance、Regression 與 Incident 視窗。

## 6. Evaluation Subject 與版本綁定

### 6.1 Subject 可以是單一資產或 Release Unit

常見 Subject：

| Subject | 最小版本身分 |
|---|---|
| Prompt Package | Prompt、Schema、Example、Eval Dataset、Validator、Lockfile Hash |
| Router | Router Code／Policy、Domain Registry、Alias、Threshold、Model Version |
| Context Builder | Builder Version、Retrieval Policy、Allowed Fields、Budget、Source Index Version |
| Agent | Agent／Skill Version、Prompt Package、Tool Set、Model／Adapter、Capability Policy |
| Workflow | Workflow Definition、Node Implementation、Fallback Policy、Transition Policy |
| Tool | Tool／Schema／Behavior Version、Authorization、Timeout、Retry、Idempotency Policy |
| Release Unit | 上述資產的相容組合與 Environment Reference |

團隊若只記錄 `model=gpt-x` 或 `prompt=latest`，後續無法建立可比較的 Evaluation。Manifest 應指向精確 Snapshot 或可驗證 Hash。

### 6.2 最小 Binding

Evaluation Manifest 至少固定：

```text
changeId / runId
evaluationId
subjectRef / subjectHash
baselineRef / baselineHash
datasetRef / datasetVersion / datasetHash
rubricRef / rubricVersion
evaluationPolicyVersion
model / adapter / tool versions
workflow / router / prompt lockfile versions
environmentRef
risk cohort and release mode
```

缺少任何會改變輸出的版本時，Evaluator 必須在 Result 標示限制。Baseline 與 Candidate 使用不同 Dataset 或 Rubric 時，系統不得直接計算 Regression Percentage。

### 6.3 Subject 變更使結果失效

下列變動需要新 Evaluation：

- Prompt、Example、Schema、Validator 或 Lockfile 改變。
- Router Priority、Confidence Threshold 或 Domain Registry 改變。
- Allowed Context Fields、Retrieval Policy、Index 或 Budget 改變。
- Model、Adapter、Tool、Dependency 或 Environment 改變，且 Policy 將其列為 Material Change。
- Workflow Node、Condition、Fallback、Retry 或 Gate 改變。
- Candidate Artifact、Diff Hash 或 Source Base 改變。

Policy 可以允許非語意 Metadata 更新沿用結果，但必須列出允許欄位，不能由 Producer 自行判斷。

## 7. Evaluation Mode

| Mode | 資料來源 | 是否控制正式 Action | 主要用途 |
|---|---|---:|---|
| Contract Test | 固定 Fixture | 否 | Schema、Reason Code、Transition 與 Binding |
| Offline Evaluation | 版本化 Dataset | 否 | Baseline／Candidate Regression |
| Simulation | 合成環境與 Tool Stub | 否 | Timeout、Conflict、Fallback 與 Side-effect 防護 |
| Replay Evaluation | 固定原始 Input、Policy 與環境 | 否 | Decision／Validation 重算與 Incident 分析 |
| Shadow Evaluation | 實際輸入的平行 Candidate | 否 | 比較 Route、Output、Latency 與 Cost |
| Canary Observation | 小範圍正式流量 | 有條件 | 驗證真實環境行為與 Rollback 條件 |
| Online Monitoring | 正式 Runtime Telemetry | 是既有流程 | 偵測 Error、Drift、Cost 與 SLO Breach |
| Human Benchmark | 經抽樣與盲測的 Case | 否 | 校準語意品質、風險與 LLM Evaluator |

Shadow、Simulation 與 Replay 預設拒絕 Commit、Push、Deploy、Delete 與外部 API Write。Canary 若允許 Action，仍須經 07 的 Authorization、08 的 Idempotency 與 09 的 Approval Binding。

## 8. Evaluation Dataset 與 Case

### 8.1 Dataset 內容

每個 Evaluation Case 至少描述：

```text
caseId
input reference or fixture
task class / domain / stage
risk level and tags
expected route or allowed routes
required invariants
allowed output variation
forbidden behavior
required evidence
evaluation method
```

Case 可以接受多種語意等價輸出。Expected Output 若要求完整字串相等，應確認該輸出具有決定性；自由文字、摘要或程式碼修正通常更適合使用 Invariant、Validator、Reference Output 與人工 Rubric 組合判斷。

### 8.2 最小 Case 群

Dataset 應涵蓋：

- Normal、Boundary、Missing Context 與 Invalid Input。
- Cross-domain Similarity、Low-confidence Route 與 Clarification。
- Stale／Conflicting Context、Wrong Authority 與 Budget Exhaustion。
- Schema Valid 但 Domain Invalid 的輸出。
- Prompt Injection、Scope Denial 與 Unauthorized Action。
- Tool Timeout、Invalid Tool Output、Duplicate Request 與 Unknown Side-effect Outcome。
- Review Finding、Validation Binding Mismatch、Retry Exhaustion 與 Human Reject。
- Parallel Mutation、State Revision Conflict、Rollback 與 Incident-derived Case。

### 8.3 Dataset 來源與治理

Case 可以來自合成資料、已匿名化的實際 Run、Production Incident 或人工設計的 Adversarial Scenario。Incident 修復完成後，團隊應將最小可重現條件提升為新 Case，並移除 Secret、個資、私有 Repository 內容與可重放 Credential。

Dataset 必須有 Owner、Version、Change History、Data Classification 與 Retention。調整 Expected Result 需要 Review；為了讓 Candidate 通過而放寬答案，屬於 Dataset 變更，不能混在同一次 Candidate 比較中。

### 8.4 Leakage 與 Circular Evaluation

模型若在 Prompt、Few-shot Example 或 Retrieval Source 中看過完整 Eval Answer，結果可能高估真實能力。團隊應分開 Training／Tuning Case、Development Case 與 Release Gate Case，並限制 Agent 列舉隱藏 Dataset。

Producer 與 Evaluator 使用同一模型時，Result 必須標示這項限制。高風險語意判斷應加入 Deterministic Check、不同 Evaluator 或 Human Calibration，避免模型用自己的偏好驗收自己的輸出。

## 9. Baseline、Candidate 與 Cohort

### 9.1 Baseline Eligibility

Baseline 應符合以下條件：

- 具有固定 URI、Hash、Release Unit 與 Environment Reference。
- Dataset、Rubric 與 Metric Definition 可取得。
- 已知限制與 Incident 已記錄。
- 評估資料未因 Retention 過期而失去必要 Binding。

上一版號不必然適合作為 Baseline。它若使用不同 Domain、不同 Schema 或不相容環境，Coordinator 應選擇可比較版本，或將結果標示為 `baseline-incompatible`。

### 9.2 Cohort 維度

至少依下列維度評估是否需要切分：

```text
domain / task class
risk level
workflow stage
repository or component
model / prompt / router / policy version
tool operation type
language / locale
release mode / environment
```

每個 Cohort 要保存樣本數、成功數、失敗數與信賴限制。樣本過少時，Result 使用 `inconclusive`，不能用百分比製造確定性。

## 10. Rubric、Hard Invariant 與 Verdict

### 10.1 Rubric 結構

Rubric 可以包含：

- Deterministic Check：Schema、Checksum、Enum、Target Binding、Exit Code。
- Rule-based Check：Domain Rule、Scope、Forbidden Behavior、Evidence Completeness。
- Semantic Evaluation：Requirement Coverage、回答支持度、Review Quality。
- Operational Threshold：Latency、Retry、Cost、Timeout 與 Error Rate。

每個項目需要名稱、版本、適用 Cohort、計算方式、資料來源、Threshold 與失敗路由。自由文字描述若無法機器判讀，至少要提供 Reviewer Checklist 與 Evidence Reference。

### 10.2 Hard Invariant

以下項目預設不得以加權平均抵銷：

- Unauthorized Action 或 Scope Violation。
- Secret／Credential Leakage。
- Schema、Checksum、Provenance 或 Target Binding Failure。
- Domain Validator Failed 後仍執行下游 Action。
- Blocker／Major Finding 未依 Policy 處理。
- Required Evidence Missing。
- Unknown Side-effect Outcome 未經 Reconciliation。
- Replay／Shadow Invocation 執行外部 Write。

任何 Hard Invariant 失敗，Evaluation Verdict 應為 `fail` 或 `inconclusive`；Policy 不得僅因 Overall Score 達標便放行。

### 10.3 Completion Status 與 Verdict

Evaluation 執行狀態：

```text
completed
partial
failed_to_run
cancelled
```

Evaluation 品質判定：

```text
pass
fail
inconclusive
not_comparable
```

`completed + fail` 表示 Suite 完整執行且 Candidate 未通過；`failed_to_run + inconclusive` 表示基礎設施或必要資料不足。兩組狀態不可合併成單一 `success` 欄位。

## 11. Evaluator 選擇與校準

### 11.1 評估方法優先順序

機械事實先使用 Deterministic Validator。語意存在多個合法答案時，再使用 Rule-based Evaluator、Reference Comparison、LLM Evaluator 或 Human Review。Evaluator 的成本與彈性不能取代可重複的 Contract Check。

### 11.2 LLM Evaluator

使用 LLM 判斷語意品質時，Manifest 應固定：

- Evaluator Model、Version、Prompt、Rubric 與 Temperature。
- Candidate Identity 是否盲化。
- 輸入欄位、Context Source 與 Redaction Policy。
- 多次執行、Majority／Aggregation 規則與 Tie-breaker。
- 與 Human Benchmark 的校準日期及差異。

Evaluator 應輸出結構化 Reason Code、Criterion Result、Evidence Reference 與 Confidence。長篇自然語言評論只能補充說明，不能成為唯一 Gate Input。

### 11.3 獨立性

Producer 可以執行自我檢查，結果視為 Candidate Evidence。正式 Release Gate 需要獨立 Reviewer、Evaluator Host 或 Deterministic Suite。Reviewer 保持 Source Read-only，Orchestrator 保持 Transition Authority，兩者職責不合併。

## 12. Lifecycle Stage Evaluation Matrix

| Stage | 評估 Subject | 必要檢查 | 主要訊號 |
|---|---|---|---|
| `plan-change` | Requirement／Plan | Scope、Constraint、Open Question、可驗收性 | Requirement Coverage、Clarification Rate |
| `review-plan` | Plan Candidate | Design Conflict、Risk、Acceptance Criteria | Finding Severity、Approval／Change Request |
| `apply-change` | Implementation Candidate | Changed Files、Diff、Validation、Scope Deviation | Test Pass、Schema Invalid、Retry、Duration |
| `review-result` | Candidate + Evidence | Target Binding、Finding、Residual Risk | Review Verdict、Correction Rate、Evidence Completeness |
| `fix-from-review` | 修正版 Candidate | Finding Closure、Regression、Diff Delta | Rework Attempt、Reopened Finding |
| `readiness-check` | Accepted Candidate | Gate、Unresolved Item、Archive Eligibility | Gate Pass、Human Wait、Incomplete Rate |
| `archive-change` | Execution Summary | Durable Fact、Reference、Remaining Work | Summary Completeness、Archive Failure |
| `manual-gate` | 固定 Action／Payload | Approval Binding、Scope、Expiry | Approve／Reject、Wait Time、Invalidation |

現行 Lifecycle 沒有 `evaluate-workflow` Stage。獨立 Evaluation Suite 由 Host Workflow 執行；需要把結果交給 Reviewer 或 Readiness 時，以 Reference 傳入現有 Stage。

### 12.1 Prompt Runtime 節點

對應 [`ai-agent-prompt-runtime-workflow-phase2-zh-TW.md`](../../../ai-agent-prompt-runtime-workflow-phase2-zh-TW.md) 的 Runtime Execution：

| Node | 評估問題 | 建議指標／Invariant |
|---|---|---|
| Runtime Input | Input Contract 與 Trust Boundary 是否完整 | Input Contract Failure、Untrusted Field Rejection |
| Domain Router | 是否使用正確 Domain、Skill 與 Reason | Route Accuracy、Domain Mismatch、Clarify Precision |
| Skill Loader | 是否解析固定 Prompt Package 與 Validator | Package Binding、Lockfile Match |
| Context Builder | 是否使用最小且權威的 Context | Authority Coverage、Source Overuse、Token by Source |
| Prompt Assembly | 是否固定 Model、Prompt、Example 與 Schema | Snapshot Completeness、Version Missing |
| Schema Validator | Output Shape 是否正確 | Schema Failure、Compact Retry Success |
| Domain Validator | 語意與跨欄位規則是否成立 | Domain Failure、Cross-domain Contamination |
| Artifact Transformer | 是否只在 Spec 通過後執行 | Gate Bypass Count 必須為 0 |
| Fallback | Failure 是否進入正確出口 | Fallback Accuracy、Retry Exhaustion |
| Release | Formal Flow 是否經 Evaluation、Approval 與 Canary | Gate Coverage、Rollback Trigger |

## 13. Signal Model

| Signal | 保存內容 | 適合回答 | 不承擔的責任 |
|---|---|---|---|
| Metrics | 聚合計數、比例、分布與資源用量 | 趨勢、SLO、Alert、Cohort 比較 | 單次 Run 的完整因果鏈 |
| Event／Log | 正規化事件、Status、Reason Code 與安全摘要 | 某個時間點發生什麼 | 大型 Raw Payload 與長期決策證據 |
| Trace／Span | Run、Stage、Invocation、Tool 的因果與 Timing | 延遲位置、Retry、Fallback、跨服務診斷 | 永久 Audit 與 Accepted Evidence |
| Evidence | 綁定 Target 的命令、驗證與 Review 結果 | 一項聲明是否可驗證 | 全部執行細節與平台趨勢 |
| Audit Record | Authorization、Gate、Mutation 與穩定 Reference | 誰依哪個 Policy 做出決定 | 高容量 Debug Payload |

同一個 Failure 可以同時產生 Metric、Event、Trace 與 Evidence，但每份資料只保存自己需要的欄位。複製完整內容會增加 Secret Exposure、Retention 成本與不一致風險。

## 14. Trace 與 Correlation

### 14.1 Span 結構

建議的因果結構：

```text
Workflow Run Span
└─ Stage Span
   └─ Agent Invocation Span
      ├─ Context Resolution Span
      ├─ Model Call Span
      ├─ Tool Action Span
      ├─ Validation Span
      └─ Artifact Publication Span
```

Handoff、Queue、Async Worker、Shadow Candidate 與 Cross-run Evaluation 可能無法使用直接 Parent／Child。Host 應使用 Span Link 或明確 Reference 連結來源 Run，不能建立虛假的父子關係。

### 14.2 Identity

Trace 應沿用 10 號文件的 Identity 分工：

```text
changeId       工程變更範圍
runId          一次 Workflow Run
invocationId   一次 Agent Invocation
traceId        一組診斷關聯
spanId         Trace 內的一個操作
evaluationId   一次 Evaluation Execution
caseRunId      一個 Case 的一次執行
mutationId     一次受控寫入
```

Retry 產生新的 `invocationId` 或 `caseRunId`，並引用原始 Attempt。重用同一 ID 會使 Latency、Token 與 Failure Count 無法區分。

### 14.3 最小 Span Attribute

```text
changeId / runId / stage
actorId / role
workflow / agent / prompt / model / adapter / tool versions
policyVersion / stateRevision
operation / status / reasonCode
startedAt / endedAt / durationMs
inputRef hashes / outputRef hashes
retryAttempt / releaseMode / riskLevel
```

Raw Prompt、完整 Source File、Secret、Credential、Approval Proof 與模型私有推理不得放入 Span Attribute。

## 15. Metrics

### 15.1 Outcome

```text
accepted_task_rate = accepted tasks / eligible tasks
rework_rate = tasks entering fix-from-review / reviewed tasks
rollback_rate = rollback changes / released changes
incident_rate = declared incidents / released changes
```

`eligible tasks`、`accepted tasks` 與觀測視窗必須在 Metric Definition 中明確定義。取消、測試 Run 與缺少必要資料的 Run 不應默默混入分母。

### 15.2 Quality 與 Capability

- Requirement Coverage、Validation Pass 與 Evidence Completeness。
- Route Accuracy、Domain Mismatch 與 Clarification Accuracy。
- Context Authority Coverage、Freshness Failure 與 Source Overuse。
- Review Finding Severity、Human Correction 與 Unsupported Claim。
- Tool Selection、Argument Validation 與 Authorization Denial。

### 15.3 Flow 與 Reliability

- Run／Stage Latency 的 p50、p95、p99。
- Queue、Handoff、Human Gate 與 Dependency Wait。
- Retry Attempt、Retry Amplification、Timeout 與 Fallback Success。
- Schema Invalid、State Conflict、Partial Write、Lease Expiry 與 Unknown Outcome。
- Checkpoint Resume、Cancel、Abandoned Run 與 Orphan Candidate。

平均值不足以描述長尾等待；Latency 與 Cost 至少保存分布或 Percentile。

### 15.4 Governance 與 Safety

- Unauthorized-action Rejection。
- Scope Violation、Policy Version Mismatch 與 Approval Invalidation。
- Secret／Sensitive Data Redaction Failure。
- Replay Side-effect Blocked。
- Gate Bypass、Accepted Pointer Conflict 與 Provenance Gap。

Denial 數量上升可能來自攻擊、Policy 變更或正常風險控制。Alert 必須能下鑽至 Cohort 與 Reason Code，不能將所有 Denial 視為 Agent Failure。

### 15.5 Efficiency

```text
cost_per_accepted_task
tokens_per_accepted_artifact
tool_calls_per_successful_stage
retry_cost_amplification
context_bytes_or_tokens_by_source
cache_hit_rate
```

成本以 Accepted Task 或 Successful Stage 為主要分母。只看 Cost per Run，系統可以藉由提前失敗降低數值，卻沒有提高交付效率。

## 16. Metric Label 與 Cardinality

適合放入 Metric Label：

```text
environment
workflow_id / workflow_version
stage
domain
risk_level
release_mode
status
reason_code
model_family / tool_id
```

下列 Identity 通常具有高基數，應保留在 Event、Trace 或 Exemplars：

```text
changeId
runId
invocationId
artifact URI / checksum
userId / repository path
raw error message
prompt or tool payload
```

Runtime Host 應限制 Label Length、Value Set 與未知值。自由文字 Error 若直接成為 Label，會增加儲存成本，也可能把敏感內容送出 Trust Boundary。

## 17. Regression 與 Drift

### 17.1 可比較性檢查

計算 Candidate Delta 前，Evaluator 先比較：

- Dataset、Rubric、Metric Definition 與 Threshold Version。
- Domain、Risk、Language、Repository 與 Release Cohort。
- Model、Prompt、Router、Tool、Policy 與 Environment。
- Sample Size、Sampling Method、Timeout 與 Retry Policy。

差異超過 Comparison Policy 的允許範圍時，Result 使用 `not_comparable`。報告可以並列原始數字，不能宣告改善百分比。

### 17.2 Regression 判定

Regression Policy 應同時包含：

- Hard Invariant：任何失敗都阻擋 Promotion。
- Absolute Threshold：例如 Domain Mismatch 必須低於固定上限。
- Relative Delta：Candidate 相對 Baseline 的允許退化。
- Cohort Rule：高風險 Domain 使用較嚴格條件。
- Minimum Sample：樣本不足時回傳 `inconclusive`。

### 17.3 Drift

Online Drift Detection 至少保存觀測視窗、Reference Window、Cohort、Feature／Metric Definition 與 Confidence。偵測到 Drift 後，Orchestrator建立診斷工作或 Candidate Evaluation，不能直接改寫 Router、Prompt、Threshold 或 Context Policy。

10 號文件的 Replay Drift 針對固定 Incident 輸入與原始 Decision；本文件的 Online Drift 針對一段時間的執行分布。兩者可以共用 Trace 與 Comparison Policy，結論維持各自 Scope。

## 18. SLI、SLO 與 Error Budget

### 18.1 SLI

SLI 必須能由明確事件計算，例如：

```text
workflow_acceptance_sli
stage_completion_latency_sli
required_evidence_availability_sli
state_mutation_success_sli
high_risk_gate_bypass_sli
```

`品質很好`、`Agent 很聰明` 或單一 LLM Judge 分數不適合作為營運 SLI。

### 18.2 SLO

每個 SLO 應列出：

- Service／Workflow 與適用 Cohort。
- SLI Formula、資料來源與觀測視窗。
- Target、Excluded Event 與 Minimum Traffic。
- Error Budget、Alert Policy 與 Owner。
- Breach 後允許的 Action。

安全 Invariant 例如外部 Write Gate Bypass，Target 應為 0，且不使用一般 Error Budget 放寬。Latency、Availability 與非高風險 Retry 可依服務條件設定 Budget。

### 18.3 SLO Breach

SLO Breach 可以暫停 Progressive Rollout、限制高風險 Cohort、切回 Stable Release 或要求人工判斷。它不授權 Orchestrator 自動修改 Requirement、Rubric、Permission 或 Human Approval Policy。

## 19. Alert 與路由

### 19.1 Alert 條件

Alert 適合處理：

- Hard Invariant、Integrity、Security 或 Gate Bypass。
- Error Rate、Latency、Retry 或 Cost 在觀測視窗內超過 Threshold。
- Telemetry Pipeline 出現資料缺口或 Redaction Failure。
- Candidate Canary 相對 Stable Cohort 產生持續 Regression。
- Provenance、Audit 或 Required Evidence 無法取得。

單一正常 Validation Failure 應由 Workflow Fallback 處理，無需每次通知 Operator。Alert Policy 要設定 Window、Minimum Count、Deduplication Key、Cooldown、Severity 與 Owner。

### 19.2 Severity 與預設路由

| Severity | 條件 | 預設路由 |
|---|---|---|
| Critical | Unauthorized Side Effect、Secret Leakage、Gate Bypass、Integrity Failure | 停止相關 Release／Action，通知 Human／Incident Operator |
| High | 高風險 Cohort Regression、SLO 快速消耗、Required Evidence Gap | 暫停 Rollout，建立 Coordinator／Reviewer 工作 |
| Medium | Retry、Latency、Cost 或 Fallback 持續偏離 Baseline | 建立診斷項目，維持安全流量 |
| Low | 趨勢變化、低樣本 Drift、非阻斷 Telemetry Gap | 記錄並進入排程 Review |

Alert 訊息只帶安全摘要、Cohort、Reason Code、Observed Value、Threshold 與 Trace／Dashboard Reference。敏感資料留在受控 Store。

## 20. Sampling、Retention 與 Redaction

### 20.1 Sampling

Host 可以抽樣成功 Run 的詳細 Trace，並完整保留 Metrics 與必要 Audit Record。下列事件預設保留：

- Hard Invariant Failure、Policy Denial 與 Approval Invalidation。
- Incident、Rollback、Human Override 與 Canary Abort。
- Integrity、Provenance、State Mutation 與 Unknown Side-effect Outcome。
- Evaluation Regression、Inconclusive Result 與 Telemetry Redaction Failure。

Tail-based Sampling 可以在 Run 結束後依 Status、Latency 或 Reason Code 決定保留完整 Trace。抽樣 Policy 本身要版本化。

### 20.2 Retention

| 資料 | 建議原則 |
|---|---|
| Aggregate Metrics | 依趨勢與容量規劃保存較長期間 |
| Full Trace | 依成本、隱私與診斷週期保存較短期間 |
| Evaluation Manifest／Report | 至少涵蓋 Release、Rollback 與 Regression 比較週期 |
| Case Result | 依 Dataset 與法遵要求保存，可保留摘要與失敗樣本 |
| Evidence／Audit | 依工程稽核、Human Approval 與 Incident Policy |

Retention 到期後，報告要標示資料缺口。系統不得改用內容不同的 Latest Artifact 重建舊數字。

### 20.3 Redaction

Telemetry Pipeline 必須在 Export 前處理：

- Secret、Credential、Cookie、Token 與 Approval Proof。
- 個資、客戶內容、私有 Repository 片段與完整 Prompt。
- Tool Argument／Output 中不需診斷的欄位。
- 模型私有推理與未篩選的聊天歷史。

需要比對內容時，可以保存經正規化的 Hash、Data Classification 與受控 Reference。Hash 不能讓低權限 Agent 推測敏感值是否存在。

## 21. Evaluation Gate 與 Accepted Pointer

### 21.1 Decision Flow

```text
publish immutable Candidate
→ build Evaluation Manifest
→ run Evaluation Suite
→ validate Result and Evidence Binding
→ Reviewer evaluates findings and residual risk
→ Orchestrator applies versioned Gate Policy
→ Human approves high-risk Action when required
→ CAS State and Accepted Pointer
→ record Mutation Receipt, Provenance and Trace
```

Evaluation `pass` 只是 Gate Input。Review、Readiness、Human Approval、State Revision 與 Candidate Binding 仍可能使 Promotion 停止。

### 21.2 Gate Invalidation

Candidate、Dataset、Rubric、Policy、Environment 或 Required Evidence 改變時，相關 Evaluation Gate 失效。Orchestrator 應建立新 Manifest 與 Result；它不能修改舊 Report 的 Subject Reference，再宣告沿用。

### 21.3 現行 Phase

Current State Phase Enum 沒有 `EVALUATING`、`OBSERVING`、`DEGRADED` 或 `CANARY_FAILED`。Evaluation 可以由獨立 Host Workflow 執行；主 Run 需要呈現阻斷狀態時，使用合法的 `INCOMPLETE`、`FAILED`、`CHANGES_REQUESTED` 或 `NEEDS_COORDINATOR_ARBITRATION`，並在 Blocker、Handoff Reason 或 Reference 中標示 Evaluation 結果。

## 22. 固定失敗代碼與路由

### 22.1 Evaluation

| Code | 判定條件 | 安全路由 |
|---|---|---|
| `EVALUATION_SUBJECT_MISSING` | Subject Reference 或 Release Unit 不完整 | `INCOMPLETE`，補齊固定版本 |
| `EVALUATION_BINDING_MISMATCH` | Result、Evidence、Candidate 或 Baseline 指向不同 Target | 停止 Gate，重新建立 Manifest |
| `EVALUATION_DATASET_UNAVAILABLE` | Dataset、Case 或 Hash 無法取得 | `inconclusive`，不得沿用舊 Dataset |
| `EVALUATION_RUBRIC_VERSION_MISSING` | Rubric 或 Threshold Version 不明 | `inconclusive`，交回 Coordinator |
| `EVALUATION_POLICY_VERSION_MISSING` | Gate／Sampling／Comparison Policy 無法取得 | 停止 Promotion |
| `EVALUATION_EVIDENCE_INCOMPLETE` | 必要 Case Result、Trace 或 Evidence 缺失 | `INCOMPLETE`，列出缺失 Reference |
| `EVALUATION_INVARIANT_FAILED` | Hard Invariant 失敗 | `fail`，依 Failure 類型停止或隔離 |
| `EVALUATION_RESULT_INCONCLUSIVE` | 樣本、Evaluator 一致性或資料不足 | 擴充受控樣本或要求 Human Review |
| `EVALUATION_BASELINE_INCOMPATIBLE` | Release Unit、Dataset、Rubric 或 Cohort 不可比較 | 回傳 `not_comparable` |
| `EVALUATION_REGRESSION_DETECTED` | Candidate 超過允許退化或高風險 Cohort 失敗 | 暫停 Rollout，建立修正 Handoff |

### 22.2 Observability

| Code | 判定條件 | 安全路由 |
|---|---|---|
| `TELEMETRY_BINDING_MISSING` | Event／Trace 無法連結 Run、Stage 或 Subject | 隔離資料，標示報告限制 |
| `TELEMETRY_REDACTION_FAILED` | Export 前無法確認敏感內容已處理 | 停止 Export，保留本機安全 Event |
| `TELEMETRY_CARDINALITY_LIMIT` | Label／Attribute 超過 Policy | 丟棄不合規 Label，保留固定 Code 與計數 |
| `OBSERVABILITY_SIGNAL_GAP` | Metrics、Event、Trace 或時間區段缺失 | `inconclusive`，不得補算確定結論 |
| `ALERT_POLICY_INVALID` | Threshold、Window、Owner 或路由不完整 | 停用該 Rule 並通知 Operator |
| `ALERT_THRESHOLD_BREACHED` | SLI／Metric 超過版本化 Threshold | 依 Severity 與 Release Policy 路由 |

### 22.3 重用既有代碼

下列條件沿用既有規範：

| 條件 | 代碼來源 |
|---|---|
| Context Authority、Freshness、Version 或 Binding | 06 的 `CONTEXT_*` |
| Authorization、Scope、Approval 或 Policy | 07 的固定拒絕代碼 |
| State、Mutation、Retry、Conflict 或外部結果不明 | 08 的固定代碼 |
| Promotion、Gate Invalidation 與 Rollback | 09 的固定代碼 |
| Provenance、Audit、Replay 與 Incident Drift | 10 的固定代碼 |

同一條件只能使用一個主要 Reason Code。Evaluation Report 可以補充分類，不能為了 Dashboard 另造同義代碼。

## 23. 現行 Schema 相容方式

### 23.1 Handoff Envelope

[`handoff-envelope.schema.json`](../schemas/handoff-envelope.schema.json) 沒有 `evaluationId`、`datasetVersion`、`rubricVersion`、`traceId` 或 `metricThresholds`。現行 Handoff 可以使用：

- `requiredInputRefs` 指向 Evaluation Manifest、Report、Evidence 與 Baseline。
- `acceptanceCriteriaRefs` 指向 Rubric、Regression Policy 與 Gate 條件。
- `reason` 說明評估目的、Cohort、Hard Invariant 與禁止的 Side Effect。
- `onSuccess`、`onFailure`、`onConflict` 使用現有合法 Transition。

獨立 Evaluation 不可新增 `evaluate-workflow` Stage。Host 以 Invocation Metadata 與 Trace 保存 `evaluationId`，再把 Report Reference 交給現有 Reviewer 或 Readiness Stage。

### 23.2 Agent Result

[`agent-result.schema.json`](../schemas/agent-result.schema.json) 沒有 `evaluation_result` Kind。現行 Reviewer／Readiness Result 可以：

- 在 `inputRefs` 引用 Evaluation Manifest 與 Candidate。
- 在 `outputRefs` 引用獨立 Evaluation Report／Evidence。
- 在 `verification` 保存已執行 Suite 的名稱、Status、Command 與 Evidence Ref。
- 在 `reviewPayload.reviewedArtifacts` 或 `readinessPayload.gates` 表達對 Evaluation Result 的審查。

各 Kind 的 `payload` 受 `additionalProperties: false` 約束，不能加入未定義的 `metrics`、`traceId` 或 `evaluationVerdict`。完整結果保存為獨立 Artifact。

### 23.3 Current State

[`current-state.schema.json`](../schemas/current-state.schema.json) 只保存目前 Phase、Owner、Revision、Accepted Artifact、Handoff、Gate、Blocker 與 Next Action。它不保存 Metrics History、Evaluation Dataset、Trace 或 SLO。

現行 Gate 沒有 `evaluationPassed`。Orchestrator 依 Workflow Policy 將 Evaluation Report 當成 Review／Readiness 的必要輸入；Schema 正式新增 Gate 前，不得在 `gates` 物件加入額外欄位。

### 23.4 Conceptual Record

本文件的 Evaluation Manifest、Evaluation Result、Telemetry Event 與 Alert Record 都是概念格式。團隊若要讓不同 Runner 或 Service 交換這些資料，應先建立 Versioned Schema、Compatibility Rule、Migration Policy 與 Example Fixture。

## 24. 概念性紀錄

### 24.1 Evaluation Manifest

```yaml
schemaVersion: 0.1.0-draft
evaluationId: eval-structured-artifact-v2-001
changeId: prompt-runtime-v2
runId: run-eval-001
mode: offline

subject:
  type: workflow-release-unit
  ref: repo://prompt-engineering/templates/workflow.example.yaml
  version: 2.0.0
  sha256: aaaa
  promptLockRef: repo://prompt-engineering/prompts/structured-artifact-generation/prompt-lock.json

baseline:
  ref: artifact://prompt-runtime/releases/structured-artifact-v1.json
  sha256: bbbb

dataset:
  ref: repo://prompt-engineering/prompts/structured-artifact-generation/eval-cases.yaml
  version: 1.1.0
  sha256: cccc

rubric:
  ref: repo://prompt-engineering/evaluation/structured-artifact-rubric.yaml
  version: 1.0.0

policyVersion: evaluation-policy-1.0.0
environmentRef: runtime://evaluation/environments/local-001.json
cohorts:
  - normal
  - cross-domain
  - prompt-injection
  - high-risk-action
sideEffects: denied
createdAt: 2026-08-15T09:00:00Z
traceId: trace-eval-001
```

`sha256` 範例為縮寫，正式資料必須符合採用 Schema 的完整格式。

### 24.2 Evaluation Result

```yaml
schemaVersion: 0.1.0-draft
evaluationId: eval-structured-artifact-v2-001
executionStatus: completed
verdict: fail

binding:
  subjectHash: aaaa
  baselineHash: bbbb
  datasetHash: cccc
  rubricVersion: 1.0.0
  policyVersion: evaluation-policy-1.0.0

summary:
  totalCases: 40
  passedCases: 39
  failedCases: 1
  inconclusiveCases: 0

invariantChecks:
  - id: domain-failure-must-stop-artifact-generation
    status: failed
    caseId: cross-domain-action-07
    code: EVALUATION_INVARIANT_FAILED

regressions:
  - cohort: cross-domain
    baselinePassRate: 1.0
    candidatePassRate: 0.875
    code: EVALUATION_REGRESSION_DETECTED

evidenceRefs:
  - evidence://prompt-runtime/evaluations/eval-structured-artifact-v2-001/case-results.json
  - runtime://prompt-runtime/telemetry/traces/trace-eval-001.json

nextAllowedAction: fix-workflow-gate
createdAt: 2026-08-15T09:08:12Z
traceId: trace-eval-001
```

### 24.3 Telemetry Event

```yaml
schemaVersion: 0.1.0-draft
eventId: evt-eval-001-domain-validation
eventType: validation-completed
changeId: prompt-runtime-v2
runId: run-eval-001
stage: apply-change
evaluationId: eval-structured-artifact-v2-001
caseRunId: case-run-cross-domain-action-07

actor:
  actorId: automation:evaluation-host-local
  role: automation

operation:
  nodeId: validate_spec_domain
  status: failed
  reasonCode: DOMAIN_FORBIDDEN_ACTION
  startedAt: 2026-08-15T09:04:01Z
  endedAt: 2026-08-15T09:04:01Z
  durationMs: 18

binding:
  workflowVersion: 2.0.0
  policyVersion: evaluation-policy-1.0.0
  inputHash: dddd
  outputHash: eeee

traceId: trace-eval-001
spanId: span-domain-validation-07
parentSpanId: span-case-run-07
```

### 24.4 Evaluation Report

Report 至少包含：

```text
Manifest Reference and integrity status
Baseline／Candidate comparability
Dataset and cohort coverage
Hard Invariant results
Metric definitions and deltas
Failed／inconclusive case references
Evaluator identity and limitations
Regression classification
recommended next allowed action
```

Report 的 `recommendation` 不等於 State Transition。Orchestrator 仍須載入最新 Current State、Gate、Approval 與 Expected Revision。

## 25. 本機完整範例

### 25.1 Baseline

團隊目前使用 Structured Artifact Workflow v1。它依序執行 Domain Router、Skill Loader、Context Builder、Structured Spec、Schema Validation、Domain Validation 與 Artifact Transformer。v1 已通過 Dataset 1.1.0，並成為 Stable Baseline。

### 25.2 Candidate v2

Candidate v2 調整 Router 與 Retry Flow。Offline Evaluation 顯示：

```text
overall case pass rate: 92.5% → 97.5%
normal cohort latency p95: 840 ms → 710 ms
cross-domain cohort pass rate: 100% → 87.5%
hard invariant: failed
```

整體成功率與 Latency 改善，但 `cross-domain-action-07` 的 Domain Validator 回傳 Failure 後，Workflow 仍執行 `generate_artifact`。這違反「Domain Validation Failure 必須阻擋 Artifact Node」的 Hard Invariant。

Evaluation Host 產生 `EVALUATION_INVARIANT_FAILED` 與 `EVALUATION_REGRESSION_DETECTED`，並將 Case Result、Workflow Version、Node Trace 與 Candidate Hash 寫入 Report。Reviewer 確認 Target Binding 後回傳 `request_changes`。Orchestrator 保留 v1 Accepted Pointer，v2 維持 Candidate。

### 25.3 Candidate v3

Implementer 修正 `generate_artifact.when` 與 Domain Failure Route，建立 v3 Candidate。Orchestrator 使用相同 Dataset、Rubric、Policy 與相容環境重新執行 Suite：

```text
all hard invariants: passed
cross-domain cohort: 8 / 8 passed
normal cohort latency p95: 735 ms
evaluation verdict: pass
```

v3 先進 Shadow，Candidate Output 不會控制外部 Action。Shadow 沒有出現 Gate Bypass 或新的 Domain Mismatch 後，Human 批准受控 Canary。Canary 的 Metrics 與 Trace 仍綁定 v3 Release Unit；若 Prompt、Router、Workflow 或 Policy 改變，原結果立即失效。

### 25.4 Promotion

Reviewer 的 `approve`、Evaluation `pass` 與 Canary Healthy 都是 Promotion Input。Orchestrator 最後檢查 Readiness、Human Approval、Current State Revision 與 Candidate Binding，才以 CAS 更新 Accepted Pointer，並保存 Mutation Receipt、Provenance Event 與 Trace。

## 26. 最小本機導入

### 26.1 可選目錄

```text
.agent-runtime/<change-id>/
├─ current-state.json
├─ runs/
│  └─ <run-id>/
│     ├─ artifacts/
│     ├─ evidence/
│     └─ evaluations/
│        └─ <evaluation-id>/
│           ├─ evaluation-manifest.json
│           ├─ case-results/
│           └─ evaluation-report.json
├─ telemetry/
│  ├─ events/
│  ├─ traces/
│  └─ metric-snapshots/
└─ host-journal/
   ├─ provenance/
   └─ mutation-receipts/
```

`evaluations/` 與 `telemetry/` 是 Git-ignored Runtime Data。可長期維護的 Dataset、Rubric、Metric Definition 與 Evaluation Policy 應進入 Git 或受治理的 Registry。Agent 只讀 Handoff 授權的 Reference，不能列舉整個 Telemetry 或 Host Journal。

### 26.2 最小 Host 介面

```text
createEvaluationManifest(subject, baseline, dataset, rubric, policy)
runEvaluationSuite(manifest)
appendTelemetryEvent(event)
publishEvaluationResult(result)
verifyEvaluationBinding(evaluationId)
compareBaselineAndCandidate(evaluationId)
evaluateReleaseGate(reportRef, stateRevision)
recordMetricSnapshot(scope, window)
applyTelemetryRetention(policy)
```

Host 應使用 Create-only Artifact、Atomic Write 與固定 Reference。Evaluator 介面不提供 `updateCurrentState()` 或 `acceptCandidate()`。

### 26.3 最小落地順序

1. 選一條可重複執行的 Workflow Vertical Slice。
2. 固定 Workflow、Prompt、Router、Model、Tool 與 Policy Version。
3. 從 5 至 10 個 Normal、Boundary 與 Failure Case 建立 Dataset。
4. 先執行 Schema、Domain、Binding 與 Side-effect Hard Invariant。
5. 每個 Run 保存 Stage、Status、Reason Code、Duration、Attempt 與 Reference。
6. 產生一份 Baseline／Candidate Evaluation Report。
7. 讓 Reviewer 讀取 Report，Orchestrator 依現有 Gate 決定路由。
8. 完整保留 Failure Trace，成功 Trace 依 Policy 抽樣。
9. 用一個已知 Gate Bypass 或 Binding Error Fixture 驗證 Alert 與 Fail-closed 行為。

第一版不需要 Dashboard、Distributed Trace Backend 或自動 Drift Model。命令列報告、JSON Artifact 與本機 Trace 已足以驗證資料綁定和決策邊界。

## 27. Production 升級判準

符合任一條件時，團隊應評估導入 OpenTelemetry 或等價的 Telemetry Standard、集中式 Metrics／Trace Backend、Evaluation Registry 與 Alert Routing：

- 多個 Orchestrator、Worker、Repository 或環境共同執行 Workflow。
- 需要跨服務串接 Client、Router、Model、Tool、Artifact 與 State Mutation。
- Evaluation Dataset、Prompt、Policy 與 Release Unit 數量已無法用本機檔案索引。
- Canary、Shadow 與多版本 Cohort 需要近即時比較。
- SLO、Cost Budget、On-call 或 Incident Response 依賴可靠 Alert。
- 法遵要求資料分區、Access Audit、Legal Hold、WORM 或跨區限制。
- Telemetry 量已需要 Tail Sampling、Cardinality Control 與分層 Retention。
- Evaluation Gate 需要在 CI／CD 或 Workflow Runtime 中自動執行。

升級後仍應保留相同語意：

```text
evaluation subject is immutable
baseline and candidate are explicitly bound
hard invariants fail closed
metrics do not replace evidence
trace does not replace audit
evaluator does not own state transition
accepted pointer changes through policy and CAS
```

單機、Single Writer、低 Run 量且沒有法遵要求時，本機 Evaluation Artifact 與 Trace 已能支援基本 Regression Gate。團隊可以先讓一條 Workflow 的資料可信，再增加平台元件。

## 28. 驗收清單

### Subject 與 Dataset

- [ ] Evaluation Manifest 固定 Subject、Baseline、Dataset、Rubric、Policy 與環境。
- [ ] Prompt、Router、Workflow、Model、Tool 與 Adapter 使用精確版本或 Hash。
- [ ] Dataset 有 Owner、Version、Risk Tag、Change History 與 Data Classification。
- [ ] Case 包含 Normal、Boundary、Negative、Cross-domain、Injection 與 Side-effect 情境。
- [ ] Dataset 變更會產生新 Version，不會原地修改舊 Result。
- [ ] Incident 修正附帶最小可重現 Regression Case。

### Evaluation

- [ ] Completion Status 與品質 Verdict 分開保存。
- [ ] Hard Invariant 不會被 Aggregate Score 抵銷。
- [ ] Candidate 與 Baseline 不可比較時回傳 `not_comparable`。
- [ ] 高風險 Cohort 有獨立 Threshold 與樣本數。
- [ ] Evaluator Identity、Model、Prompt、Rubric 與限制可追溯。
- [ ] Producer 自我檢查不會直接成為 Release Gate Verdict。
- [ ] Evaluation Result 綁定 Candidate、Evidence 與 Dataset Hash。

### Observability

- [ ] Metrics、Event、Trace、Evidence 與 Audit Record 分層保存。
- [ ] Run、Stage、Invocation、Evaluation 與 Mutation Identity 可區分。
- [ ] Async Handoff 與 Shadow Candidate 使用 Span Link 或明確 Reference。
- [ ] Latency 與 Cost 保存 Distribution／Percentile，不只保存平均值。
- [ ] 高基數 Identity 不進 Metric Label。
- [ ] Event 使用固定 Status 與 Reason Code，Raw Error 保存在受控 Trace。
- [ ] Signal Gap 會標示 `inconclusive`，不會補算確定結論。

### Gate 與 Lifecycle

- [ ] Evaluation Result 不能自行更新 Current State 或 Accepted Pointer。
- [ ] Reviewer Verdict、Orchestrator Transition 與 Human Approval 維持分離。
- [ ] Candidate、Rubric、Policy 或 Evidence 改變後，舊 Gate 會失效。
- [ ] Shadow／Replay／Simulation 預設封鎖外部 Write。
- [ ] Promotion 使用最新 State Revision、Gate、Approval 與 Candidate Binding。
- [ ] Current State 沒有加入未定義的 Evaluation Phase 或 Gate。
- [ ] Handoff 與 Agent Result 沒有加入 Schema 未定義欄位。

### Safety、Retention 與 Operations

- [ ] Telemetry Export 前執行 Redaction 與 Data Classification。
- [ ] Trace 不保存 Secret、Credential、Approval Proof、完整敏感 Payload 或模型私有推理。
- [ ] Incident、Policy Denial、Human Override 與 Integrity Failure 不會因抽樣遺失 Audit Record。
- [ ] Metrics、Trace、Evaluation Report、Evidence 與 Audit 有各自 Retention。
- [ ] Alert Rule 有 Window、Threshold、Minimum Count、Owner、Cooldown 與路由。
- [ ] Critical Alert 可以暫停相關 Release 或高風險 Action。
- [ ] Telemetry Backend 不會擴張 Agent 的 Repository、Runtime 或客戶資料存取權限。

## 29. 參考文件

- [Prompt Package 接入 Workflow 的 Runtime Execution](../../../ai-agent-prompt-runtime-workflow-phase2-zh-TW.md)
- [Artifact-based Shared State + Structured Handoff 參考架構](./02-reference-architecture-zh-TW.md)
- [Repository Knowledge、Runtime State、Evidence 與 Trace 分層](./03-runtime-storage-and-retention-zh-TW.md)
- [安全、治理與長期維護](./05-security-and-maintainability-zh-TW.md)
- [Context Authority 與 Retrieval Policy](./06-context-authority-and-retrieval-policy-zh-TW.md)
- [Role Capability 與 Scope Control](./07-role-capability-and-scope-control-zh-TW.md)
- [Mutation、Concurrency 與 Conflict Resolution](./08-mutation-concurrency-and-conflict-resolution-zh-TW.md)
- [Context Promotion 與 Rollback](./09-context-promotion-and-rollback-zh-TW.md)
- [Provenance Audit 與 Incident Replay](./10-provenance-audit-and-incident-replay-zh-TW.md)
- [Context Engineering Core](../../../context-engineering/docs/01-context-engineering-core-zh-TW.md)
- [Agent Platform Operations](../../../context-engineering/docs/03-agent-platform-operations-zh-TW.md)
- [Tool Governance、Evaluation 與 Observability](../../../agent-design/tool-schema-routing/docs/03-tool-governance-and-evaluation-zh-TW.md)
- [Handoff Envelope Schema](../schemas/handoff-envelope.schema.json)
- [Current State Schema](../schemas/current-state.schema.json)
- [Agent Result Schema](../schemas/agent-result.schema.json)
- [Workflow Policy Template](../templates/workflow-policy.template.yaml)
