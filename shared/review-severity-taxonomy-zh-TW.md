# Review Severity Taxonomy

[English](./review-severity-taxonomy.md) | [繁體中文](./review-severity-taxonomy-zh-TW.md)

## Blocker

會阻止 implementation handoff、readiness 或 archive。例：安全漏洞、資料毀損風險、契約破壞、build failure、核心流程失效、直接違反 spec。

## Major

archive 前應修復。例：重要邊界場景錯誤、error/timeout/cancel 處理不足、缺少 regression tests、subsystem 責任漂移。

## Minor

本身不阻擋 archive。例：命名、文字、可讀性、低風險重複。

每項 finding 應包含 severity、location、problem、trigger、impact、suggested fix、related requirement/task/contract。
