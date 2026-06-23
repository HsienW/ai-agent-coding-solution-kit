# OpenSpec Agent Workflow Router

[English](./agent-workflow-router-overview.md) | [繁體中文](./agent-workflow-router-overview-zh-TW.md)

## 這是什麼

`openspec-agent-workflow-router` 是給 OpenSpec / SDD 專案使用的輕量 workflow router。它把大型多 agent 流程拆成多個階段 prompt。

它不是每次都把完整 lifecycle prompt 丟給 agent，而是先判斷目前階段，再只載入對應 prompt：

```text
plan-change
review-plan
apply-change
review-result
fix-from-review
readiness-check
archive-change
```

## 目標

- 減少 token 浪費，一次只載入一個 stage prompt。
- 拆清楚 planning、review、implementation、fixing、readiness、archive 的責任。
- 讓 Planner、Reviewer、Implementer、Coordinator、Archivist 之間的交接格式一致。
- 鼓勵 spec-first development，而不是靠聊天紀錄直接實作。
- 預設讓 git commit 與 push 仍由人控制。

## 解決什麼問題

### Agent 讀太多

大型 prompt 常讓 agent 在只需要單一階段時，也讀入 planning、review、implementation、archive 的全部規則。這會增加成本，也讓 agent 不夠聚焦。

### Review 發生太晚

很多 AI coding 流程只在 code 寫完後才 review。加入 `review-plan` 後，可以在實作前先抓出 design 與 tasks 問題。

### 交接格式不一致

不同 agent 常用不同輸出格式。這套 router 統一 findings、verdict、validation summary 與 archive readiness。

### Specs 與 Code 漂移

如果沒有明確 gate，tasks 可能在驗證前就被勾選，或實作超出已核准 spec。這套流程把 tasks、validation、review、archive 綁在一起。

## 真實踩過的坑

實作上 agent workflow 中常見的問題：

- 還沒判斷階段就掃描整個 repo。
- 不小心讀取 `node_modules`、build output、coverage output、generated files 或 vendored code。
- 把聊天紀錄看得比已核准 OpenSpec artifacts 更高。
- 測試、build 或 validation 尚未通過就勾選 tasks。
- reviewer findings 尚未解決就進入 archive。
- 為了通過單一案例使用 hard-coded mapping、keyword shortcut 或一次性分支。
- 尚未確認是否仍有 Blocker / Major 就 archive。
- 最終 scope 與 validation 尚未確定就產生 commit message。

## 使用方式

可直接使用 copy-paste prompts：

```text
openspec/prompts/
```

或安裝 Codex skill template：

```text
openspec/templates/codex-skill/openspec-workflow-router/
```

然後指定一個 stage：

```text
Use openspec-workflow-router to run review-result for change <change_name>.
Context:
<implementation summary, changed files, diff summary, validation results>
```

如果 stage 不明確，agent 應該只問一個簡短問題，而不是一次載入多個 prompts。
