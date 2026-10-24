# 🎓 AI Agent Code Solution Kit

[![English](https://img.shields.io/badge/English-555555?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-0969da?style=for-the-badge)](./README-zh-CN.md)

> 持续更新的 AI Agent Coding 文档、模板与工程实践知识库。

本仓库整理我在 AI Agent 开发与 AI 辅助编程过程中，实际研究、使用与验证过的文档、Prompt、模板、规范与工作流程。
这些零散经验会逐步整理成可阅读、可复用、可持续维护的工程资产，供个人项目与团队开发流程使用。

## 📚 主要内容

本仓库目前聚焦以下方向：

* **OpenSpec / SDD**：需求研究、规范设计、任务拆解、Change 管理与验收流程
* **Context Engineering**：上下文路由、渐进式披露、记忆设计与长任务管理
* **Prompt Engineering**：研究、规划、开发、Review 与调试 Prompt
* **Agent Design**：Agent 运行时架构、Tool Contract、Routing、Guardrail、治理与评测
* **Agent Workflows**：Claude Code、Codex CLI、Gemini CLI 等工具的协作模式
* **Agent Thoughts**: AI Agent basics and usage, plus experiences with AI in software development
* **可复用资产**：通用 Markdown 文档、模板、Checklist、Agent 规则与 Prompt
* **共享规则**：跨角色、跨流程重复使用的规则文档

## 📂 目录结构

| 方向 | 核心内容 | 阅读入口 |
| --- | --- | --- |
| OpenSpec / SDD | 规范驱动开发、阶段路由、规划与审查、实现验收与归档 | [OpenSpec 工作流路由](./openspec/openspec-agent-workflow-router/README-zh-TW.md) |
| Context Engineering | 上下文路由、渐进式披露、RAG、记忆设计、Runtime 与治理 | [上下文工程](./context-engineering/README-zh-TW.md) |
| Prompt Engineering | Prompt 契约、结构化产物生成、注入防护、评估与版本管理 | [Prompt 工程](./prompt-engineering/README-zh-TW.md)、[结构化产物生成](./prompt-engineering/prompts/structured-artifact-generation/) |
| Agent Design | Tool Schema、Tool Contract、能力路由、权限边界与评测 | [Agent 设计](./agent-design/README-zh-TW.md)、[Tool Schema 与 Routing](./agent-design/tool-schema-routing/README-zh-TW.md) |
| Agent Workflows | 多 Agent 生命周期、共享工作状态、结构化交接与协作流程 | [OpenSpec 多 Agent 生命周期](./agent-workflows/openspec-multi-agent-lifecycle/)、[共享状态与结构化交接](./agent-workflows/workflow-artifact-shared-state-handoff/README-zh-TW.md) |
| Agent Thoughts | AI Agent 基础认知、操作使用与 AI 软件开发心得 | [AI Agent 基础认知与操作](./agent-thoughts/ai-agent-basics/)、[AI 软件开发心得](./agent-thoughts/ai-software-development/) |
| 共享规则 | 跨角色、跨流程重复使用的规则文档 | [共享规则目录](./shared/) |

各类内容有明确分工：

* `docs`：概念、方法与工程实践记录
* `templates`：可直接复制、修改的文档骨架
* `prompts`：交给 AI Agent 使用的指令
* `patterns`：可跨项目复用的设计模式
* `examples`：实现与工作流程示例
* `schemas`：描述数据结构或交付物格式的规范

## 🌱 持续维护

本仓库会随 AI Coding Agent、OpenSpec、Context Engineering 与 Prompt Engineering 的实践发展持续更新。
内容主要来自个人研究与工程实践，使用前仍应根据实际模型、工具版本与项目环境完成验证。
