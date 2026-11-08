# 上下文工程 Templates

[![English](https://img.shields.io/badge/English-555555?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-0969da?style=for-the-badge)](./README-zh-CN.md)

此目录用来放置由架构文件衍生、可直接复制使用的 Template。

[返回模块首页](../README-zh-CN.md)

## 预计 Templates

```text
templates/
├─ context-plan.template.md
├─ route-policy.template.md
├─ memory-policy.template.md
├─ tool-definition.template.md
├─ runtime-event.template.ts
├─ client-context-pack.template.ts
├─ artifact-schema.template.json
├─ evaluation-case.template.json
├─ context-trace.template.json
└─ release-checklist.template.md
```

每份 Template 应说明用途、定义 Required 与 Optional Fields、使用合成示例值、标示敏感字段、附上 Validation Notes，并连回标准文件。

## Context Plan Skeleton

```md
# Context Plan

## Task
- Objective:
- Risk level:
- Expected output:

## Route Decision
- Query route:
- Domain route:
- Source routes:
- Retrieval route:
- Memory routes:
- Tool route:
- Output route:
- Fallback route:

## Source Authority
- Authoritative sources:
- Advisory sources:
- Prohibited sources:

## Token Budget
- System:
- Interaction:
- Retrieval:
- Memory:
- Tools:
- Structured state:
- Reserved output:

## Progressive Disclosure
- Initial level:
- Expansion condition:
- Maximum level:

## Evaluation
- Correctness:
- Faithfulness:
- Token baseline:
- Latency budget:
- Fallback criteria:
```
