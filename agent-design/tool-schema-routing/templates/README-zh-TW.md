# Tool Schema 與 Routing 模板

[English](./README.md) | [繁體中文](./README-zh-TW.md)

以下模板可直接複製修改，且不綁定特定 Agent Framework。

## 1. Tool Design Record

```md
# Tool: <tool_name>

## Purpose
<只描述一項能力。>

## Use when
- <觸發意圖>
- <必要上下文>

## Do not use when
- <相似但必須排除的 Domain>
- <不安全或不支援的操作>

## Domain and risk
- Domain: <domain>
- Operation: read | write
- Risk: low | medium | high
- Owner: <team or service>

## Input contract
- Required fields:
- Optional fields:
- Conditional requirements:
- Identifier namespaces:
- Units and formats:

## Output contract
- Domain discriminator:
- Status enum:
- Allowed action enum:
- Reason codes:
- Freshness/source metadata:

## Execution policy
- Authorization:
- Idempotency:
- Timeout/retry:
- Rate limit:
- Audit:

## Fallback
<Fail-closed 或降級行為。>

## Evaluation
- Positive cases:
- Negative cases:
- Cross-domain cases:
- Injection/abuse cases:
```

## 2. Function Tool Schema

```ts
export const toolSchema = {
  type: "function",
  function: {
    name: "get_resource_state",
    description: `
取得 <resource> 狀態。
當 <selection conditions> 時使用。
返回 <structured result summary>。
不要用於 <excluded domains or operations>。
    `.trim(),
    parameters: {
      type: "object",
      additionalProperties: false,
      properties: {
        resourceId: {
          type: "string",
          pattern: "^resource_[A-Za-z0-9_-]{6,64}$",
          maxLength: 80,
          description: "<業務語義、格式與示例>。"
        },
        intent: {
          type: "string",
          enum: ["check_state", "get_allowed_action"],
          description: "本次要求的操作。"
        },
        surface: {
          type: "string",
          enum: ["assistant", "portal", "support"],
          description: "互動入口。"
        }
      },
      required: ["resourceId", "intent"]
    }
  }
} as const;
```

## 3. Output Contract

```ts
type ResourceToolResult = {
  domain: "resource_domain";
  resourceId: string;
  status: "ready" | "blocked" | "expired";
  allowedAction: "continue" | "view" | "none";
  reasonCode?: string;
  source: "system_of_record" | "policy_engine" | "cache";
  fetchedAt: string;
  traceId: string;
  schemaVersion: string;
};
```

## 4. Domain Registry Entry

```yaml
domain: resource_domain
displayName: Resource Domain
aliases:
  - resource
identifierPatterns:
  - '^resource_'
supportedSurfaces:
  - assistant
  - portal
identifyingFields:
  - resourceId
targetTools:
  - get_resource_state
groupedTool: null
riskLevel: medium
mutation: false
enabled: true
schemaVersion: 1.0.0
owner: resource-platform
```

## 5. Route Decision Record

```ts
type ToolRouteDecision = {
  requestId: string;
  domain: string | "unknown";
  confidence?: number;
  candidateToolNames: string[];
  routeSource:
    | "trusted_metadata"
    | "deterministic_rule"
    | "model_classifier"
    | "fallback";
  missingContext: string[];
  reasonCode: string;
  registryVersion: string;
};
```

## 6. Evaluation Case

```json
{
  "caseId": "route-cross-domain-001",
  "userText": "這個現在可以用嗎？",
  "structuredContext": {
    "declaredDomain": "redeemable_voucher",
    "resourceId": "voucher_demo_1001"
  },
  "expectedDomain": "redeemable_voucher",
  "allowedTools": ["get_redeemable_voucher_state"],
  "forbiddenTools": ["get_claimable_grant_state"],
  "expectedArguments": {
    "voucherId": "voucher_demo_1001",
    "intent": "check_usability"
  },
  "expectedBehavior": "call_tool",
  "riskTags": ["cross_domain", "structured_context_priority"]
}
```

## 7. Pull Request Checklist

```md
## Tool contract
- [ ] Name 明確且以動詞開頭。
- [ ] Description 包含使用、返回與排除條件。
- [ ] Input Field 包含格式、單位與約束。
- [ ] Output Contract 已版本化。
- [ ] Read／Write Operation 已分離。

## Routing
- [ ] 可信 Metadata 優先。
- [ ] Registry Entry 已版本化。
- [ ] Candidate Set 已縮小。
- [ ] Unknown 與 Conflict 能安全失敗。

## Safety and operations
- [ ] 有 Server-side Authorization。
- [ ] Write Operation 可冪等且有 Audit。
- [ ] Timeout、Retry 與 Fallback 已定義。
- [ ] 有 Kill Switch 與 Rollback。

## Evaluation
- [ ] 已新增 Positive Case。
- [ ] 已新增 Negative／No-tool Case。
- [ ] 已新增 Cross-domain Confusion Case。
- [ ] 已新增 Injection 與 Unauthorized Case。
- [ ] 已檢查 Token、Latency 與 Success Metrics。
```

## 8. Release Plan

```md
# Tool Release Plan

- Tool / domain:
- Schema versions:
- Router version:
- Registry version:
- Risk level:
- Owner:

## Stages
1. Offline Contract 與 Route 評測
2. Shadow Traffic
3. Read-only Canary
4. Limited Action Enablement
5. Controlled Rollout

## Success gates
- Selection accuracy:
- Domain mismatch rate:
- Argument validation failure rate:
- Unauthorized-action rejection rate:
- P95 latency:
- Token cost per successful task:

## Rollback
- Kill switch:
- Previous registry version:
- Previous prompt/model version:
- Deterministic fallback:
```
