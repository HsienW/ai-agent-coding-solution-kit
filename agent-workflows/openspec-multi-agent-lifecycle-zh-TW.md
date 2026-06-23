[English](./openspec-multi-agent-lifecycle.md) | [繁體中文](./openspec-multi-agent-lifecycle-zh-TW.md)

# OpenSpec Multi-Agent Lifecycle

這份 lifecycle 描述一套不綁定特定工具的 OpenSpec / SDD 工作流。它把計劃、審查、實作、完成判定與封存拆開，讓每個 agent 只讀當前階段需要的 prompt 與上下文。

## Lifecycle

```text
Requirement
  -> plan-change
  -> review-plan
  -> apply-change
  -> review-result
  -> fix-from-review, if needed
  -> readiness-check
  -> archive-change
```

## 角色

| 角色 | 責任 |
| --- | --- |
| Planner | 將需求轉成 proposal、design、tasks 與 spec deltas。 |
| Reviewer | 審查計劃或實作結果，並提出 findings。 |
| Implementer | 實作已核准變更，或依 reviewer findings 修正。 |
| Coordinator | 檢查 requirements、tasks、validation 與 review gates 是否完成。 |
| Archivist | 執行 archive 步驟並產生 commit message 建議。 |

具體工具由專案自行映射。例如某些專案可將 Planner 對應到 CCR、Reviewer 對應到 Qwen、Implementer 對應到 Codex。

## 階段 Gate

### 1. plan-change

輸入：

- 需求摘要。
- 已知限制。
- 相關 project rules 或既有 specs。

輸出：

- Change name。
- Proposal draft。
- Design draft。
- Tasks draft。
- Validation plan。
- Open questions。

Gate：計劃必須具體到足以進入 review。

### 2. review-plan

輸入：

- Proposal。
- Design。
- Tasks。
- Specs 或 spec deltas。

輸出：

- Findings。
- Open questions。
- Verdict：`PASS`、`PASS_WITH_MINOR` 或 `FAIL`。

Gate：若計劃仍有未解 Blocker 或 Major，不應開始實作。

### 3. apply-change

輸入：

- 已核准 OpenSpec artifacts。
- 相關 project rules。
- 直接受影響檔案與 tests。

輸出：

- Implementation summary。
- Modified files。
- Validation results。
- 已更新 tasks，但只能在工作與驗證完成後更新。

Gate：不得在實作與驗證通過前勾選 tasks。

### 4. review-result

輸入：

- Implementation summary。
- Modified files。
- Diff summary。
- Validation results。
- 相關 OpenSpec artifacts。

輸出：

- Findings。
- Open questions。
- Verdict：`PASS`、`PASS_WITH_MINOR` 或 `FAIL`。

Gate：未解 Blocker 或 Major 必須回到 `fix-from-review`。

### 5. fix-from-review

輸入：

- Reviewer findings。
- Findings 指到的檔案與 tests。
- 相關 requirements 與 tasks。

輸出：

- Addressed findings。
- Modified files。
- Focused validation results。
- Remaining risks。

Gate：修正與相關驗證完成後，才重新進入 review 或 readiness。

### 6. readiness-check

輸入：

- Final implementation summary。
- Review verdict。
- Validation results。
- Changed file list 或 diff summary。

輸出：

- Verdict：`READY_TO_ARCHIVE` 或 `NOT_READY`。
- Rationale。
- Remaining work。
- Archive notes。

Gate：只有 `READY_TO_ARCHIVE` 才能 archive。

### 7. archive-change

輸入：

- Readiness verdict。
- Final OpenSpec artifacts。
- Validation results。

輸出：

- Archive result。
- Modified files。
- Validation result。
- Suggested commit message。

Gate：workflow 可以產生 commit message 建議，但 `git commit` 與 `git push` 預設仍由人控制。

## 上下文紀律

每個階段只應讀取：

- 對應 stage prompt。
- 使用者提供的摘要、diff 與 validation results。
- 相關 OpenSpec artifacts。
- 指名檔案與相鄰 tests。
- 相關 project rules。

避免掃描整個 repo。忽略 generated、vendored、dependency、build、cache、coverage 目錄。只有 dependency change 進入 scope 時才讀 lockfile。

## 失敗停止規則

遇到以下情況時停止並回報，不要硬往下做：

- Specs 與 project rules 衝突。
- Requirements 與既有 contracts 衝突。
- Review 仍有未解 Blocker 或 Major。
- 必要 validation 尚未執行。
- Diff 包含無關 scope。
