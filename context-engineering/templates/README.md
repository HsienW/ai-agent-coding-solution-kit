# Context Engineering Templates

[![English](https://img.shields.io/badge/English-0969da?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-555555?style=for-the-badge)](./README-zh-CN.md)

This directory is reserved for copy-paste templates derived from the architecture documents.

[Back to module overview](../README.md)

## Planned Templates

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

Every template should state its purpose, define required and optional fields, use synthetic values, identify sensitive fields, include validation notes, and link to the canonical documentation.

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
