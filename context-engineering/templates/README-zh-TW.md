# 上下文工程 Templates

[English](./README.md) | [繁體中文](./README-zh-TW.md)

此目錄用來放置由架構文件衍生、可直接複製使用的 Template。

[返回模組首頁](../README-zh-TW.md)

## 預計 Templates

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

每份 Template 應說明用途、定義 Required 與 Optional Fields、使用合成示例值、標示敏感字段、附上 Validation Notes，並連回標準文件。

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
