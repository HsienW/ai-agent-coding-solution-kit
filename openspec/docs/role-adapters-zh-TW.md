# 角色 Adapter

[English](./role-adapters.md) | [繁體中文](./role-adapters-zh-TW.md)

這套 workflow 不綁定特定工具。

| 通用角色 | 責任 |
| --- | --- |
| Planner | 建立或更新 change 計劃。 |
| Reviewer | 審查計劃與實作結果。 |
| Implementer | 修改 code、tests、specs。 |
| Coordinator | 判定 readiness 與流程 gate。 |
| Archivist | 執行 archive 並產生 commit message 建議。 |

範例：

```text
Planner = CCR
Reviewer = Qwen
Implementer = Codex
Coordinator = CCR
Archivist = Codex plus human git owner
```
