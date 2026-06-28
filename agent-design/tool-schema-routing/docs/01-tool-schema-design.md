# Tool Schema Design for Reliable AI Agents

[English](./01-tool-schema-design.md) | [繁體中文](./01-tool-schema-design-zh-TW.md)

## 1. Tool Schema Is a Model-facing Contract

A normal function signature is consumed by deterministic code. A tool schema is
consumed by a probabilistic model that must first decide whether the capability
is relevant and then construct arguments from incomplete language.

A production tool definition therefore has at least four contracts:

1. **Selection contract:** name and description tell the model when to call it.
2. **Argument contract:** JSON Schema constrains fields, formats, and values.
3. **Output contract:** runtime validation constrains returned operational facts.
4. **Execution contract:** authorization, idempotency, timeout, retry, and audit
   constrain what the system may actually do.

Most SDKs expose only the input schema to the model. That does not make the other
three contracts optional.

## 2. Why Mechanical DRY Fails at the Model Boundary

Consider four resources with a similar visual result:

- Claimable Grant
- Redeemable Voucher
- Membership Entitlement
- Policy-based Subsidy

A generic input looks attractive:

```ts
const getBenefit = {
  name: "get_benefit",
  description: "Get benefit state.",
  parameters: {
    type: "object",
    properties: {
      id: { type: "string" },
      type: { type: "string" },
      status: { type: "string" }
    },
    required: ["id"]
  }
};
```

The shapes are shared, but the meanings are not:

| Domain | Primary action | Important state |
|---|---|---|
| Grant | claim | claimable, claimed, depleted, blocked |
| Voucher | redeem | usable, threshold-not-met, inapplicable, redeemed |
| Entitlement | authorize use | eligible, inactive, level-insufficient |
| Subsidy | verify qualification | region-ineligible, category-ineligible, quota-exhausted |

Compressing these states into `available` and `used` removes information the
model and runtime need to act correctly.

The preferred design is:

```text
Specialized model-facing schemas
        ↓
Domain adapters
        ↓
Shared validation, telemetry, fallback, and rendering core
        ↓
Unified artifact model
```

This is **specialized boundaries, shared core**.

## 3. High-quality Schema Principles

### 3.1 Naming expresses intent

Use an explicit verb, object, and meaningful qualifier.

Good:

```text
get_claimable_grant_state
get_redeemable_voucher_state
verify_policy_subsidy_eligibility
create_membership_activation_request
```

Weak:

```text
benefit
coupon
handle_resource
process_data
```

Avoid abbreviations that are not part of the product's public vocabulary.

### 3.2 Description provides decision information

A useful description contains:

```text
[Capability] + [When to use] + [Return summary] + [Exclusions / cautions]
```

Example:

```ts
description: `
Get the current state of a claimable grant.
Use when the user asks whether an allocated grant can be claimed, was already
claimed, expired, depleted, or is blocked.
Returns claim state, amount, expiration, reason code, and allowed next action.
Do not use for voucher redemption, membership access, or policy qualification.
`
```

Do not place volatile policy content in the description. The description should
help select a capability; live rules belong in a service, policy engine, or
retrieved source.

### 3.3 Parameters encode constraints

Each parameter should explain:

- semantic meaning
- accepted format
- units
- valid values
- examples when useful
- whether it is trusted metadata or user-derived input

Use JSON Schema constraints when supported:

```ts
{
  type: "string",
  pattern: "^grant_[A-Za-z0-9_-]{6,64}$",
  maxLength: 70,
  description: "Claimable grant identifier. Do not pass a voucher identifier."
}
```

Use `additionalProperties: false` to reduce invented arguments, while still
validating again on the server because model providers differ in enforcement.

### 3.4 Required fields reflect execution requirements

A field is required only if the operation cannot safely execute without it.

- **Required:** the runtime cannot identify or authorize the resource without it.
- **Optional:** a safe default exists or the field only narrows the result.
- **Conditionally required:** required for a specific intent or operation mode.

JSON Schema can express conditional requirements with `oneOf`, `anyOf`, `if`,
`then`, and `dependentRequired`, but provider support varies. Always duplicate
critical checks in runtime validation.

### 3.5 Examples guide format, not policy

Examples are useful for identifiers, dates, units, and enum values. Avoid
examples that accidentally encode confidential names or unstable business rules.

## 4. Specialized Input Schemas

### 4.1 Claimable Grant

```ts
export const getClaimableGrantStateTool = {
  type: "function",
  function: {
    name: "get_claimable_grant_state",
    description: `
Get the current state of a claimable grant.
Use for questions about claimability, previous claim, expiration, depletion,
or risk blocking.
Returns claim state, value, expiration, reason code, and allowed next action.
Do not use for voucher redemption, membership access, or policy subsidies.
    `.trim(),
    parameters: {
      type: "object",
      additionalProperties: false,
      properties: {
        grantId: {
          type: "string",
          pattern: "^grant_[A-Za-z0-9_-]{6,64}$",
          description:
            "Claimable grant identifier. Example: grant_demo_4821."
        },
        intent: {
          type: "string",
          enum: [
            "check_claimability",
            "check_claim_status",
            "check_expiration",
            "get_allowed_action"
          ],
          description: "The user's requested operation."
        },
        surface: {
          type: "string",
          enum: ["assistant", "checkout", "account_portal", "support"],
          description: "Interaction surface that initiated the request."
        }
      },
      required: ["grantId", "intent", "surface"]
    }
  }
} as const;
```

### 4.2 Redeemable Voucher

```ts
export const getRedeemableVoucherStateTool = {
  type: "function",
  function: {
    name: "get_redeemable_voucher_state",
    description: `
Get the state and redemption constraints of a voucher.
Use for usability, threshold, scope, and estimated redemption value.
Returns redemption state, face value, threshold, scope, and allowed next action.
Do not use to claim grants, activate memberships, or verify policy subsidies.
    `.trim(),
    parameters: {
      type: "object",
      additionalProperties: false,
      properties: {
        voucherId: {
          type: "string",
          pattern: "^voucher_[A-Za-z0-9_-]{6,64}$",
          description: "Voucher definition identifier."
        },
        holderVoucherId: {
          type: "string",
          pattern: "^holder_voucher_[A-Za-z0-9_-]{6,64}$",
          description:
            "Holder-specific voucher instance. Required when checking a user's current state."
        },
        intent: {
          type: "string",
          enum: [
            "check_usability",
            "explain_threshold",
            "check_scope",
            "estimate_redemption"
          ]
        },
        transactionContext: {
          type: "object",
          additionalProperties: false,
          properties: {
            resourceId: { type: "string", maxLength: 128 },
            estimatedAmountMinor: {
              type: "integer",
              minimum: 0,
              description: "Estimated amount in the smallest currency unit."
            },
            categoryCode: { type: "string", maxLength: 64 }
          }
        }
      },
      required: ["voucherId", "intent"]
    }
  }
} as const;
```

These tools may still share transport, retries, logging, and artifact conversion.
Their schemas differ because the model must preserve domain meaning.

## 5. Output Contracts

Do not rely on natural-language tool results. Validate structured output before
returning it to the model or client.

```ts
type BaseToolResult = {
  traceId: string;
  schemaVersion: string;
  source: "system_of_record" | "policy_engine" | "cache";
  fetchedAt: string;
};

type ClaimableGrantResult = BaseToolResult & {
  domain: "claimable_grant";
  grantId: string;
  claimStatus:
    | "claimable"
    | "claimed"
    | "expired"
    | "depleted"
    | "risk_blocked";
  valueMinor: number;
  allowedAction: "claim" | "view" | "none";
  reasonCode?: string;
};

type RedeemableVoucherResult = BaseToolResult & {
  domain: "redeemable_voucher";
  voucherId: string;
  redemptionStatus:
    | "usable"
    | "threshold_not_met"
    | "out_of_scope"
    | "redeemed"
    | "expired";
  faceValueMinor: number;
  thresholdMinor?: number;
  allowedAction: "redeem" | "view_rules" | "none";
  reasonCode?: string;
};
```

Important properties:

- a discriminating `domain` field
- domain-specific status and action enums
- source and freshness metadata
- stable machine-readable reason codes
- no model-generated operational facts

## 6. Unified Artifacts Without Semantic Collapse

Clients may render different domains through one artifact contract:

```ts
type BenefitArtifact = {
  artifactId: string;
  domain: string;
  title: string;
  valueText?: string;
  description: string;
  stateText: string;
  primaryAction: {
    type: string;
    label: string;
    enabled: boolean;
  };
  traceId: string;
};
```

The unification occurs **after** the domain result is validated. Do not use a
unified display model as the model-facing business input.

## 7. Token and Review Cost

Explicit names consume more tokens than `id`, `type`, and `status`. That cost is
usually justified when it prevents a wrong or unsafe call.

Control cost by:

- exposing only a route-specific candidate set
- keeping descriptions focused on selection rather than complete policy
- moving live rules to services or retrieval
- using stable shared wording for common constraints
- grouping only truly equivalent low-risk capabilities
- measuring token cost per successful call, not schema size alone

A short ambiguous schema is not cheaper if it causes retries, clarification,
wrong actions, incidents, or human correction.

## 8. Design Review Checklist

- [ ] Name starts with an explicit verb.
- [ ] Name identifies the resource or operation domain.
- [ ] Description states when to use and when not to use.
- [ ] Description summarizes returned facts and side effects.
- [ ] Every argument has semantic meaning, format, and units.
- [ ] Domain identifiers are not collapsed into a generic `id` without reason.
- [ ] Enums preserve domain semantics.
- [ ] `required` reflects safe execution requirements.
- [ ] `additionalProperties: false` is used where supported.
- [ ] Runtime validation repeats critical constraints.
- [ ] A versioned output contract exists.
- [ ] Authorization and idempotency are designed for write tools.
- [ ] Test cases include negative selection and cross-domain confusion.

## 9. Further Reading

- [Tool Schema 設計模式詳解](https://developer.volcengine.com/articles/7622979721047277606)
- [Registry-driven Tool Routing](./02-registry-driven-tool-routing.md)
- [Tool Governance, Evaluation, and Observability](./03-tool-governance-and-evaluation.md)
