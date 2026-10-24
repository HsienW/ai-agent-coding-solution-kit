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
* **Agent Thoughts**: AI Agent fundamentals and usage, plus reflections on AI in software development
* **Reusable assets**: general-purpose Markdown documents, templates, Checklists, Agent rules, and Prompts
* **Shared rules**: rules used across roles and workflows

## 📂 Repository structure

| Area | Core topics | Reading entry points |
| --- | --- | --- |
| OpenSpec / SDD | Specification-driven development, stage routing, planning and review, implementation acceptance, and archiving | [OpenSpec workflow router](./openspec/openspec-agent-workflow-router/README.md) |
| Context Engineering | Context routing, progressive disclosure, RAG, memory design, runtime, and governance | [Context engineering](./context-engineering/README.md) |
| Prompt Engineering | Prompt contracts, structured artifact generation, injection prevention, evaluation, and version management | [Prompt engineering](./prompt-engineering/README.md), [Structured artifact generation](./prompt-engineering/prompts/structured-artifact-generation/) |
| Agent Design | Tool Schema, Tool Contract, capability routing, permission boundaries, and evaluation | [Agent design](./agent-design/README.md), [Tool Schema and Routing](./agent-design/tool-schema-routing/README.md) |
| Agent Workflows | Multi-agent lifecycles, shared work state, structured handoffs, and collaboration workflows | [OpenSpec multi-agent lifecycle](./agent-workflows/openspec-multi-agent-lifecycle/), [Shared state and structured handoff](./agent-workflows/workflow-artifact-shared-state-handoff/README.md) |
| Agent Thoughts | AI Agent fundamentals, usage, and reflections on AI in software development | [AI Agent fundamentals and usage](./agent-thoughts/ai-agent-basics/), [Reflections on AI in software development](./agent-thoughts/ai-software-development/) |
| Shared rules | Rules reused across roles and workflows | [Shared rules directory](./shared/) |

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
