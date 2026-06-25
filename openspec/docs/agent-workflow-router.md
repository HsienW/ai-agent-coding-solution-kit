# OpenSpec Agent Workflow Router

[English](./agent-workflow-router.md) | [繁體中文](./agent-workflow-router-zh-TW.md)

OpenSpec Agent Workflow Router is a stage router for OpenSpec / SDD multi-agent work. It keeps prompts small by loading only the stage needed for the current task.

## Stages

| Stage | Purpose |
| --- | --- |
| `plan-change` | Turn a requirement into an OpenSpec change plan. |
| `review-plan` | Review proposal, design, tasks, and specs before implementation. |
| `apply-change` | Implement an approved change. |
| `review-result` | Review implementation output and validation results. |
| `fix-from-review` | Fix reviewer findings. |
| `readiness-check` | Decide whether the change is ready to archive. |
| `archive-change` | Archive and draft a commit message. |

## Layout

```text
openspec/docs/
openspec/prompts/
openspec/templates/codex-skill/
agent-workflows/
shared/
```

Use `openspec/prompts/` for copy-paste prompts with any agent. Use `openspec/templates/codex-skill/` to install the router as a Codex skill template.
