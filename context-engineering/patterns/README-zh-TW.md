# 上下文工程 Patterns

[English](./README.md) | [繁體中文](./README-zh-TW.md)

此目錄用來放置從完整參考文件中抽出的、小型且可重複採用的架構 Pattern。

[返回模組首頁](../README-zh-TW.md)

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

## 預計 Pattern 清單

| Pattern | 解決問題 |
|---|---|
| Route Before Assembly | 避免所有資料源與 Tool 全部載入 |
| Progressive Disclosure | 證據不足時才展開高成本 Context |
| Source Authority Ladder | 避免過期 Memory 覆蓋即時狀態 |
| Hybrid Memory | 分離 Profile、Summary、Window、Vector 與 Graph Memory |
| Tool Result Pruning | 避免大型 Tool Payload 佔滿 Prompt |
| Client Context Pack | 標準化 Client 提供的 Context |
| Runtime Event Protocol | 讓終端可以觀測 Agent 執行 |
| Structured Artifact Rendering | 渲染驗證後的結構，避免從文字反推 UI |
| Checkpoint and Resume | 恢復長時間工作流 |
| Shadow Route Evaluation | 在不改變可見結果時比較 Route Policy |

Pattern 應保持領域中立、使用合成示例、說明 Trade-off、列出 Failure Mode，並提供可量測驗收標準。
