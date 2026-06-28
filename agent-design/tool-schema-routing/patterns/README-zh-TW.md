# Tool Schema 與 Routing 設計模式

[English](./README.md) | [繁體中文](./README-zh-TW.md)

## Pattern 1: 語義分域、內核共享

### Context

多個能力可能生成相似 UI 或 Artifact，但 Domain 語義、狀態、授權或風險不同。

### Problem

模型可見 Tool 過度泛化，雖然減少代碼重複，卻會讓 Tool Selection 與 Argument 語義變模糊。

### Solution

- 模型邊界使用專屬 Schema
- 透過 Domain Adapter 轉換結果
- 內部復用 Transport、Validation、Telemetry、Fallback 與 Artifact Conversion
- Domain Output 驗證後才統一輸出

```text
Specialized Tool Schemas
        ↓
Domain Adapters
        ↓
Shared Tool Core
        ↓
Unified Artifact Model
```

### Consequences

收益：

- 降低跨域混淆
- 授權與 Owner 更清楚
- 可針對 Domain 做評測
- 真正相同的內部能力仍可復用

成本：

- Schema 與測試增加
- 需要顯式 Mapping
- 需要 Registry 與版本治理

### 不適用情況

能力在語義、風險、Output Contract 與授權上確實等價時，可直接使用 Group Tool。

---

## Pattern 2: Stable Router Contract, Dynamic Registry

### Context

Tool Catalog 持續擴張，且頻繁新增相似 Domain。

### Problem

把所有 Domain 寫死在 Router Enum，會讓 Router 變成中央 Schema 與發布瓶頸。

### Solution

- Router Input 只接收通用 Evidence
- Domain、Alias、ID Pattern、Surface、Risk 與 Tool Mapping 放進版本化 Registry
- 有可信 Metadata 時確定性路由
- 只有未解歧義才使用模型分類
- Route Result 返回 Candidate Tool、Reason Code 與 Registry Version

```text
Stable Router Contract
        ↓
Versioned Domain Registry
        ↓
Specialized or Grouped Tools
```

### Consequences

收益：

- 新增低風險 Domain 不需要改 Router Schema
- Alias 與 Mapping 可測試、可回滾
- 模型只看到較小 Candidate Set

成本：

- Registry 需要 Owner 與發布流程
- 需要 Route Regression Test
- Registry 未版本化會造成靜默行為改變

### 不適用情況

系統只有少量且穩定的 Tool，並且應用代碼已能完全確定性選擇時，不需要額外 Router。

---

## Pattern 3: 高風險專屬、低風險收斂

### Context

每個 Subtype 各建 Tool 會爆量；全部塞進 Mega Tool 又會丟失語義。

### Problem

系統需要一套可重複使用的 Tool Granularity 判斷方式。

### Solution

涉及狀態、金額、身份、資格、政策、存取或不可逆操作時，使用專屬 Tool。只有唯讀／展示子類型，且語義、Output Contract 與 Fallback 相同時，才收斂到 Group Tool。

```text
高風險 Domain → Dedicated Tool
低風險長尾 Domain → Group Tool
```

### Consequences

收益：

- 控制 Catalog 規模
- 保留安全邊界
- 降低長尾展示能力的 Token 與 Review 成本

成本：

- 需要維護 Risk Classification
- Group Tool 仍要驗證 Subtype
- 低風險 Domain 未來可能要升級成專屬 Tool

### 不適用情況

風險未知時，不應預設放進 Group Tool；應先進入額外 Design Review。

---

## Pattern 選擇矩陣

| 情況 | 建議模式 |
|---|---|
| 同一 UI、不同語義 | 語義分域、內核共享 |
| Domain 頻繁增加 | Stable Router Contract, Dynamic Registry |
| Catalog 成長且風險混合 | 高風險專屬、低風險收斂 |
| 可信 Metadata 已標示 Domain | 確定性路由，不調模型 Router |
| Domain 未知或互相衝突 | 澄清或安全 Fallback |
