# 🎓 AI Agent Code Solution Kit

[English](./README.md) | [繁体中文](./README-zh-TW.md)

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
* **可复用资产**：通用 Markdown 文档、模板、Checklist、Agent 规则与 Prompt
* **共享规则**：跨角色、跨流程重复使用的规则文档

## 📂 目录结构

```text
ai-agent-coding-solution-kit/
├─ openspec/
│  └─ openspec-agent-workflow-router/
│     ├─ docs/
│     ├─ prompts/
│     └─ templates/
│        └─ codex-skill/
├─ context-engineering/
│  ├─ docs/
│  ├─ patterns/
│  └─ templates/
├─ prompt-engineering/
│  ├─ docs/
│  ├─ prompts/
│  │  └─ structured-artifact-generation/
│  └─ templates/
├─ agent-design/
│  └─ tool-schema-routing/
│     ├─ docs/
│     ├─ patterns/
│     └─ templates/
├─ agent-workflows/
│  ├─ openspec-multi-agent-lifecycle/
│  └─ workflow-artifact-shared-state-handoff/
│     ├─ docs/
│     ├─ examples/
│     │  └─ generic-feature-change/
│     ├─ patterns/
│     ├─ prompts/
│     ├─ schemas/
│     └─ templates/
└─ shared/
```

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
