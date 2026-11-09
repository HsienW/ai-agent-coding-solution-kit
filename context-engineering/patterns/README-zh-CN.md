# 上下文工程 Patterns

[![English](https://img.shields.io/badge/English-555555?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-0969da?style=for-the-badge)](./README-zh-CN.md)

此目录用来放置从完整参考文件中抽出的、小型且可重复采用的架构 Pattern。

[返回模块首页](../README-zh-CN.md)

## Pattern 格式

```md
# Pattern Name

## Problem
## Context
## Forces
## Solution
## Structure
## Example
## Trade-offs
## Failure Modes
## When Not to Use
## Evaluation
```

## 预计 Pattern 清单

| Pattern | 解决问题 |
|---|---|
| Route Before Assembly | 避免所有数据源与 Tool 全部加载 |
| Progressive Disclosure | 证据不足时才展开高成本 Context |
| Source Authority Ladder | 避免过期 Memory 覆盖即时状态 |
| Hybrid Memory | 分离 Profile、Summary、Window、Vector 与 Graph Memory |
| Tool Result Pruning | 避免大型 Tool Payload 占满 Prompt |
| Client Context Pack | 标准化 Client 提供的 Context |
| Runtime Event Protocol | 让终端可以观测 Agent 执行 |
| Structured Artifact Rendering | 渲染验证后的结构，避免从文字反推 UI |
| Checkpoint and Resume | 恢复长时间工作流 |
| Shadow Route Evaluation | 在不改变可见结果时比较 Route Policy |

Pattern 应保持领域中立、使用合成示例、说明 Trade-off、列出 Failure Mode，并提供可测量验收标准。
