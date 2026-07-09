# 導入 Checklist

## 基礎契約

- [ ] 專案已放入三份 Schema。
- [ ] Agent 結果會通過 `agent-result.schema.json`。
- [ ] Handoff 只傳 Artifact Reference，不複製全文。
- [ ] Current State 可表達目前 Phase、Owner、Gate、Blocker 與下一步。

## Runtime 邊界

- [ ] `.agent-runtime/` 已加入 `.gitignore`。
- [ ] Runtime Artifact 不含 Secret、Token、API Key 或公司內部敏感資料。
- [ ] Artifact URI 不允許 `../` 或任意絕對路徑。
- [ ] Reviewer 只能讀取 Handoff 明確引用的 Artifact。

## 角色權限

- [ ] Reviewer 預設唯讀。
- [ ] Implementer 是主要程式碼寫入者。
- [ ] Coordinator 負責 Scope、仲裁與 Gate 判定。
- [ ] Commit / Push / Deploy 保留人工批准。

## 驗證

- [ ] 所有 JSON 範例可被 Schema 驗證。
- [ ] Agent 輸出通過 Schema 後，仍會檢查 `changeId`、`runId`、`producer`、`kind`。
- [ ] Review 有 Blocker / Major 時不得進入 Archive。
- [ ] 測試失敗或 Evidence 不足時不得標記 Ready。

## 後續升級

- [ ] 若多人協作需要共享 Runtime，將 `.agent-runtime/` 升級為本機 Projection，另建遠端 Runtime Store。
- [ ] 若需要自動流轉，再導入 Orchestrator；不要讓腳本一開始就自動 Commit。
