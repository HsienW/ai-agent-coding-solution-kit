# Tool Schema & Routing Templates

[English](./README.md) | [繁體中文](./README-zh-TW.md)

Copy and adapt these templates. They are intentionally framework-neutral.

## 1. Tool Design Record

```md
# Tool: <tool_name>

## Purpose
<One capability only.>

## Use when
- <triggering intent>
- <required context>

## Do not use when
- <similar but excluded domain>
- <unsafe or unsupported operation>

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
<Fail-closed or degraded behavior.>

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
Get <resource> state.
Use when <selection conditions>.
Returns <structured result summary>.
Do not use for <excluded domains or operations>.
    `.trim(),
    parameters: {
      type: "object",
      additionalProperties: false,
      properties: {
        resourceId: {
          type: "string",
          pattern: "^resource_[A-Za-z0-9_-]{6,64}$",
          maxLength: 80,
          description: "<Semantic meaning, format, and example>."
        },
        intent: {
          type: "string",
          enum: ["check_state", "get_allowed_action"],
          description: "Requested operation."
        },
        surface: {
          type: "string",
          enum: ["assistant", "portal", "support"],
          description: "Interaction surface."
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
  "userText": "Can I use this now?",
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
- [ ] Name is explicit and verb-first.
- [ ] Description includes use, return, and exclusion conditions.
- [ ] Input fields include formats, units, and constraints.
- [ ] Output contract is versioned.
- [ ] Read and write operations are separate.

## Routing
- [ ] Trusted metadata takes precedence.
- [ ] Registry entry is versioned.
- [ ] Candidate set is minimized.
- [ ] Unknown and conflicting states fail safely.

## Safety and operations
- [ ] Server-side authorization exists.
- [ ] Write operations are idempotent and audited.
- [ ] Timeout, retry, and fallback are defined.
- [ ] Kill switch and rollback are available.

## Evaluation
- [ ] Positive cases added.
- [ ] Negative/no-tool cases added.
- [ ] Cross-domain confusion cases added.
- [ ] Injection and unauthorized cases added.
- [ ] Token, latency, and success metrics reviewed.
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
1. Offline contract and route evaluation
2. Shadow traffic
3. Read-only canary
4. Limited action enablement
5. Controlled rollout

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
