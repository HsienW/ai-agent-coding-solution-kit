# Tool 治理、評測與可觀測性

[English](./03-tool-governance-and-evaluation.md) | [繁體中文](./03-tool-governance-and-evaluation-zh-TW.md)

## 1. Schema 能通過編譯，不代表設計完成

生產級 Tool 是一項受治理的能力，必須有 Owner、生命週期、評測集、發布策略與營運控制。

```text
設計
→ Review
→ 離線評測
→ Shadow Traffic
→ 小流量唯讀發布
→ 受控開啟 Action
→ 持續監控
→ 修訂或回滾
```

## 2. Tool Registry Metadata

```ts
type ToolRegistration = {
  toolName: string;
  domain: string;
  owner: string;
  descriptionVersion: string;
  inputSchemaVersion: string;
  outputSchemaVersion: string;
  riskLevel: "low" | "medium" | "high";
  operationType: "read" | "write";
  authorizationPolicy: string;
  idempotencyPolicy?: string;
  timeoutMs: number;
  retryPolicy: string;
  fallbackPolicy: string;
  releaseStatus: "draft" | "shadow" | "canary" | "active" | "disabled";
  killSwitchKey: string;
  evaluationDataset: string;
};
```

Registry 應為 Machine-readable，並放在 Source Control 或受審批的 Configuration System 中版本化。

## 3. 版本策略

下列項目應分開版本化：

- Tool Name 與 Selection Description
- Input Schema
- Output Contract
- Execution Behavior
- Domain Registry 與 Alias
- Prompt 或 Routing Policy

即使只優化 description、沒有改 API，也可能改變模型行為，因此仍要評測並寫 Release Note。

建議規則：

- 新增 Optional Input：Schema Minor Version
- 新增 Enum：只有所有 Consumer 都能容忍未知值時才算 Minor
- Rename／Remove Field：Major Version
- Side Effect 或 Authorization 改變：Behavior Major Version
- Alias 或 Route Priority 改變：Registry Version + Route Regression Test

## 4. 評測資料集

資料集不能只有 Happy Path。

```ts
type ToolEvaluationCase = {
  caseId: string;
  userText: string;
  structuredContext?: Record<string, unknown>;
  expectedDomain: string | "none";
  allowedTools: string[];
  forbiddenTools: string[];
  expectedArguments?: Record<string, unknown>;
  expectedMissingArguments?: string[];
  expectedBehavior:
    | "call_tool"
    | "clarify"
    | "answer_without_tool"
    | "reject";
  riskTags: string[];
};
```

應包含：

- 明確請求
- 隱含意圖
- 模糊語句
- 缺少 ID
- 結構化上下文互相衝突
- 跨 Domain 近似案例
- 不應調任何 Tool 的 Negative Case
- Prompt Injection
- 未授權寫操作
- 歷史上下文與當前資源衝突
- 多語言與 Paraphrase

## 5. 核心指標

### Selection

```text
Tool selection accuracy
Domain classification accuracy
No-tool precision and recall
Cross-domain mismatch rate
Candidate shortlist recall
```

### Arguments

```text
Exact argument match
Required-field completion rate
Identifier-type mismatch rate
Enum validity rate
Unit and format accuracy
Runtime validation failure rate
```

### Execution and safety

```text
Tool success rate
Timeout and retry rate
Unauthorized-action rejection rate
Idempotency conflict rate
Fallback rate
Unsafe side-effect rate
```

### User and cost outcomes

```text
Human correction rate
Task completion rate
Clarification rate
Latency per successful task
Token cost per successful task
Tool calls per successful task
```

不能只優化 Selection Accuracy。Router 選對 Domain，仍可能產生危險參數或執行未授權操作。

## 6. Domain Mismatch

當可信的當前上下文與選中的 Tool 不一致，就是 Mismatch：

```text
current structured domain = claimable_grant
selected tool domain      = redeemable_voucher
result                     = domain mismatch
```

參考公式：

```text
domain_mismatch_rate = mismatched_calls / calls_with_trusted_domain
```

應依以下維度切分：

- Model 與 Model Version
- Prompt Version
- Router Version
- Registry Version
- Surface
- Domain Pair
- Tool Risk Level

整體 Mismatch 很低，也可能掩蓋兩個高風險 Domain 之間的嚴重誤調。

## 7. Trace Model

```ts
type ToolTrace = {
  requestId: string;
  sessionId?: string;
  route: {
    registryVersion: string;
    declaredDomain?: string;
    selectedDomain: string;
    routeSource: string;
    confidence?: number;
    candidateTools: string[];
  };
  call: {
    toolName: string;
    inputSchemaVersion: string;
    argumentsHash: string;
    validationResult: "pass" | "fail";
    authorizationResult: "allow" | "deny";
    startedAt: string;
    durationMs: number;
    outcome: string;
  };
  result: {
    outputSchemaVersion: string;
    source: string;
    freshnessMs?: number;
    allowedAction?: string;
  };
};
```

不要記錄 Raw Secret 或敏感個資；依 Data Classification Policy 對 Argument 做 Hash 或 Redaction。

## 8. 灰度策略

### Stage 0: Offline

- Schema Lint
- Contract Test
- Routing Regression Test
- Adversarial 與 Injection Test

### Stage 1: Shadow

新 Router 或 Schema 觀察真實流量，但不控制 Action。和現有路徑比較 Domain、參數、延遲與成本。

### Stage 2: Read-only Explanation

允許真實 Read Tool 與解釋，但操作仍走既有確定性流程。

### Stage 3: Limited Action Enablement

小流量開啟低風險 Action，並強制 Authorization 與 Audit。

### Stage 4: Controlled Rollout

按 Domain 與 Risk Level 擴量，不只看全局百分比。

### Stage 5: Full Release with Rollback

必須保留：

- Domain-level Kill Switch
- Tool-level Disable Flag
- Registry Rollback
- Prompt／Model Rollback
- 回退確定性 Workflow

## 9. 安全與授權

Tool Description 不是安全控制。

每個 Write Tool 都必須：

- 驗證 Caller 身份
- 在 Server 端授權 Actor、Resource 與 Action
- 以 System of Record 驗證參數
- 使用 Idempotency Key
- 套用 Rate Limit 與 Abuse Control
- 寫入不可竄改 Audit Event
- 支援安全 Retry
- 必要時要求 Confirmation 或 Approval

Read Tool 與 Write Tool 應分開。泛化 `manage_resource` 比明確讀寫 Tool 更難授權、評測與治理。

## 10. Fallback Policy

Fallback 必須明確且安全：

```text
參數無效
→ 不調 Backend；要求補上下文或 Fail Closed

授權拒絕
→ 不改參數重試；說明允許的下一步

Timeout 或 Dependency Failure
→ 只按 Policy Retry；否則返回暫時不可用 Artifact

未知 Domain
→ 不映射到最接近 Tool；澄清或使用非操作型回覆

Output Contract 失敗
→ 隔離結果、告警，不從該結果生成 Action
```

## 11. 事故排查順序

1. 可信上下文與 Source Authority
2. Router 與 Registry Version
3. Model 與 Prompt Version
4. Candidate Tool Set
5. 生成參數與 Schema Validation
6. Authorization Decision
7. Backend Result 與 Output Validation
8. Artifact／Action Mapping
9. Release Cohort 與 Feature Flag

修復應落在最窄且正確的層。不要用更長 description 掩蓋 Backend Policy 錯誤。

## 12. Governance Review Checklist

- [ ] Tool 有 Owner 與 Risk Level。
- [ ] Read／Write Operation 已分離。
- [ ] Input／Output Schema 已版本化。
- [ ] Authorization 與 Idempotency 在 Server 端執行。
- [ ] Registry Alias 與 Route Priority 已版本化。
- [ ] 評測包含 Negative 與 Cross-domain Case。
- [ ] Trace 能串起 Route、Call、Result 與 Artifact。
- [ ] 敏感 Argument 已 Redact 或 Hash。
- [ ] 有 Shadow 與 Canary Stage。
- [ ] Domain-level Kill Switch 與 Rollback 已演練。
- [ ] 成本以 Successful Task 為單位量測。
- [ ] 高風險 Action 遇到 Output Contract 失敗時 Fail Closed。
