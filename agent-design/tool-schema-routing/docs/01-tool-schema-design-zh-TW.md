# 可靠 AI Agent 的 Tool Schema 設計

[English](./01-tool-schema-design.md) | [繁體中文](./01-tool-schema-design-zh-TW.md)

## 1. Tool Schema 是模型可見契約

普通函數簽名由確定性程序消費；Tool Schema 則由概率模型消費。模型必須先判斷能力是否相關，再從不完整的自然語言中組裝參數。

因此，生產級 Tool 至少有四層契約：

1. **選擇契約**：name 與 description 告訴模型何時應調用。
2. **參數契約**：JSON Schema 約束字段、格式與可選值。
3. **輸出契約**：Runtime Validation 約束 Tool 返回的操作事實。
4. **執行契約**：授權、冪等、Timeout、Retry 與 Audit 約束系統實際可做的事。

多數 SDK 只把 Input Schema 暴露給模型，不代表另外三層可以省略。

## 2. 為什麼不能機械套用 DRY

以下四類資產最終可能渲染成相似卡片：

- Claimable Grant：可領取資產
- Redeemable Voucher：可核銷憑證
- Membership Entitlement：會員權益
- Policy-based Subsidy：政策型補助

泛化輸入看起來很省事：

```ts
const getBenefit = {
  name: "get_benefit",
  description: "取得優惠狀態。",
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

但 JSON Shape 相似，不代表業務語義相同：

| Domain | 核心動作 | 重要狀態 |
|---|---|---|
| Grant | 領取 | 可領、已領、已耗盡、風控阻擋 |
| Voucher | 核銷 | 可用、未達門檻、不適用、已核銷 |
| Entitlement | 授權使用 | 有資格、未啟用、等級不足 |
| Subsidy | 資格核驗 | 地區不符、品類不符、額度耗盡 |

把這些狀態壓成 `available` 與 `used`，會丟失模型與 Runtime 做正確判斷所需的資訊。

建議架構是：

```text
模型可見 Schema 分域
        ↓
Domain Adapter
        ↓
共享驗證、埋點、Fallback 與渲染 Core
        ↓
統一 Artifact Model
```

這就是 **語義分域、內核共享（Specialized Boundaries, Shared Core）**。

## 3. 高品質 Schema 原則

### 3.1 命名即意圖

使用明確的動詞、對象與限定條件。

推薦：

```text
get_claimable_grant_state
get_redeemable_voucher_state
verify_policy_subsidy_eligibility
create_membership_activation_request
```

不推薦：

```text
benefit
coupon
handle_resource
process_data
```

除非縮寫已是公開且穩定的領域詞彙，否則不要用縮寫壓縮 Tool Name。

### 3.2 Description 即決策文檔

有效 description 應包含：

```text
[功能] + [使用時機] + [返回摘要] + [排除場景／注意事項]
```

```ts
description: `
取得可領取資產的即時狀態。
當使用者詢問是否可領、是否已領、是否過期、是否耗盡或遭風控阻擋時使用。
返回領取狀態、價值、過期時間、原因碼與允許的下一步操作。
不要用於核銷憑證、會員權益或政策補助。
`
```

不要把頻繁變動的政策規則塞進 description。description 的責任是協助選 Tool；即時規則應由服務、Policy Engine 或 Retrieval 提供。

### 3.3 Parameter 即約束

每個參數應說明：

- 業務語義
- 接受格式
- 單位
- 可選值
- 必要示例
- 是可信 Metadata 還是由使用者文字抽取

```ts
{
  type: "string",
  pattern: "^grant_[A-Za-z0-9_-]{6,64}$",
  maxLength: 70,
  description: "可領取資產 ID，不可傳入 Voucher ID。"
}
```

支援時應使用 `additionalProperties: false` 降低模型杜撰參數；但各模型供應商的 Enforcement 不一致，Server 仍必須再次驗證。

### 3.4 Required／Optional 必須有依據

- **Required**：缺少便無法安全識別、授權或執行。
- **Optional**：有安全預設值，或只用來縮小結果範圍。
- **Conditionally required**：特定 intent 或 operation mode 下才需要。

JSON Schema 可用 `oneOf`、`anyOf`、`if/then`、`dependentRequired` 表達條件必填，但供應商支援度不同，因此關鍵條件必須在 Runtime 再驗一次。

### 3.5 Example 用來引導格式，不用來固化政策

示例適合表達 ID、日期、單位和 Enum；不要放真實內部名稱，也不要讓示例意外成為過期業務規則。

## 4. 分域 Input Schema

### 4.1 Claimable Grant

```ts
export const getClaimableGrantStateTool = {
  type: "function",
  function: {
    name: "get_claimable_grant_state",
    description: `
取得可領取資產的即時狀態。
用於查詢能否領取、是否已領、是否過期、是否耗盡或遭風控阻擋。
返回領取狀態、價值、過期時間、原因碼與允許操作。
不要用於 Voucher 核銷、會員權益或政策補助。
    `.trim(),
    parameters: {
      type: "object",
      additionalProperties: false,
      properties: {
        grantId: {
          type: "string",
          pattern: "^grant_[A-Za-z0-9_-]{6,64}$",
          description: "可領取資產 ID，例如 grant_demo_4821。"
        },
        intent: {
          type: "string",
          enum: [
            "check_claimability",
            "check_claim_status",
            "check_expiration",
            "get_allowed_action"
          ],
          description: "使用者本次要求的操作。"
        },
        surface: {
          type: "string",
          enum: ["assistant", "checkout", "account_portal", "support"],
          description: "觸發請求的互動入口。"
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
取得 Voucher 狀態與核銷限制。
用於查詢可用性、門檻、適用範圍與預估核銷價值。
返回核銷狀態、面值、門檻、範圍與允許操作。
不要用於領取 Grant、啟用會員或核驗政策補助。
    `.trim(),
    parameters: {
      type: "object",
      additionalProperties: false,
      properties: {
        voucherId: {
          type: "string",
          pattern: "^voucher_[A-Za-z0-9_-]{6,64}$",
          description: "Voucher 定義 ID。"
        },
        holderVoucherId: {
          type: "string",
          pattern: "^holder_voucher_[A-Za-z0-9_-]{6,64}$",
          description: "持有人 Voucher 實例；查個人即時狀態時需要。"
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
              description: "預估金額，使用最小貨幣單位。"
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

兩個 Tool 仍可共用 Transport、Retry、Logger、Trace 和 Artifact 轉換；Schema 分開是因為模型必須保留 Domain 語義。

## 5. Output Contract

不要讓 Tool 只返回自然語言。結果送回模型或 Client 前，必須完成結構化驗證。

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

重要屬性：

- `domain` 判別字段
- Domain-specific Status 與 Action Enum
- Source 與 Freshness Metadata
- 穩定的 Machine-readable Reason Code
- 不允許模型杜撰操作事實

## 6. 可以統一渲染，但不能提前抹平語義

Client 最後可以共用 Artifact：

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

統一發生在 Domain Result 驗證之後。不要反過來拿 Display Model 當作模型可見的業務輸入。

## 7. Token 與 Review 成本

`grantId` 比 `id` 長，`redemptionStatus` 也比 `status` 長。但只要能避免錯調或不安全操作，這點 Token 通常值得。

控制成本的方法：

- 只暴露路由後的候選 Tool 集合
- description 聚焦選擇，不承載完整政策
- 即時規則放在 Service 或 Retrieval
- 共用穩定的約束文案
- 只收斂真正等價的低風險能力
- 衡量每次成功 Tool Call 的總 Token，而非只看 Schema 長度

模糊 Schema 看起來較短，但若造成重試、澄清、錯誤操作、事故或人工修正，總成本反而更高。

## 8. Design Review Checklist

- [ ] Tool Name 以明確動詞開頭。
- [ ] Tool Name 能辨識資源或操作 Domain。
- [ ] description 說明何時使用與何時不要使用。
- [ ] description 摘要返回事實與 Side Effect。
- [ ] 每個參數有語義、格式與單位。
- [ ] Domain ID 不在無必要下壓成泛 `id`。
- [ ] Enum 保留 Domain 語義。
- [ ] `required` 對應安全執行的最低需求。
- [ ] 供應商支援時使用 `additionalProperties: false`。
- [ ] Runtime 重複驗證關鍵約束。
- [ ] 有版本化 Output Contract。
- [ ] Write Tool 有授權與冪等設計。
- [ ] 測試包含負向選擇與跨域混淆案例。

## 9. 延伸閱讀

- [Tool Schema 設計模式詳解](https://developer.volcengine.com/articles/7622979721047277606)
- [Registry-driven Tool Routing](./02-registry-driven-tool-routing-zh-TW.md)
- [Tool 治理、評測與可觀測性](./03-tool-governance-and-evaluation-zh-TW.md)
