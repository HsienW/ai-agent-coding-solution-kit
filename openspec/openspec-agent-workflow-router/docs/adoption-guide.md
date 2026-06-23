# Adoption Guide

[English](./adoption-guide.md) | [繁體中文](./adoption-guide-zh-TW.md)

## 1. Map Roles

Map the generic roles to your tools or people:

- Planner
- Reviewer
- Implementer
- Coordinator
- Archivist

## 2. Point To Project Rules

Use your project's rule files, such as `AGENTS.md`, tool-specific rule files, package rules, or domain docs.

## 3. Use One Stage At A Time

Ask for a specific stage:

```text
Use openspec-workflow-router to run {stage} for change {change_name}.
Context:
{summary_or_diff_or_validation_results}
```

## 4. Archive Only After Gates Pass

Archive only when validation passed, tasks are truthful, and no Blocker or Major remains.
