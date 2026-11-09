# Agent Design

[![English](https://img.shields.io/badge/English-555555?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-0969da?style=for-the-badge)](./README-zh-CN.md)

一套不绑定特定框架的 AI Agent 设计方法、契约、模式与运营实践集合，目标是建立可控、可评测、可治理的 Agent 系统。

> 这个 `模式` 记录参考架构。
> 所有场景、标识符、政策、Payload 与指标皆为合成数据。
> 生产落地仍需依实际业务完成安全、隐私、法务、可靠性与评测验证。

## 收录范围

`agent-design` 聚焦 Agent 运行时的能力与契约边界：

- Tool Schema 与 Tool Contract
- Tool Routing 与候选能力选择
- Agent 状态与工作流边界
- Human-in-the-loop 控制
- Guardrail、权限与授权
- 规划与执行策略
- 评测、可观测性、发布与回滚

它与仓库其他 `模式` 的分工如下：

- `context-engineering`：决定模型在每一步应看到什么信息。
- `agent-workflows`：描述研发期间多个 Coding Agent 如何协作。
- `agent-design`：定义应用 Agent 在运行时如何选能力、调工具并安全行动。

## 模式

### [Tool Schema 与 Routing](./tool-schema-routing/README-zh-CN.md)

一套用来设计模型可见 Tool Contract、隔离相似 Domain、路由大型 Tool Catalog，并治理 Tool 行为的参考架构。

核心观点：

- 语义分域、内核共享
- Router Contract 稳定，Domain Registry 动态扩展
- 高风险能力使用专属 Tool，低风险长尾能力使用 Group Tool
- 结构化事实优先于模型推断
- 评测与可观测性必须和 Schema 一起设计
