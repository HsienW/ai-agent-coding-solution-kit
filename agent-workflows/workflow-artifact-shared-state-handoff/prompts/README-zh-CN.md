# Prompts

[![English](https://img.shields.io/badge/English-555555?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-0969da?style=for-the-badge)](./README-zh-CN.md)

本目录提供工具中立的角色 Prompt。请按你的实际 CLI 或 Agent Framework 包装后使用。

角色分工：

- `coordinator.md`：规划、Scope、仲裁。
- `implementer.md`：实现、测试、Evidence。
- `reviewer.md`：只读 Review，只输出 `review_result`。
- `readiness-and-archive.md`：Gate 核对与最终摘要。

使用建议：

1. 每个角色只读 Handoff 要求的 Artifact。
2. 每个角色都输出符合 Schema 的 JSON。
3. 不把 Git Commit、Push、Deploy 授权给模型。
