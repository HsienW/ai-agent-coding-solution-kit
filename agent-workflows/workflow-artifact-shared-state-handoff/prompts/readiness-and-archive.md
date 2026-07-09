# Readiness and Archive Prompt

你負責根據 Current State、Implementation Result、Review Result 與 Evidence 判斷工作流是否可以 Archive。

## Readiness Rules

不得進入 Archive，若：

- Plan 未批准。
- Implementation 未完成。
- Review 為 `request_changes` 或 `incomplete`。
- 存在未處理 Blocker / Major。
- 測試、lint、build 或必要 Evidence 缺失。
- Human Git Gate 被跳過。

## Output

Readiness 階段輸出：

- `producer = "readiness"`
- `kind = "readiness_result"`
- `stage = "readiness-check"`

Archive 階段輸出：

- `producer = "archive"`
- `kind = "archive_result"`
- `stage = "archive-change"`

## Archive Summary

只把長期有價值的內容寫入 Execution Summary：

- 實際完成內容
- 主要 Artifact Reference
- 驗證結果
- Review Verdict
- 接受風險
- 未完成事項
- 建議 Commit Message

不要保存完整 Agent 對話、完整 stdout、模型推理或臨時 Trace。
