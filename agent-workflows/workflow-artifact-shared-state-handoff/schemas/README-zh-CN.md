# Schemas

[![English](https://img.shields.io/badge/English-555555?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-0969da?style=for-the-badge)](./README-zh-CN.md)

本目录提供多 Agent Coding Workflow 的三份核心契约：

- `agent-result.schema.json`：规范 Coordinator、Implementer、Reviewer、Readiness、Archive 的统一输出 Envelope。
- `handoff-envelope.schema.json`：规范角色间交接时需要传递的输入引用、预期输出与转移策略。
- `current-state.schema.json`：规范当前工作流状态、Owner、Gate、Blocker 与下一步。

建议做法：

1. 先用 Schema 验证数据形状。
2. 再用程序检查 Context Invariant，例如 `changeId`、`runId`、`producer`、`kind` 是否符合当前任务。
3. 不要让 Schema 直接承担权限控制、Secret 扫描或 Git Gate 判断。
