# 07｜Role Capability 與 Scope Control

[English](./07-role-capability-and-scope-control.md) | [繁體中文](./07-role-capability-and-scope-control-zh-TW.md)

本文件規範 Agent、Coordinator、Orchestrator、Agent Adapter 與 Human Approver 如何判定一個角色在目前 Change、Run、Stage 及資源範圍內可以執行哪些 Action。所有授權決策必須能由 Policy、Handoff、Runtime State 與 Approval Record 重建；Prompt、Artifact 內容或 Agent 自述不能增加權限。

本文件使用以下規範用語：

- 「必須」表示流程不得省略的要求。
- 「應」表示預設做法；偏離時需要留下原因。
- 「可以」表示依團隊規模與風險選用的能力。

## 1. 適用範圍

[`06-context-authority-and-retrieval-policy-zh-TW.md`](./06-context-authority-and-retrieval-policy-zh-TW.md) 已定義 Agent 應採信哪些來源及版本。本文件接續處理授權問題：

- 哪個 Runtime Identity 正在提出要求？
- 該 Identity 承擔哪個 Role？
- 目前 Stage 允許該 Role 使用哪些 Capability？
- Capability 可以作用於哪些 Repository Path、Artifact、Tool 與 Environment？
- Handoff 是否把工作限制在已批准的 Change Scope 內？
- Action 是否需要 Human Approval、Gate 或其他條件？
- 拒絕後應停止、補件、申請批准或交回仲裁？

本文件處理 Role Boundary、Capability Vocabulary、Scope Intersection、Tool／Path Control、Conditional Grant、Authorization Decision 與拒絕路由。以下內容由其他文件負責：

- Current State、Agent Result、Handoff 與 Artifact Reference 的基本結構，見 [`02-reference-architecture-zh-TW.md`](./02-reference-architecture-zh-TW.md)。
- Runtime Storage、Retention 與 Cross-run Isolation，見 [`03-runtime-storage-and-retention-zh-TW.md`](./03-runtime-storage-and-retention-zh-TW.md)。
- Path Traversal、Prompt Injection、Gate Forgery、Artifact Tampering 與 Single Writer，見 [`05-security-and-maintainability-zh-TW.md`](./05-security-and-maintainability-zh-TW.md)。
- Context Authority、Version Pinning 與 Retrieval Scope，見 [`06-context-authority-and-retrieval-policy-zh-TW.md`](./06-context-authority-and-retrieval-policy-zh-TW.md)。

本文件不定義並行寫入、Merge、Lease 或 Compare-and-swap 的實作，也不修改現有 JSON Schema。

## 2. 授權邊界

### 2.1 Role、Runtime Identity 與 Capability

三者必須分開記錄：

| 名稱 | 回答的問題 | 範例 |
|---|---|---|
| Role | 此次執行承擔什麼責任？ | `reviewer` |
| Runtime Identity | 實際由哪個 Agent、Process 或 Human 提出要求？ | `agent:reviewer-local-01` |
| Capability | 該 Identity 可以對哪類資源執行什麼 Action？ | `artifact.write.own` |

同一個 Agent CLI 可以在不同 Run 承擔不同 Role，但每次 Invocation 必須固定一個 Runtime Identity、Role、Change ID、Run ID 與 Stage。Orchestrator 不得只憑模型名稱、Prompt 文字或目錄名稱推斷 Identity。

Role 描述責任，Capability 才是執行權限。Prompt 中出現「可以修改」、「請執行」或「已獲批准」等文字，不構成授權來源。

### 2.2 Policy Plane 與 Execution Plane

授權責任分為兩層：

```text
Workflow Policy / Role Policy / Approval Record
                    ↓
          Authorization Decision
                    ↓
Agent Adapter / Sandbox / Tool Host / Filesystem
```

Policy Plane 決定是否允許，Execution Plane 負責強制執行。Agent Adapter 必須把決策轉成實際限制，例如唯讀掛載、允許的 Working Directory、Tool Allowlist、Command Timeout 與 Network Boundary。只在 Prompt 裡寫「不得修改」不足以形成安全邊界。

### 2.3 最小權限與預設拒絕

每次 Invocation 只取得完成該 Stage 所需的最小 Capability。系統遇到以下情況時必須拒絕：

- Capability 未知或未註冊。
- Role Policy 沒有授予該 Capability。
- Resource、Path、Tool Argument 或 Environment 超出 Scope。
- Policy、State Revision、Handoff 或 Approval 已過期。
- 必要 Gate 尚未通過。
- 無法確認要求是否具有副作用。

缺少規則不能解讀為允許。新 Tool、新 Action 或新資源類型上線前，必須先加入受版本控管的 Policy。

## 3. Capability Vocabulary

Capability 使用穩定名稱描述「資源類型 + Action」。名稱不綁定特定 CLI 或模型供應商。

### 3.1 建議的 Capability

| Capability | 說明 |
|---|---|
| `spec.read` | 讀取已批准的 Requirement、Proposal、Design、Tasks 或 ADR |
| `spec.write` | 在批准的 Change Scope 內建立或修改規格 |
| `source.read` | 讀取產品程式碼、測試與設定檔 |
| `source.write` | 修改被指派的產品程式碼、測試或設定檔 |
| `artifact.read` | 讀取 Handoff 指定的 Agent Result 或其他 Artifact |
| `artifact.write.own` | 建立本 Role、本 Stage 的輸出 Artifact |
| `evidence.read` | 讀取指定的 Diff、Validation 或 Command Evidence |
| `evidence.write.own` | 建立本次執行產生的 Evidence |
| `runtime.read` | 讀取 Current State 與有效 Handoff |
| `runtime.transition.propose` | 在 Agent Result 中提出下一個 Transition |
| `runtime.transition.apply` | 驗證 Gate 後更新 Current State 與 Handoff |
| `tool.execute.read` | 執行不應改變外部狀態的 Tool |
| `tool.execute.write` | 執行會改變檔案、服務或外部系統的 Tool |
| `git.commit` | 建立 Git Commit |
| `git.push` | 將 Commit 推送至遠端 |
| `deploy.execute` | 部署到指定 Environment |
| `resource.delete` | 刪除檔案、資料或外部資源 |
| `permission.change` | 修改 Role、Policy、Credential 或 Access Control |

Capability ID 應保持小而穩定；Path、Tool ID、Environment 與 Risk Level 放在 Scope 或 Condition，不為每個檔案產生一個新 Capability 名稱。

### 3.2 Action 必須可區分

以下 Action 不得合併成籠統的 `write`：

| Action | 意義 |
|---|---|
| `enumerate` | 列出容器、目錄或集合中的資源名稱 |
| `read` | 讀取一個已知資源 |
| `create` | 建立新資源，但不覆寫既有內容 |
| `update` | 修改既有資源 |
| `delete` | 刪除資源 |
| `execute` | 執行 Tool、Command、Migration 或 Deployment |
| `propose` | 提出狀態轉移或 Approval Request |
| `approve` | 對已固定的 Action 與 Scope 給出批准 |
| `apply` | 套用已驗證的 Transition 或變更 |

允許 `read` 不會自動允許 `enumerate`。允許 `create` 也不代表可以 `update` 或覆寫同名 Artifact。

## 4. Scope 維度

一筆 Grant 必須能限制實際作用範圍。至少評估下列維度：

| 維度 | 必要資訊 | 說明 |
|---|---|---|
| Actor | `agentId`、`role` | 綁定提出要求的 Identity 與 Role |
| Lifecycle | `changeId`、`runId`、`stage` | 防止跨 Change、跨 Run 或跨階段使用 |
| Resource | URI Scheme、Resource Class | 區分 Spec、Source、Runtime、Evidence 與外部系統 |
| Path | Canonical Root、Allowlist Pattern | 限制 Repository、Worktree 與 Runtime 路徑 |
| Action | `read`、`create`、`execute` 等 | 限制對資源的操作方式 |
| Tool | Tool ID、Command 類型、Argument Constraint | 限制執行介面與參數 |
| Environment | local、CI、staging、production | 防止本機權限被帶到高風險環境 |
| Risk | low、medium、high | 決定是否需要 Approval 或直接拒絕 |
| Time | `issuedAt`、`expiresAt` | 限制 Grant 與 Approval 的有效期間 |
| Policy | `policyId`、`policyVersion` | 讓決策可以重建與稽核 |

資源可以使用既有 URI Scheme 表達：

```text
repo://openspec/changes/feature-example/design.md
repo://src/payment/service.ts
runtime://feature-example/run-002/artifacts/review-result.json
evidence://feature-example/run-002/validation-summary.json
```

URI 只是識別方式。Adapter 仍必須解析實際路徑、正規化結果，並確認目標位於已批准 Root 內。Path Traversal、Symbolic Link 與 Junction 的檢查沿用 05 的安全規則。

## 5. Effective Scope

### 5.1 交集模型

實際可用的權限由多層限制取交集：

```text
Effective Scope =
  Platform Maximum
  ∩ Adapter Maximum
  ∩ Role Policy
  ∩ Stage Policy
  ∩ Handoff Scope
  ∩ Change / Run Boundary
  ∩ Environment Boundary
```

每一層都可以縮小 Scope。較低層不能放寬較高層的限制；Explicit Deny 優先於所有 Allow。

Human Approval 只滿足既有 Conditional Grant。若 Platform Maximum 或 Role Policy 已拒絕 `permission.change`，單次 Approval 不會把該 Capability 加入 Effective Scope。

### 5.2 判定順序

Orchestrator 或 Policy Engine 必須依固定順序判定：

1. 驗證 Runtime Identity、Role、Change ID、Run ID 與 Stage。
2. 載入指定版本的 Platform、Adapter、Role 與 Stage Policy。
3. 檢查 Handoff 是否有效，並固定其中的 Resource Reference。
4. 計算各層 Scope 的交集。
5. 正規化 Resource、Path、Tool ID 與 Arguments。
6. 先套用 Explicit Deny，再判斷 Allow。
7. 檢查 Risk、Gate、Approval、到期時間與 State Revision。
8. 產生 `allow`、`deny` 或 `pause` 決策，並寫入 Audit Trace。

任何步驟無法完成時，流程必須 fail closed。Agent 不得改用其他 Tool、改寫 Path 或縮短 Command 後自行重試，以此猜測可用權限。

### 5.3 State 與 Policy 綁定

Authorization Decision 應至少綁定：

```text
policyId + policyVersion
changeId + runId + stage
stateRevision
actorId + role
capability + resource
```

Current State Revision 改變後，先前的 Decision 應重新評估。這可以避免 Agent 在 Handoff 更新、Scope 收窄或 Gate 被撤回後沿用舊授權。

## 6. Role Capability Matrix

下表定義預設邊界。實際專案可以再縮小，不得省略 Human Gate 或擴大 Reviewer 的產品寫入權限。

| 角色 | 預設允許 | 固定限制 |
|---|---|---|
| Coordinator | 讀取 Spec、Source 與 Runtime；在批准的 Change 內維護 Plan／Spec；建立 `coordinator_result`；提出 Transition | 不修改產品程式碼；不直接更新 Current State；不執行 Git／Deploy／Delete |
| Implementer | 讀取批准的 Contract；修改指定 Worktree 與 Path；執行允許的 Validation；建立 `implementation_result` 與本輪 Evidence；提出 Transition | 不擴大 Scope；不批准自己的結果；不直接更新 Current State；不執行 Git／Deploy |
| Reviewer | 讀取指定 Spec、Source、Diff 與 Evidence；建立 `review_result`；提出 Transition | 不修改 Spec、Source、Implementation Result、Evidence 或 Current State；不執行破壞性 Command |
| Readiness | 讀取 Accepted Artifact、Review Result、Gate 與 Evidence；建立 `readiness_result`；提出 Transition | 不修改實作、Review Verdict、Evidence、Accepted Pointer 或 Current State |
| Archive | 讀取已通過 Readiness 的 Artifact；建立 `archive_result` 與受控 Execution Summary；提出 Transition | 不修改實作與既有結果；不跳過 Human Git Gate；不自行宣告 Commit 完成 |
| Orchestrator／Automation | 載入 Policy、組裝 Handoff、呼叫 Adapter、驗證 Result、套用合法 Transition、更新 Runtime State | 不修改產品程式碼與批准的 Spec；不代替 Reviewer 判定品質；不代替 Human 批准高風險 Action |
| Human | 仲裁 Scope、批准高風險 Action，依組織 Policy 執行受控 Git／Deploy／Delete | Approval 必須綁定固定 Action 與 Scope；口頭同意不更新 Gate |

`automation` 是 Runtime Identity 類型，可承擔 Orchestrator 的機械操作。它不能因為屬於自動化流程而取得更高權限。

## 7. Own Output Write

### 7.1 「Reviewer 唯讀」的精確定義

Reviewer 對產品程式碼、規格、Implementation Result、既有 Evidence 與 Current State 維持唯讀。Reviewer 仍需建立自己的 `review_result`，因此可以取得一筆窄化的 `artifact.write.own`：

```text
Capability: artifact.write.own
Action: create
Resource: runtime artifact
Path: .agent-runtime/<change-id>/<run-id>/artifacts/review-result.json
Owner: reviewer
Stage: review-result
Overwrite: deny
```

若同一路徑已存在，Adapter 應拒絕覆寫，或由 Orchestrator 分配新的 Artifact ID／Attempt Path。Reviewer 不得修改 Implementation Result 來「順手修正」受審內容。

### 7.2 其他 Stage Output

相同規則適用於各角色的輸出：

| Role | 可建立的 Own Output | 不得改寫的上游內容 |
|---|---|---|
| Coordinator | `coordinator-result.json` | 已批准 Requirement、Current State |
| Implementer | `implementation-result.json`、本輪 Evidence | Review Result、Accepted Pointer |
| Reviewer | `review-result.json` | Source、Spec、Implementation Result、Evidence |
| Readiness | `readiness-result.json` | Review Result、Gate、Accepted Artifact |
| Archive | `archive-result.json`、受控 Execution Summary | Implementation、Review、Readiness Result |

Own Output 必須與 `producer`、`kind`、`stage`、`changeId` 及 `runId` 相符。Orchestrator 在接受 Result 前必須驗證這些欄位。

## 8. Path 與資料可見性

### 8.1 Path Scope

Adapter 必須先將要求的路徑解析為 Canonical Path，再與 Allowlist 比對。下列寫法不得直接轉成 Filesystem 權限：

```text
repo://src/**
runtime://feature-example/**
```

Wildcard 可以作為 Policy Pattern，但實際執行仍要落到已解析的 Root 與 Path。`..`、替代資料流、Symbolic Link、Junction 或不同大小寫造成的 Root Escape 必須拒絕。

同一 Role 需要不同寫入區時，應分開授權，例如：

```text
source.write       -> assigned worktree paths
artifact.write.own -> fixed result path
evidence.write.own -> fixed evidence directory
```

三個 Scope 不能合併成整個 Repository 的 Write Access。

### 8.2 Enumerate 與 Read

目錄或集合名稱也可能揭露敏感資訊。Policy 必須分別處理：

- 已知 URI 的 `read`。
- 指定 Root 下的 `enumerate`。
- 依 Pattern 搜尋內容的 `search`。

Reviewer 可以讀取 Handoff 指向的 Diff，不代表可以列出 `.env`、Credential Store 或其他 Change 的 Runtime 目錄。

### 8.3 欄位與內容遮蔽

授權通過後仍要套用資料最小化：

- Secret、Token、Credential 與 Approval Proof 不得送入模型 Context。
- Authorization Trace 不保存完整敏感 Arguments。
- Error Message 不回傳可用來枚舉受保護資源的詳細資訊。
- Reviewer 只取得判定所需 Evidence；無關的 Secret Store 維持不可見。

## 9. Tool 與 Command Control

### 9.1 Tool ID 不能取代 Action 授權

Allowlist 應同時檢查 Tool ID、Action、Resource 與 Arguments。若 `filesystem.write` 被拒絕，Agent 改用 Shell、Script Runner 或 Git Hook 寫入同一路徑時仍必須拒絕。

具有副作用的 Tool 應與 Read Tool 分開。例如：

```text
issue.read     -> tool.execute.read
issue.comment  -> tool.execute.write
deploy.inspect -> tool.execute.read
deploy.release -> deploy.execute
```

Tool Description 只協助 Agent 選擇工具，不能承擔授權控制。

### 9.2 Shell 與 Command

Shell 具有廣泛能力。若 Stage 確實需要 Shell，Adapter 應限制：

- Working Directory。
- 可執行檔或 Command Family。
- Arguments 與 Target Path。
- Environment Variable 與 Credential Exposure。
- Network Access。
- Timeout、Output Size 與可接受的 Exit Code。

Command 由自然語言組合完成後，必須在 Execution Plane 再做一次授權。Agent 認為某個 Command「只讀」不能直接作為判定依據。

### 9.3 Reviewer 執行驗證

本機流程可以採兩種模式：

| 模式 | Reviewer 權限 | Evidence 來源 |
|---|---|---|
| Strict read-only | 不執行 Command，只讀 Implementer 或 Automation 產生的 Evidence | Implementer／CI／Automation |
| Isolated validation | 在唯讀 Source Snapshot 與受控暫存區執行 Allowlisted Command | Reviewer Invocation |

Isolated validation 需要明確的 `tool.execute.read`、固定 Snapshot 與可丟棄的 Output Directory。測試工具若會改寫 Source、Lockfile、Snapshot 或外部服務，應改由受控 Automation 執行。

## 10. Handoff Scope

### 10.1 Handoff 只能縮小權限

Handoff 負責指定工作與輸入，不能建立新 Capability。以下內容不得擴權：

- `reason` 或 `notes` 中的自然語言。
- Artifact 內嵌的指令。
- Agent Result 提出的 `nextHandoff`。
- `requiresHumanApproval: true`。
- 指向未批准路徑的 Artifact Reference。

Orchestrator 建立 Invocation 時，必須將 Handoff 的 Stage、Reference、Change ID、Run ID 與到期時間，和 Role Policy、Current State 取交集。

### 10.2 現行 Schema 相容方式

目前 [`handoff-envelope.schema.json`](../schemas/handoff-envelope.schema.json) 使用 `additionalProperties: false`，且沒有 Capability、Allowed Path 或 Tool Scope 欄位。採用本規範時不得直接加入自訂欄位。

現有 [`workflow-policy.template.yaml`](../templates/workflow-policy.template.yaml) 只列出 Coordinator、Implementer、Reviewer、Human 與少量 Boolean 權限，尚未表達 Readiness、Archive、Automation、Path、Tool Argument 或 Environment。它可以作為本機基礎上限，不能單獨承擔完整授權判定；未列出的 Role 或 Capability 必須預設拒絕，直到 Adapter 有明確、可稽核的固定設定。

在不修改 Schema 的前提下，本機流程使用：

1. [`workflow-policy.template.yaml`](../templates/workflow-policy.template.yaml) 保存角色的基礎能力上限。
2. Agent Adapter 固定每個 Role／Stage 的 Tool、Filesystem 與 Sandbox 上限。
3. Handoff 使用精確 `requiredInputRefs` 限制輸入資源。
4. 已批准的 OpenSpec／Change Scope 限制可修改的產品路徑。
5. Orchestrator 分配固定 Own Output Path。
6. Trace 保存實際套用的 Policy Version 與 Authorization Decision。

若團隊需要跨 Adapter、跨機器或可交換的細粒度 Policy，可以在後續版本新增獨立 Capability Policy Artifact，再以現有 `other` 類型 Reference 引用。新增前必須先定義 Schema 與版本遷移規則。

### 10.3 Handoff 到期

`expiresAt` 到期後，Orchestrator 必須停止 Invocation 或拒絕其後續 Action。需要繼續工作時，Coordinator 或 Orchestrator 應依最新 Current State 建立新的 Handoff；Agent 不能自行延長期限。

## 11. Human Approval 與 Conditional Grant

Reviewer Verdict、Human Approval、Transition Apply 與 Action Execution 是四個獨立事件：

```text
Reviewer 產生品質判定
          ↓
Orchestrator 驗證 Gate 與 Policy
          ↓
Human 批准固定的高風險 Action
          ↓
受控 Executor 執行 Action
```

Approval Record 應綁定：

- Approver Identity。
- Requested Capability 與 Action。
- Resource、Path 或 Environment。
- Change ID、Run ID、Stage 與 State Revision。
- Policy ID 與 Policy Version。
- Evidence Reference 與預期副作用。
- Issued At、Expires At 與是否可重複使用。

Approval 不應是一個可由 Agent Result 寫入的 Boolean。現有 Current State 中的 `humanApproved` 只能由 Orchestrator 或受控 Approval Service 根據有效 Approval Record 更新。

Human 拒絕後，Executor 不得改變參數重試。Scope 需要調整時，流程回到新的 Approval Request。

## 12. 概念性 Capability Policy

以下 YAML 用來說明授權模型。它不符合目前的 `workflow-policy.template.yaml`，也不能嵌入現行 Handoff：

```yaml
schemaVersion: 0.1.0-draft
policyId: local-reviewer-policy
policyVersion: 0.1.0
role: reviewer

grants:
  - capability: spec.read
    actions: [read]
    resources:
      - repo://openspec/changes/${changeId}/**

  - capability: source.read
    actions: [read]
    resources:
      - repo://src/**
      - repo://tests/**

  - capability: runtime.read
    actions: [read]
    resources:
      - runtime://${changeId}/current-state.json
      - runtime://${changeId}/${runId}/handoff-envelope.json

  - capability: artifact.read
    actions: [read]
    resources:
      - runtime://${changeId}/${runId}/artifacts/implementation-result.json

  - capability: evidence.read
    actions: [read]
    resources:
      - evidence://${changeId}/${runId}/**

  - capability: artifact.write.own
    actions: [create]
    resources:
      - runtime://${changeId}/${runId}/artifacts/review-result.json

denies:
  - capability: source.write
  - capability: spec.write
  - capability: runtime.transition.apply
  - capability: git.commit
  - capability: git.push
  - capability: deploy.execute

conditions:
  stages: [review-result]
  maxRiskLevel: low
  requireCurrentStateRevision: true
  expiresAt: "2026-08-10T12:00:00Z"
```

正式導入此格式前，至少需要：

```text
schemas/capability-policy.schema.json
schemas/authorization-decision.schema.json
templates/role-capability-policy.template.yaml
examples/reviewer-source-readonly-output-write/
```

這些資產屬於後續契約升級，不是本文件的前置條件。

## 13. Authorization Decision

### 13.1 最小結果

每次受控 Action 應留下可稽核的 Decision。概念性結果如下：

```json
{
  "decision": "deny",
  "code": "RESOURCE_OUT_OF_SCOPE",
  "actorId": "agent:reviewer-local-01",
  "role": "reviewer",
  "changeId": "feature-example",
  "runId": "run-review-002",
  "stage": "review-result",
  "capability": "source.write",
  "resourceClass": "repository-path",
  "policyId": "local-reviewer-policy",
  "policyVersion": "0.1.0",
  "safeReason": "Requested path is outside the reviewer write scope.",
  "nextAllowedAction": "produce-review-result",
  "traceId": "authz-review-002-004"
}
```

此結果是概念格式，不屬於現有 Agent Result Schema。實作初期可以將等價欄位保存於 Trace。

### 13.2 固定拒絕代碼

| Code | 條件 | 預設路由 |
|---|---|---|
| `CAPABILITY_UNKNOWN` | Policy 不認得要求的 Capability | `deny`，交回 Orchestrator 設定檢查 |
| `CAPABILITY_NOT_GRANTED` | Role／Stage 沒有取得該 Capability | `deny`，不得自行重試 |
| `RESOURCE_OUT_OF_SCOPE` | Resource 或 Path 超出批准範圍 | `deny`，交回 Coordinator 評估 Scope |
| `PATH_ENUMERATION_DENIED` | 允許讀已知資源，但不允許列出容器 | `deny`，改用精確 Reference |
| `TOOL_NOT_ALLOWED` | Tool 不在 Adapter Allowlist | `deny`，選用已批准 Tool 或補件 |
| `ARGUMENT_SCOPE_VIOLATION` | Tool Arguments 指向未批准目標 | `deny`，不能以參數變形探測權限 |
| `RISK_LEVEL_EXCEEDED` | Action 風險高於 Policy 上限 | `pause` 或 `deny` |
| `APPROVAL_REQUIRED` | Conditional Grant 尚未取得批准 | `pause`，建立 Approval Request |
| `APPROVAL_INVALID_OR_EXPIRED` | Approval 不匹配、失效或已使用 | `pause`，要求新 Approval |
| `SCOPE_EXPIRED` | Handoff 或 Invocation Scope 到期 | `incomplete`，重新建立 Handoff |
| `POLICY_VERSION_MISMATCH` | Decision 所用版本與目前 Policy 不一致 | `blocked`，重新評估 |
| `STATE_GATE_NOT_SATISFIED` | Current State 尚未通過必要 Gate | `blocked`，返回合法前置 Stage |

`safeReason` 只說明資源類別與可採取的下一步，不應揭露 Secret、Credential、Approval Token 或受保護路徑是否存在。

## 14. Audit Trace

Allow、Deny 與 Pause 都應留下 Trace。最小內容包括：

- Timestamp 與 Trace ID。
- Actor ID、Role、Change ID、Run ID 與 Stage。
- Policy ID、Policy Version 與 State Revision。
- Requested Capability、Action 與安全化的 Resource Reference。
- Decision、Reason Code 與使用的 Gate／Approval Reference。
- 實際執行的 Adapter、Tool ID 與 Exit Status。

Trace 不應保存：

- Secret、Token、Credential 或 Approval Proof 原文。
- 完整敏感 Tool Arguments。
- Agent Chain-of-thought。
- 與本次授權判定無關的 Repository 內容。

同一筆 Action 的 Authorization Decision 與 Tool Execution 應共用 Trace ID，才能區分「已允許但執行失敗」與「尚未取得權限」。

## 15. 本機 Lifecycle 範例

以下範例沿用 Single Writer 與 `.agent-runtime`：

```text
Change: feature-example
Run: run-002
Assigned source scope:
  src/payment/**
  tests/payment/**
```

### 15.1 Coordinator：`plan-change`

輸入是已批准的 Requirement 與現有 Source。Coordinator 可以更新該 Change 的 Proposal、Design 與 Tasks，並建立 `coordinator-result.json`；產品程式碼與 Current State 維持不可寫。

### 15.2 Implementer：`apply-change`

Orchestrator 把 Implementer 限制在指定 Worktree、`src/payment/**`、`tests/payment/**` 與本 Run 的 Output Directory。Implementer 可以修改程式碼、執行 Allowlisted Validation、建立 Implementation Result 與 Evidence。若嘗試修改 `src/identity/**`，Adapter 回傳 `RESOURCE_OUT_OF_SCOPE`。

### 15.3 Reviewer：`review-result`

Reviewer 讀取固定版本的 Spec、Diff、Implementation Result 與 Validation Evidence。Source Snapshot 維持唯讀，唯一可建立的檔案是本 Run 的 `review-result.json`。若 Reviewer 嘗試修正程式碼，授權層拒絕 `source.write`；Reviewer 應把問題寫成 Finding，交由 Implementer 處理。

### 15.4 Readiness：`readiness-check`

Readiness 只讀 Accepted Implementation、通過的 Review、必要 Evidence 與 Gate，建立 `readiness-result.json`。它可以判定 `incomplete`，但不能自行補跑具有副作用的 Command，也不能更新 `humanApproved`。

### 15.5 Archive 與 Human Gate

Archive 建立 `archive-result.json` 與 Execution Summary。若流程下一步需要 Commit，Orchestrator 產生固定 Commit Scope 與 Diff Reference 的 Approval Request。Human Approval 通過後，具有 `git.commit` 的受控 Executor 才能執行；Archive Agent 本身不取得 Git 權限。

### 15.6 Runtime State 更新

各角色只能在 Agent Result 中提出 Transition。Orchestrator 驗證 Result Schema、Capability、Gate、State Revision 與 Transition Table 後，才更新 `current-state.json`。Artifact Producer 不能把自己的輸出直接設為 Accepted。

## 16. 最小本機導入

現有 Schema 不變，也能採用本規範的大部分控制：

1. 把 [`workflow-policy.template.yaml`](../templates/workflow-policy.template.yaml) 視為現有四個 Role 的基礎上限；未列出的 Role 或 Capability 預設拒絕，不從 Handoff 動態擴權。
2. 在每個 Agent Adapter 固定 Role／Stage 的 Filesystem、Tool、Shell 與 Network 權限。
3. Coordinator 在已批准的 Change 文件中列出 Implementer 可修改的 Path。
4. Orchestrator 使用 Handoff 的精確 Reference 組裝輸入，不開放整個 Runtime Root。
5. 為每個 Role 分配固定、不可覆寫的 Own Output Path。
6. Reviewer 採 Source Read-only + Review Result Create-only。
7. Current State 只由 Orchestrator 或受控 Automation 更新。
8. Commit、Push、Deploy、Delete 與 Permission Change 保留 Human Gate。
9. Trace 至少保存 Policy Version、Requested Capability、Decision Code 與 Trace ID。
10. 未知 Capability、Scope 或 Policy Version 一律拒絕。

當 Boolean 權限已無法表達不同 Path、Tool Argument、Environment 或 Approval Condition，再新增獨立 Capability Policy Schema。升級前應先用 Trace 驗證實際需要的 Capability，避免把未使用的權限預先開放。

## 17. 驗收清單

### Identity 與 Policy

- [ ] 每次 Invocation 都固定 Actor ID、Role、Change ID、Run ID 與 Stage。
- [ ] Role、Runtime Identity 與 Capability 分開記錄。
- [ ] Policy 有 ID 與 Version；版本不相容時 fail closed。
- [ ] Prompt、Artifact 與 Agent Result 不能增加 Capability。

### Scope

- [ ] Effective Scope 由 Platform、Adapter、Role、Stage、Handoff、Run 與 Environment 取交集。
- [ ] Explicit Deny 優先。
- [ ] Path 經 Canonicalization 後仍位於批准 Root。
- [ ] `enumerate`、`read`、`create`、`update` 與 `delete` 分開控制。
- [ ] Tool ID、Action、Resource 與 Arguments 同時接受授權檢查。

### Role Boundary

- [ ] Implementer 只能修改被指派的 Worktree 與 Path。
- [ ] Reviewer 對 Source、Spec 與上游 Artifact 維持唯讀。
- [ ] Reviewer 只能建立自己的 Review Result，不能覆寫既有 Artifact。
- [ ] Readiness 與 Archive 只能建立各自的 Stage Output。
- [ ] Orchestrator 才能套用合法 Transition 並更新 Current State。

### Handoff 與 Approval

- [ ] Handoff 只能縮小既有權限。
- [ ] 現行 Handoff 沒有加入 Schema 未定義的 Capability 欄位。
- [ ] `requiresHumanApproval` 不被當成 Approval Proof。
- [ ] Reviewer Verdict、Human Approval、Transition Apply 與 Action Execution 分開記錄。
- [ ] Approval 綁定 Action、Scope、Policy Version、State Revision 與有效期。

### Audit 與失敗處理

- [ ] Allow、Deny 與 Pause 都有 Trace ID 與固定 Reason Code。
- [ ] 拒絕訊息不洩漏 Secret 或受保護資源資訊。
- [ ] Agent 遇到拒絕後不以其他 Tool 或參數變形繞過限制。
- [ ] Current State Revision 改變後重新評估授權。
- [ ] 未知 Capability 或缺少必要 Policy 時採預設拒絕。

## 18. 參考文件

- [Artifact-based Shared State + Structured Handoff 參考架構](./02-reference-architecture-zh-TW.md)
- [Repository Knowledge、Runtime State、Evidence 與 Trace 分層](./03-runtime-storage-and-retention-zh-TW.md)
- [安全、治理與長期維護](./05-security-and-maintainability-zh-TW.md)
- [Context Authority 與 Retrieval Policy](./06-context-authority-and-retrieval-policy-zh-TW.md)
- [Agent Platform Operations](../../../context-engineering/docs/03-agent-platform-operations-zh-TW.md)
- [Tool Governance and Evaluation](../../../agent-design/tool-schema-routing/docs/03-tool-governance-and-evaluation-zh-TW.md)
- [Handoff Envelope Schema](../schemas/handoff-envelope.schema.json)
- [Current State Schema](../schemas/current-state.schema.json)
- [Agent Result Schema](../schemas/agent-result.schema.json)
- [Workflow Policy Template](../templates/workflow-policy.template.yaml)
