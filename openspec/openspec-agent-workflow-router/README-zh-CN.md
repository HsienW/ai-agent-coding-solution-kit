# OpenSpec Agent Workflow Router

[![English](https://img.shields.io/badge/English-555555?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-0969da?style=for-the-badge)](./README-zh-CN.md)

## 这是什么

`openspec-agent-workflow-router` 是给 OpenSpec / SDD 项目使用的轻量 workflow router。它把大型多 agent 流程拆成多个阶段 prompt。

它不是每次都把完整 lifecycle prompt 丢给 agent，而是先判断当前阶段，再只加载对应 prompt：

```text
plan-change
review-plan
apply-change
review-result
fix-from-review
readiness-check
archive-change
```

## 目标

- 减少 token 浪费，一次只加载一个 stage prompt。
- 拆清楚 planning、review、implementation、fixing、readiness、archive 的责任。
- 让 Planner、Reviewer、Implementer、Coordinator、Archivist 之间的交接格式一致。
- 鼓励 spec-first development，而不是靠聊天记录直接实现。
- 默认让 git commit 与 push 仍由人控制。

## 解决什么问题

### Agent 读太多

大型 prompt 常让 agent 在只需要单一阶段时，也读入 planning、review、implementation、archive 的全部规则。这会增加成本，也让 agent 不够聚焦。

### Review 发生太晚

很多 AI coding 流程只在 code 写完后才 review。加入 `review-plan` 后，可以在实现前先抓出 design 与 tasks 问题。

### 交接格式不一致

不同 agent 常用不同输出格式。这套 router 统一 findings、verdict、validation summary 与 archive readiness。

### Specs 与 Code 漂移

如果没有明确 gate，tasks 可能在验证前就被勾选，或实现超出已批准 spec。这套流程把 tasks、validation、review、archive 绑在一起。

## 真实踩过的坑

实现上 agent workflow 中常见的问题：

- 还没判断阶段就扫描整个 repo。
- 不小心读取 `node_modules`、build output、coverage output、generated files 或 vendored code。
- 把聊天记录看得比已批准 OpenSpec artifacts 更高。
- 测试、build 或 validation 尚未通过就勾选 tasks。
- reviewer findings 尚未解决就进入 archive。
- 为了通过单一案例使用 hard-coded mapping、keyword shortcut 或一次性分支。
- 尚未确认是否仍有 Blocker / Major 就 archive。
- 最终 scope 与 validation 尚未确定就产生 commit message。

## 使用方式

可直接使用 copy-paste prompts：

```text
openspec/prompts/
```

或安装 Codex skill template：

```text
openspec/templates/codex-skill/openspec-workflow-router/
```

然后指定一个 stage：

```text
Use openspec-workflow-router to run review-result for change <change_name>.
Context:
<implementation summary, changed files, diff summary, validation results>
```

如果 stage 不明确，agent 应该只问一个简短问题，而不是一次加载多个 prompts。
