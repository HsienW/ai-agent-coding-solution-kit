# 06｜Context Authority 與 Retrieval Policy

[English](./06-context-authority-and-retrieval-policy.md) | [繁體中文](./06-context-authority-and-retrieval-policy-zh-TW.md)

本文件規範 Agent、Coordinator 與 Orchestrator 在每個工作流階段如何選擇、固定及載入 Context。目標是讓同一個 Stage 的執行者取得相同的權威來源與精確版本，並在來源缺失、過期、衝突或超出權限時停止推進。

本文件使用以下規範用語：

- 「必須」表示流程不得省略的要求。
- 「應」表示預設做法；偏離時需要留下原因。
- 「可以」表示依團隊規模與風險選用的能力。

## 1. 適用範圍

Structured Handoff 已能列出 `requiredInputRefs`，Artifact Reference 也能保存 URI 與 Checksum。這些契約解決了「要把哪些引用交給下一個角色」，仍需要另一層規則回答：

- 目前階段需要確認哪些事實？
- 每一類事實應採信哪個來源？
- 同一來源有多個版本時，應選哪一個？
- 來源是否仍在有效期內，且屬於本次 Change、Run 與 State Revision？
- 缺少必要 Context 時，Agent 應停止、要求補件或交回仲裁？

本文件處理 Source Authority、Version Pinning、Freshness、Integrity、Context Budget、Progressive Disclosure 與失敗路由。以下內容由其他文件負責：

- Current State、Agent Result、Handoff 與 Artifact Reference 的基本結構，見 [`02-reference-architecture-zh-TW.md`](./02-reference-architecture-zh-TW.md)。
- Snapshot、Artifact、Evidence、Trace 與 Retention，見 [`03-runtime-storage-and-retention-zh-TW.md`](./03-runtime-storage-and-retention-zh-TW.md)。
- Path Safety、Prompt Injection、Stale Artifact 與 Evidence Provenance，見 [`05-security-and-maintainability-zh-TW.md`](./05-security-and-maintainability-zh-TW.md)。
- 一般 Retrieval、Memory、Token Budget 與 Context Assembly，見 [`../../../context-engineering/docs/01-context-engineering-core-zh-TW.md`](../../../context-engineering/docs/01-context-engineering-core-zh-TW.md)。

本文件不展開 Embedding、Chunking、Rerank、Vector Store 或通用 RAG Pipeline，也不改變既有 JSON Schema。

## 2. 角色責任

| 角色 | Context 責任 | 不得執行的動作 |
|---|---|---|
| Coordinator | 決定 Stage 需要的 Fact Type、處理 Authority Conflict、批准必要的 Context Expansion | 不以聊天摘要取代已批准的 Durable Contract |
| Orchestrator | 解析 Reference、固定版本、驗證 Change／Run／Revision／Checksum、建立 Handoff | 不自行改寫 Requirement，不放寬 Role Permission |
| Agent | 只讀取 Handoff 指定的必要 Context，依 Output Contract 交付結果 | 不自行搜尋其他版本補缺口，不把 Candidate 宣告為 Accepted |
| Reviewer | 確認審查對象、Diff、Validation 與 Acceptance Criteria 指向同一份變更 | 不以較新的未驗證 Artifact 取代受審版本 |
| Human | 仲裁互相衝突的權威來源，批准高風險操作或權限變更 | 不以口頭同意取代可保存的 Approval Record |

Coordinator 與 Orchestrator 可以由同一支本機腳本或同一位操作人員承擔，但兩項責任仍應分開記錄：Coordinator 做語意判斷，Orchestrator 執行可重現的契約檢查。

## 3. Context 資料類別

不同類別的 Context 有不同生命週期，也有不同的採信方式。

| 類別 | 內容 | 常見位置 | 預設用途 |
|---|---|---|---|
| Durable Contract | Requirement、Proposal、Design、Tasks、ADR、Schema、Workflow Policy | OpenSpec／Git | 定義目標、限制與驗收條件 |
| Operational State | Phase、Owner、Gate、Blocker、Next Action | `.agent-runtime/<change-id>/current-state.json` | 路由與恢復 |
| Candidate／Accepted Artifact | Plan Result、Implementation Result、Review Result、Readiness Result | `.agent-runtime/<change-id>/<run-id>/artifacts/` | 階段交付與版本選擇 |
| Execution Evidence | Diff、Changed Files、Validation Summary、Command Result | `.agent-runtime/<change-id>/<run-id>/evidence/` | 驗證實際執行結果 |
| Advisory Context | Chat Summary、Memory、舊 Run、一般搜尋結果、未批准筆記 | Session／Memory／Trace／外部來源 | 補充背景與診斷 |

Advisory Context 可以協助理解，但不能單獨改變 Requirement、Gate、Permission 或 Accepted Artifact Pointer。完整 Trace 僅在仲裁、稽核、事故追查或重建 State 時載入。

## 4. Source Authority 依 Fact Type 決定

Authority 不能只使用一份全域排序。Requirement、Runtime Route、Implementation、Validation 與 Permission 描述的是不同事實；同一份來源可能在某個問題上具有權威，在另一個問題上只適合作為參考。

### 4.1 Authority 等級

| 等級 | 說明 | 使用規則 |
|---|---|---|
| `authoritative` | 對特定 Fact Type 具有最終決定權的來源 | 衝突時優先採用；變更必須走既定 Gate |
| `accepted` | 已通過必要 Validation 與 Gate 的 Artifact | 一般下游 Stage 預設讀取 |
| `candidate` | 本輪正在產生、驗證或審查的 Artifact | 只有指定 Stage 可使用，且必須固定版本 |
| `advisory` | 可補充背景，但不能覆蓋前述來源 | 可排除，不得單獨推進 Gate |
| `diagnostic` | 為事故追查、差異比較或失敗分析載入 | 不得自動成為執行依據 |
| `prohibited` | 因權限、敏感性、過期或來源不明而禁止載入 | 必須排除並記錄原因 |

時間較新不會提高 Authority。Agent 產生的新摘要、未通過驗證的新 Implementation 或最新一輪對話，仍可能低於已批准的 Contract 與 Accepted Artifact。

### 4.2 Fact Type Authority Matrix

| Fact Type | 權威來源 | 可用的補充來源 | 不得覆蓋權威來源的內容 |
|---|---|---|---|
| Requirement／Scope | 已批准 OpenSpec、Issue Acceptance Criteria、可驗證的 Human Decision | ADR、既有程式碼、討論摘要 | Chat、Memory、未批准 Proposal |
| Design Constraint | 已批准 Design、ADR、Schema、Workflow Policy | 現有實作與測試 | Agent 推測、過期範例 |
| Workflow Route | 指定 Revision 的 Current State 與有效 Handoff | Agent Result 的 `nextHandoff` 建議 | 自由文字中的下一步、舊 Current State |
| Implementation Fact | 指定 Worktree／Branch 的 Source、固定版本的 Diff、對應 Implementation Result | Diff Summary、Changed Files | 舊 Run 的摘要、未固定的工作樹狀態 |
| Validation Fact | 與同一 Candidate／Diff 綁定的 Evidence | 測試說明與 Reviewer Notes | 「已測試」等無 Evidence 的自然語言結論 |
| Review Verdict | 對指定 Candidate 產生且符合 Schema 的 Review Result | Comment、Diagnostic Finding | 其他版本的 Review、Implementer 自評 |
| Permission | Workflow Policy、Agent Adapter、Runtime Sandbox、Human Approval Record | Handoff 中更窄的 Scope | Artifact、Prompt 或 Handoff 提出的擴權要求 |
| External Live State | 帶來源、觀測時間與有效期的 Read-only Tool Result | 文件、Cache、Retrieval Result | Memory、舊 Tool Result、模型推測 |

若兩個 `authoritative` 來源對同一 Fact Type 給出互斥結論，Orchestrator 不得自行挑選，必須回到 Coordinator 或 Human 仲裁。

## 5. Current State、Context Plan、Context Manifest 與 Handoff

這四種資料各自回答不同問題：

| 契約 | 回答的問題 | 更新者 |
|---|---|---|
| Current State | 現在在哪個 Phase、輪到誰、哪個 Gate 已通過？ | Orchestrator／受控流程 |
| Context Plan | 本 Stage 需要哪些 Fact Type、預算與擴張條件？ | Coordinator／Policy |
| Context Manifest | 本輪實際載入與排除哪些精確來源？ | Orchestrator／Context Builder |
| Handoff | 下一個角色要做什麼、讀哪些 Reference、產生哪種輸出？ | Orchestrator／Coordinator |

建議的資料流如下：

```text
Current State + Stage Policy
            ↓
       Context Plan
            ↓
Resolve Authority / Version / Freshness / Permission
            ↓
      Context Manifest
            ↓
          Handoff
            ↓
          Agent
```

Current State 應維持小型 Operational View。Context Manifest 保存解析結果與排除原因，不應把完整來源內容複製進 Current State。

### 5.1 現行 Schema 相容方式

目前 [`handoff-envelope.schema.json`](../schemas/handoff-envelope.schema.json) 的 `requiredInputRefs` 已支援 `uri`、`sha256` 與 `description`，而且使用 `additionalProperties: false`。尚未新增 Context Manifest Schema 前，Handoff 必須遵守現行欄位：

- URI 指向精確路徑，不使用會隨時間漂移的 `latest` Alias。
- 可取得內容雜湊時填入 `sha256`。
- `description` 說明 Fact Type、用途及 Candidate／Accepted 身分。
- Change ID 與 Run ID 由被引用的 Artifact 本身驗證。
- Handoff 可以使用 `expiresAt` 限制交接有效期。

不得把 `authority`、`version`、`validUntil` 等尚未定義的欄位直接塞入 Handoff，否則會違反現行 Schema。需要完整描述時，可先把 Conceptual Context Manifest 當成獨立 Artifact，再由 `requiredInputRefs` 引用。

## 6. Lifecycle Stage Context Matrix

每個 Stage 只載入完成該工作所需的 Context。表中的「必要來源」缺少任一項時，Agent 應輸出 `incomplete`，由 Orchestrator 依第 11 節路由。

| Stage | 必要來源 | 預設不載入 |
|---|---|---|
| `plan-change` | Requirement、現行 Main Spec、相關 ADR／Schema、Current State、已知限制 | 完整舊對話、所有 Source Files、歷史 Trace |
| `review-plan` | Candidate Proposal／Design／Tasks、Requirement、Acceptance Criteria、相關 ADR／Schema | Implementation Diff、無關 Source、舊 Review 全文 |
| `apply-change` | Approved Scope／Design／Tasks、Current State、Handoff、必要 Source Files、適用 Workflow Policy | 未批准 Proposal、其他 Change 的 Artifact、完整歷史 |
| `review-result` | 固定版本的 Candidate Implementation Result、同版本 Diff、同 Run Validation、Approved Scope／Design／Acceptance Criteria | 其他 Candidate、舊 Validation、Implementer 未引用的聊天說明 |
| `fix-from-review` | 指定 Candidate、未解 Findings、Approved Scope、必要 Source、Focused Validation 要求 | 已關閉且無關的 Findings、其他 Run 的 Diff |
| `readiness-check` | 通過 Review 的 Artifact、對應 Validation、Review Result、Task Completion、Current State Gates | 未通過驗證的新 Candidate、Raw Trace、無關歷史版本 |
| `archive-change` | Readiness Result、Accepted Artifact References、最終 Diff／Validation Summary、Scope Deviation、Human Gate 狀態 | 失敗嘗試的完整內容、未處理 Candidate、Debug Log |

### 6.1 Candidate 的例外讀取

一般下游 Stage 預設使用 Accepted Artifact。以下工作可以讀取 Candidate：

- `review-plan` 審查 Candidate Plan。
- `review-result` 審查已完成必要 Validation 的 Candidate Implementation。
- `fix-from-review` 修正 Handoff 明確指定的 Candidate。
- Coordinator 執行 Conflict Arbitration 或失敗診斷。

Handoff 必須固定 Candidate 的 URI 與 Checksum。Agent 不得自行改讀同目錄中時間較新的檔案。

## 7. Context Selection Pipeline

Orchestrator 或 Context Builder 應按固定順序處理 Context：

1. 讀取 Current State，驗證 `changeId`、`runId`、`phase`、`currentOwner` 與 `revision`。
2. 依 Stage Policy 列出必要 Fact Type、可選 Fact Type、輸出契約與 Context Budget。
3. 為每個 Fact Type 解析權威來源；有多個候選時套用 Accepted／Candidate 規則。
4. 固定 URI、Version Identity、Checksum、Change ID、Run ID 與 State Revision。
5. 檢查 Role Permission、Path Scope、Sensitivity 與 Handoff Expiration。
6. 檢查 Freshness、Integrity、Validation Target 與 Cross-run Contamination。
7. 去除重複來源，排除低 Authority、過期、無關或禁止載入的來源。
8. 套用 Context Budget；必要來源超出預算時停止，不任意截斷契約或 Evidence。
9. 記錄 Required、Optional 與 Excluded Sources，以及每個排除原因。
10. 建立 Handoff，將精確 Reference 交給指定 Agent。

可以用下列偽程式碼實作：

```text
state = loadCurrentState(changeId)
assert state.revision == expectedRevision
assert state.currentOwner == targetRole

plan = resolveStagePolicy(state.phase, targetRole)
sources = resolveSources(plan.requiredFactTypes)
sources = pinVersionsAndChecksums(sources)
sources = filterByAuthorityFreshnessAndPermission(sources)

if missingRequiredSource(sources):
  return INCOMPLETE

manifest = buildContextManifest(state, plan, sources)
handoff = buildHandoff(state, manifest)
```

Agent 開始工作前應再次檢查 Handoff 是否過期、Current State Revision 是否仍相同，以及必要 Artifact 是否通過 Checksum 驗證。任一條件不符時，不執行原工作命令。

## 8. Version、Freshness 與 Integrity

「版本」依來源類型採不同識別方式：

| 來源 | Version Identity | Freshness 判斷 | Integrity 判斷 |
|---|---|---|---|
| OpenSpec／Git File | Commit、Blob Hash、Change Revision、固定路徑 | 是否仍是已批准版本 | Git Object Identity／Checksum |
| Current State | `changeId`、`runId`、`revision` | 是否仍是目前 Revision | Schema Validation／Atomic Read |
| Agent Result | `artifactId`、`runId`、固定 URI | 是否仍是 Handoff 指定 Candidate／Accepted 版本 | `sha256`／Schema Validation |
| Diff／Validation Evidence | Target Artifact、Diff Hash、Command Scope | 是否針對本輪 Candidate 產生 | Checksum／Exit Code／Target Binding |
| External Tool Result | Source ID、Observation ID | `observedAt`、`validUntil`、來源 SLA | Tool Signature／Response Hash／Trace ID |

### 8.1 Version Pinning

Handoff 建立後，必要來源必須固定。執行期間出現新版本時：

- 原 Handoff 繼續使用已固定版本，前提是它仍有效且 Current State Revision 未變。
- 新版本改變 Scope、Permission、Acceptance Criteria 或 Current State 時，Orchestrator 應使舊 Handoff 失效並重新建立。
- Agent 不得把相同目錄中修改時間較新的檔案視為自動替代品。

### 8.2 Freshness

Freshness Policy 應依來源類型設定，避免對所有 Context 使用同一個 TTL：

- Approved OpenSpec 以批准版本與變更狀態判斷，不用短 TTL。
- Current State 必須符合預期 Revision。
- Validation 必須綁定受審 Candidate，不能沿用上一輪結果。
- External Live State 應保存觀測時間與有效期。
- Chat Summary 與 Memory 預設為 Advisory，無法確認來源時不得升級 Authority。

### 8.3 Integrity 與 Binding

Checksum 一致只能證明內容未變，不能證明內容適合目前 Stage。Orchestrator 還必須驗證：

- Artifact 的 `changeId` 與目前 Change 相同。
- Artifact 的 `runId` 符合 Handoff 或明確允許的前一 Run。
- Validation 指向同一份 Candidate／Diff。
- Review Result 審查的 Artifact 與 Readiness 使用的 Artifact 相同。
- Current State Pointer 由 Coordinator 或 Orchestrator 更新，不由 Producer 自行接受。

## 9. Minimal Context 與 Progressive Disclosure

初始 Context 只包含 Required Sources。Optional Sources 在必要證據不足時才展開；Excluded Sources 保留 Reference 與排除原因，不載入內容。

### 9.1 Context Budget

本機版本可以先設定容易檢查的預算：

- `maxRequiredRefs`
- `maxOptionalRefs`
- `maxTotalBytes`
- `maxFilesOpened`
- `maxDisclosureLevel`
- `reservedOutputTokens`

預算不得導致 Requirement、Acceptance Criteria、必要 Evidence 或否定條件被截斷。必要來源超出預算時，Agent 應停止並要求 Coordinator 縮小 Scope、拆分 Stage 或批准 Context Expansion。

### 9.2 Expansion 規則

Context Expansion 必須記錄：

- 缺少哪一類 Fact 或 Evidence。
- 已讀取哪些來源，仍不足以完成哪個判斷。
- 希望增加的 Reference 或 Fact Type。
- 預估增加的 File／Byte／Token 成本。
- 最大 Disclosure Level。

Agent 可以提出 Expansion Request，無權自行擴大 Path Scope、Tool Permission 或 Sensitive Data Access。

### 9.3 預設排除項目

除非 Stage Policy 明確要求，初始 Context 應排除：

- 完整對話歷史。
- 其他 Change／Run 的 Artifact。
- 全量 Test stdout 與 Tool Trace。
- 未通過 Validation 的 Candidate。
- 可由權威來源重新取得的舊 Memory。
- Role 無權存取的路徑與敏感欄位。

## 10. Conceptual Context Manifest

下列 JSON 是規範閱讀範例，尚未對應本庫的正式 JSON Schema。它不能直接當成 `handoff-envelope.schema.json` 的內嵌物件；採用時應保存為獨立 Artifact，再由 Handoff 引用。

```json
{
  "schemaVersion": "0.1.0-draft",
  "changeId": "feature-example",
  "runId": "run-20260804-001",
  "stage": "readiness-check",
  "consumerRole": "readiness",
  "stateRevision": 7,
  "generatedAt": "2026-08-04T10:30:00Z",
  "requiredSources": [
    {
      "refId": "approved-scope",
      "factType": "requirement_scope",
      "authority": "authoritative",
      "uri": "repo://openspec/changes/feature-example/proposal.md",
      "version": "git:abc1234",
      "sha256": "1111111111111111111111111111111111111111111111111111111111111111",
      "purpose": "Verify accepted implementation against approved scope"
    },
    {
      "refId": "accepted-implementation-v1",
      "factType": "implementation",
      "authority": "accepted",
      "uri": "artifact://feature-example/run-20260804-001/implementation-result-v1.json",
      "version": "1",
      "sha256": "2222222222222222222222222222222222222222222222222222222222222222",
      "purpose": "Readiness target"
    },
    {
      "refId": "validation-v1",
      "factType": "validation",
      "authority": "accepted",
      "uri": "evidence://feature-example/run-20260804-001/validation-summary-v1.json",
      "version": "1",
      "sha256": "3333333333333333333333333333333333333333333333333333333333333333",
      "boundTo": "accepted-implementation-v1",
      "purpose": "Verify validation coverage and result"
    }
  ],
  "optionalSources": [
    {
      "refId": "focused-test-log-v1",
      "factType": "diagnostic_evidence",
      "authority": "diagnostic",
      "uri": "evidence://feature-example/run-20260804-001/focused-test-log-v1.txt",
      "loadWhen": "validation summary lacks failure details"
    }
  ],
  "excludedSources": [
    {
      "refId": "implementation-v2",
      "uri": "artifact://feature-example/run-20260804-001/implementation-result-v2.json",
      "reason": "validation_failed",
      "authority": "candidate"
    }
  ],
  "budget": {
    "maxFilesOpened": 12,
    "maxTotalBytes": 200000,
    "maxDisclosureLevel": 1
  },
  "fallbackPolicy": {
    "missingRequiredSource": "INCOMPLETE",
    "authorityConflict": "NEEDS_COORDINATOR_ARBITRATION"
  }
}
```

正式 Schema 若在後續版本導入，至少應驗證 Required／Optional／Excluded Sources、Authority、Version Identity、Freshness、Checksum、Binding、Budget 與 Fallback Policy。導入前仍以現有 Handoff、Agent Result 與 Current State Schema 為準。

## 11. 固定失敗代碼與路由

Agent 不應以自由文字決定 Context 錯誤的處理方式。Result 可以在 `summary`、`risks`、`payload` 或既有 Blocker 結構中保存下列代碼；Orchestrator 依固定規則更新 State。

| Code | 觸發條件 | Agent Status | 建議路由 |
|---|---|---|---|
| `CONTEXT_SOURCE_MISSING` | 必要 Fact Type 沒有可用來源 | `incomplete` | 補齊來源或退回 Coordinator |
| `CONTEXT_SOURCE_STALE` | Current State、Validation 或 Live State 已過期 | `incomplete` | 重新解析來源並建立 Handoff |
| `CONTEXT_VERSION_MISMATCH` | 實際內容與固定 Version／Checksum 不符 | `incomplete` | 使原 Handoff 失效 |
| `CONTEXT_RUN_MISMATCH` | Artifact 屬於其他 Change／Run，且未被明確允許 | `incomplete` | 隔離 Artifact，重新解析 |
| `CONTEXT_INTEGRITY_FAILED` | Checksum、Schema 或 Target Binding 驗證失敗 | `failed` | 停止流程並保留 Evidence |
| `CONTEXT_AUTHORITY_CONFLICT` | 同一 Fact Type 有互斥的權威來源 | `blocked` | `NEEDS_COORDINATOR_ARBITRATION` |
| `CONTEXT_SCOPE_DENIED` | Role 無權讀取必要來源 | `incomplete` | 縮小工作或請 Human 審核權限 |
| `CONTEXT_BUDGET_EXCEEDED` | 必要 Context 超出預算且無法安全壓縮 | `incomplete` | 拆分 Scope 或批准 Expansion |

遇到 `CONTEXT_SCOPE_DENIED` 時，Agent 不得嘗試其他路徑、不同 Tool 或替代帳號規避限制。需要增加權限時，由 Human 或受控 Permission Workflow 處理。

## 12. 本機範例：Latest 與 Accepted 分離

假設 Implementer 先產生 v1，Validation 與 Review 均通過；隨後又產生 v2，但 v2 的 Validation 失敗：

```text
implementation v1
  → validation passed
  → review approved
  → accepted

implementation v2
  → validation failed
  → remains candidate
```

目前 `current-state.schema.json` 使用 `latestArtifacts`。在 Schema 尚未新增 `acceptedArtifacts` 前，Orchestrator 必須把這些 Pointer 解讀為「目前最新可用且已接受的 Artifact」，不能依建立時間自動指向 v2。

Readiness Handoff 應固定 v1：

```json
{
  "from": "reviewer",
  "to": "readiness",
  "stage": "readiness-check",
  "requiredInputRefs": [
    {
      "refId": "accepted-implementation-v1",
      "type": "agent-result",
      "uri": "artifact://feature-example/run-20260804-001/implementation-result-v1.json",
      "sha256": "2222222222222222222222222222222222222222222222222222222222222222",
      "description": "Accepted implementation for readiness; candidate v2 failed validation"
    }
  ]
}
```

上例是閱讀片段，省略了 Handoff Schema 的其他必要欄位，不能直接拿去驗證。

若工作目標改為調查 v2 失敗原因，Coordinator 應建立新的 Diagnostic Handoff，明確引用 v2 與其 Validation Evidence。這次讀取不會改變 v1 的 Accepted 身分，也不會讓 v2 進入 Readiness。

## 13. 最小本機導入方式

不新增 Schema 也能先採用本文件的大部分規則：

1. 在 Stage Policy 或 Coordinator Prompt 中列出必要 Fact Type。
2. Handoff 使用精確 URI，不使用 `latest` 檔名或模糊目錄引用。
3. 可計算內容雜湊時，在 Artifact Reference 填入 `sha256`。
4. `description` 標記來源用途、Candidate／Accepted 身分與固定版本。
5. Agent 啟動前檢查 Current State Revision、Handoff Expiration 與 Artifact Run ID。
6. Agent 結果缺少必要 Context 時輸出 `incomplete`，並附上固定失敗代碼。
7. Orchestrator 記錄實際載入與排除的 Reference；第一版可保存在 Trace。
8. Coordinator 或 Orchestrator 管理 Accepted Pointer，Producer 只產生 Candidate。

當 Context 選擇需要跨 Agent Adapter、跨機器執行或進入嚴格稽核時，再考慮加入：

```text
schemas/context-manifest.schema.json
templates/context-manifest.template.json
examples/context-authority-conflict/
```

這些資產屬於後續契約升級，不是採用本文件的前置條件。

## 14. 驗收 Checklist

### Authority

- [ ] 每個必要 Fact Type 都有明確的權威來源。
- [ ] Advisory、Diagnostic 與 Prohibited Context 不會覆蓋權威來源。
- [ ] Authority Conflict 會進入 Coordinator／Human 仲裁。

### Version 與 Freshness

- [ ] Handoff 使用精確 URI，並在可行時保存 Checksum。
- [ ] Current State Revision、Change ID 與 Run ID 在執行前完成驗證。
- [ ] Validation 與 Review 綁定同一份 Candidate／Diff。
- [ ] External Live State 有 Observation Time 與 Validity Policy。

### Context Assembly

- [ ] 每個 Stage 只載入必要來源。
- [ ] Optional Context 依 Expansion Policy 載入。
- [ ] 排除來源有可檢查的原因。
- [ ] Context Budget 不會截斷 Requirement、否定條件或必要 Evidence。

### Agent 行為

- [ ] Agent 不自行改讀較新的 Artifact。
- [ ] Agent 不從 Chat 或 Memory 猜測缺少的 Requirement、Permission 或 Gate。
- [ ] 缺少必要 Context 時輸出固定錯誤代碼與 `incomplete`／`blocked`。
- [ ] Producer 不能自行接受 Artifact 或更新 Accepted Pointer。

### Schema 相容性

- [ ] 現行 Handoff 只使用 `handoff-envelope.schema.json` 已定義的欄位。
- [ ] Conceptual Context Manifest 保存在獨立 Artifact，不內嵌到現行 Handoff。
- [ ] 本文件沒有改變 Current State、Agent Result 或 Handoff Schema 的語意。

## 15. 參考資料

- [Artifact-based Shared State + Structured Handoff 參考架構](./02-reference-architecture-zh-TW.md)
- [Repo、Runtime State、Evidence 與 Trace 的分離](./03-runtime-storage-and-retention-zh-TW.md)
- [安全、治理與長期維護性](./05-security-and-maintainability-zh-TW.md)
- [Context Engineering Core](../../../context-engineering/docs/01-context-engineering-core-zh-TW.md)
- [多智能體系統協作：一份真正可落地的實踐指南](https://www.puppyone.ai/zh/blog/collaboration-in-multi-agent-systems-practical-guide)
- [Harness + Mut：AI Agent Delivery 與 Context Control 的推薦分層](https://www.puppyone.ai/zh/blog/harness-mut-recommended-split-for-ai-agent-delivery-and-context-control)
