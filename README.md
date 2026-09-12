# 🎓 AI Agent Code Solution Kit

[繁體中文](./README-zh-TW.md) | [简体中文](./README-zh-CN.md)

> A continuously updated knowledge base of AI Agent Coding documentation, templates, and engineering practices.

This repository collects the documents, Prompts, templates, specifications, and workflows that I have researched, used, and verified while developing AI Agents and working with AI-assisted programming.
It turns scattered experience into readable, reusable, and maintainable engineering assets for personal projects and team development workflows.

## 📚 Contents

The repository currently focuses on these areas:

* **OpenSpec / SDD**: requirement research, specification design, task breakdown, Change management, and acceptance workflows
* **Context Engineering**: context routing, progressive disclosure, memory design, and long-running task management
* **Prompt Engineering**: Prompts for research, planning, development, Review, and debugging
* **Agent Design**: Agent runtime architecture, Tool Contracts, Routing, Guardrails, governance, and evaluation
* **Agent Workflows**: collaboration patterns for tools such as Claude Code, Codex CLI, and Gemini CLI
* **Reusable assets**: general-purpose Markdown documents, templates, Checklists, Agent rules, and Prompts
* **Shared rules**: rules used across roles and workflows

## 📂 Repository structure

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

Each content type has a distinct purpose:

* `docs`: concepts, methods, and engineering notes
* `templates`: document structures ready to copy and adapt
* `prompts`: instructions for AI Agents
* `patterns`: design patterns reusable across projects
* `examples`: implementation and workflow examples
* `schemas`: specifications that describe data structures or deliverable formats

## 🌱 Ongoing maintenance

This repository evolves with AI Coding Agent, OpenSpec, Context Engineering, and Prompt Engineering practices.
The material comes from personal research and engineering work. Validate it against the model, tool version, and project environment before adoption.
