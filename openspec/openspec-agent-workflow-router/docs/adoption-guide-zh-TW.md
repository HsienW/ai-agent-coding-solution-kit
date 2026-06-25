# 採用指南

[English](./adoption-guide.md) | [繁體中文](./adoption-guide-zh-TW.md)

## 1. 映射角色

將通用角色映射到你的工具或人：

- Planner
- Reviewer
- Implementer
- Coordinator
- Archivist

## 2. 指向專案規則

使用你的專案規則來源，例如 `AGENTS.md`、工具專屬規則、套件規則或 domain docs。

## 3. 一次只跑一個階段

指定明確階段：

```text
Use openspec-workflow-router to run {stage} for change {change_name}.
Context:
{summary_or_diff_or_validation_results}
```

## 4. Gate 通過後才 Archive

只有驗證通過、tasks 如實更新，且沒有 Blocker / Major 時，才進入 archive。
