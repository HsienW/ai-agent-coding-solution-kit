# Tool Schema 与 Routing 设计模式

[![English](https://img.shields.io/badge/English-555555?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-0969da?style=for-the-badge)](./README-zh-CN.md)

## Pattern 1: 语义分域、内核共享

### Context

多个能力可能生成相似 UI 或 Artifact，但 Domain 语义、状态、授权或风险不同。

### Problem

模型可见 Tool 过度泛化，虽然减少代码重复，却会让 Tool Selection 与 Argument 语义变模糊。

### Solution

- 模型边界使用专属 Schema
- 通过 Domain Adapter 转换结果
- 内部复用 Transport、Validation、Telemetry、Fallback 与 Artifact Conversion
- Domain Output 验证后才统一输出

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
- 授权与 Owner 更清楚
- 可针对 Domain 做评测
- 真正相同的内部能力仍可复用

成本：

- Schema 与测试增加
- 需要显式 Mapping
- 需要 Registry 与版本治理

### 不适用情况

能力在语义、风险、Output Contract 与授权上确实等价时，可直接使用 Group Tool。

---

## Pattern 2: Stable Router Contract, Dynamic Registry

### Context

Tool Catalog 持续扩张，且频繁新增相似 Domain。

### Problem

把所有 Domain 写死在 Router Enum，会让 Router 变成中央 Schema 与发布瓶颈。

### Solution

- Router Input 只接收通用 Evidence
- Domain、Alias、ID Pattern、Surface、Risk 与 Tool Mapping 放进版本化 Registry
- 有可信 Metadata 时确定性路由
- 只有未解歧义才使用模型分类
- Route Result 返回 Candidate Tool、Reason Code 与 Registry Version

```text
Stable Router Contract
        ↓
Versioned Domain Registry
        ↓
Specialized or Grouped Tools
```

### Consequences

收益：

- 新增低风险 Domain 不需要改 Router Schema
- Alias 与 Mapping 可测试、可回滚
- 模型只看到较小 Candidate Set

成本：

- Registry 需要 Owner 与发布流程
- 需要 Route Regression Test
- Registry 未版本化会造成静默行为改变

### 不适用情况

系统只有少量且稳定的 Tool，并且应用代码已能完全确定性选择时，不需要额外 Router。

---

## Pattern 3: 高风险专属、低风险收敛

### Context

每个 Subtype 各建 Tool 会爆量；全部塞进 Mega Tool 又会丢失语义。

### Problem

系统需要一套可重复使用的 Tool Granularity 判断方式。

### Solution

涉及状态、金额、身份、资格、政策、访问或不可逆操作时，使用专属 Tool。只有只读／展示子类型，且语义、Output Contract 与 Fallback 相同时，才收敛到 Group Tool。

```text
高風險 Domain → Dedicated Tool
低風險長尾 Domain → Group Tool
```

### Consequences

收益：

- 控制 Catalog 规模
- 保留安全边界
- 降低长尾展示能力的 Token 与 Review 成本

成本：

- 需要维护 Risk Classification
- Group Tool 仍要验证 Subtype
- 低风险 Domain 未来可能要升级成专属 Tool

### 不适用情况

风险未知时，不应默认放进 Group Tool；应先进入额外 Design Review。

---

## Pattern 选择矩阵

| 情况 | 建议模式 |
|---|---|
| 同一 UI、不同语义 | 语义分域、内核共享 |
| Domain 频繁增加 | Stable Router Contract, Dynamic Registry |
| Catalog 成长且风险混合 | 高风险专属、低风险收敛 |
| 可信 Metadata 已标示 Domain | 确定性路由，不调模型 Router |
| Domain 未知或互相冲突 | 澄清或安全 Fallback |
