# Templates

[![English](https://img.shields.io/badge/English-555555?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-0969da?style=for-the-badge)](./README-zh-CN.md)

这些模板可作为项目导入 Artifact-based Shared State + Structured Handoff 的起点。

- `current-state.template.json`：建立一个 Change / Run 的当前状态。
- `handoff-envelope.template.json`：定义某一阶段交给下一角色的工作契约。
- `workflow-policy.template.yaml`：集中描述角色权限、状态转移与保留策略。
- `execution-summary.template.md`：Change 完成后才进 Git 的长期摘要。
- `integration-checklist-zh-TW.md`：导入前检查表。
- `gitignore.snippet`：建议加入 `.agent-runtime/` 忽略规则。
