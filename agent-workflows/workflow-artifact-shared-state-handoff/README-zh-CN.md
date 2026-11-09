# Artifact-based Shared State + Structured Handoff

[![English](https://img.shields.io/badge/English-555555?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-0969da?style=for-the-badge)](./README-zh-CN.md)

> 从人工搬运上下文，演进到可验证、可持久化、可编排的多 Agent Coding Workflow。

## 工作流定位

这个工作流讨论一个常见但容易被低估的问题：

当规划者、实现者、审查者与人工批准者分别由不同 AI Agent 或 CLI 工具承担时，前一个角色的结论往往无法直接成为下一个角色的可靠输入。用户只好手工复制规格、修改摘要、Diff、测试结果与 Review Findings，并在每个阶段重新解释背景。

这种模式已经具有多角色分工，但人仍同时扮演：

- Context Router
- Clipboard Transport
- Shared State Store
- Workflow Engine
- Integration Arbiter
- Git Gatekeeper

本文提出两个互补机制，作为核心改造：

1. **Artifact-based Shared State**：保存各角色共同依据的工作事实。
2. **Structured Handoff**：定义任务如何从一个角色可靠交给下一个角色。

两者可简化为：

```text
Artifact-based Shared State
= 大家共同相信什麼

Structured Handoff
= 現在由誰，基於哪些事實，完成什麼工作
```

## 为什么值得单独设计？

Prompt、Skill、AGENTS.md 或其他规则资产可以降低重复指令，但它们主要解决“Agent 应如何工作”，不会自动解决：

- 现在处于哪一个阶段？
- 哪一份实现结果是最新且已接受的？
- 哪些测试真的执行过？
- Review 是否存在未处理的 Blocker？
- 下一个角色必须读取哪些输入？
- 失败后应回到实现、规格仲裁，还是人工处理？
- 是否允许 Archive、Commit、Push 或 Deploy？

只要这些信息仍存在人的记忆或剪贴簿里，流程就很难进一步自动化，也很难保证可恢复、可审计与可重跑。

## 核心架构

```text
┌──────────────────────────────────────────────┐
│ Durable Knowledge                            │
│ Requirements / Design / Tasks / Decisions    │
└──────────────────────┬───────────────────────┘
                       │ references
                       ▼
┌──────────────────────────────────────────────┐
│ Artifact-based Shared State                  │
│ Current State / Results / Evidence Refs      │
└──────────────────────┬───────────────────────┘
                       │ structured handoff
                       ▼
┌──────────────────────────────────────────────┐
│ Coordinator → Implementer → Reviewer         │
│      ↑                 │          │           │
│      └── arbitration ──┴── fixes ─┘           │
└──────────────────────┬───────────────────────┘
                       │ protected gate
                       ▼
┌──────────────────────────────────────────────┐
│ Human Approval                               │
│ Commit / Push / Deploy / Destructive Action  │
└──────────────────────────────────────────────┘
```

从架构责任来看：

- Shared State 是 **Data Plane**。
- Structured Handoff 是 **Control Plane**。
- Orchestrator 是未来负责执行状态转移的 **Process Manager**。
- Human-in-the-loop 是不可逆操作与高风险边界的 **Approval Gate**。

## 本工作流适用场景

适合：

- Planner、Coder、Reviewer 分别使用不同模型或 CLI。
- 已有 SDD、OpenSpec、ADR、Issue 或其他规格资产。
- Reviewer 采只读权限，不能直接修改代码。
- Commit、Push、Deploy 仍需要人工批准。
- 希望逐步导入 Node.js、Python 或 Workflow Engine 编排。
- 希望降低重复 Prompt 与上下文搬运，而不是追求完全无人化。

不一定需要：

- 单一 Agent 可以在一次短 Session 内完成的简单任务。
- 没有角色分工、没有审查 Gate 的一次性脚本生成。
- 完全不需要恢复、审计、重试或版本识别的实验。

## 这不是什么

本工作流不是：

- 特定模型供应商的专用方案。
- 特定 CLI 的操作教学。
- 一套必须安装的 Agent Framework。
- 要求所有状态都写成 Markdown 的文件治理方案。
- 让 LLM 自由决定所有流程与 Git 操作的自治系统。
- 宣称所有大型公司都采用同一实现的“唯一最佳实践”。

公开框架与协议在 Shared State、Artifact、Handoff、Persistence、HITL 等方向上有明显共通点，但具体数据模型、权限与存储方式仍应依团队规模与风险调整。

## 建议阅读顺序

1. [`docs/01-problem-and-maturity-model-zh-TW.md`](docs/01-problem-and-maturity-model-zh-TW.md)  
   从人工编排工作流开始，识别真正的瓶颈与成熟度。
2. [`docs/02-reference-architecture-zh-TW.md`](docs/02-reference-architecture-zh-TW.md)  
   定义 Shared State、Artifact、Handoff、State Machine 与角色边界。
3. [`docs/03-runtime-storage-and-retention-zh-TW.md`](docs/03-runtime-storage-and-retention-zh-TW.md)  
   解决 Repo 文件爆炸、Runtime State、Evidence 与 Trace 分离。
4. [`docs/04-adoption-roadmap-zh-TW.md`](docs/04-adoption-roadmap-zh-TW.md)  
   提供从人工流程到 Node Orchestrator 的分阶段导入路线。
5. [`docs/05-security-and-maintainability-zh-TW.md`](docs/05-security-and-maintainability-zh-TW.md)  
   处理路径安全、权限、Schema、版本、并发与长期维护。
6. [`patterns/README-zh-TW.md`](patterns/README-zh-CN.md)  
   将整套设计映射到 Blackboard、Repository、State Machine、Process Manager 等经典模式。

## 文件与工程资产

架构文件搭配 Schema、角色 Prompt、模板与示例，供整合工作流时使用。相关资产位于以下目录：

```text
artifact-shared-state-handoff/
├─ schemas/
│  ├─ agent-result.schema.json
│  ├─ handoff-envelope.schema.json
│  └─ current-state.schema.json
├─ templates/
├─ prompts/
└─ examples/
```

## 快速判断：Node.js 脚本是否已算多 Agent Orchestrator？

只有依序执行：

```text
run planner
run coder
run reviewer
```

通常仍只是自动化 Pipeline。

当脚本开始负责以下能力时，才接近真正的 Workflow Orchestrator：

- 加载并验证 Current State。
- 验证前置条件与 Agent 输出 Schema。
- 保存 Artifact 与 Evidence Reference。
- 根据结果推进或拒绝状态转移。
- 对 Review Findings 建立修正回路。
- 对设计冲突升级给 Coordinator。
- 在人工批准点 Pause／Resume。
- 支持 Retry、Idempotency 与 Recovery。

## 最小原则

可以用一句话概括本工作流：

> 先把人工流程转成稳定契约，再把稳定契约交给程序编排。

不要一开始就导入数据库、Queue、完整 Trace 与全自动 Git 操作。第一个可用版本只需要：

- 一份可验证的 Current State。
- 一种共用 Agent Result Envelope。
- 一种 Structured Handoff Envelope。
- 一张明确的状态转移表。
- 一个只读 Reviewer 边界。
- 一个人工 Commit Gate。

## 主要参考资料

- LangGraph：State 是节点共同使用的共享记忆；Checkpointer 可保存状态快照并支持 HITL 与故障恢复。  
  <https://docs.langchain.com/oss/javascript/langgraph/thinking-in-langgraph>  
  <https://docs.langchain.com/oss/javascript/langgraph/persistence>
- OpenAI Agents SDK：Handoff、Code-led／LLM-led Orchestration 与 Human-in-the-loop。  
  <https://openai.github.io/openai-agents-js/guides/handoffs/>  
  <https://openai.github.io/openai-agents-js/guides/multi-agent/>  
  <https://openai.github.io/openai-agents-js/guides/human-in-the-loop/>
- Agent2Agent Protocol：Task、Message、Artifact 与 Artifact Lifecycle。  
  <https://a2a-protocol.org/latest/topics/key-concepts/>  
  <https://a2a-protocol.org/latest/topics/life-of-a-task/>
- Enterprise Integration Patterns：Process Manager、Message、Pipes and Filters、Claim Check。  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html>
- Azure Architecture Center：Event Sourcing、Materialized View、Claim Check。  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing>  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view>  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check>

---

本工作流以框架中立、模型中立与规格工具中立为原则。采用前仍应依实际 CLI 版本、权限模型、数据敏感度与团队流程进行验证。
