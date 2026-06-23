# Minimal Context Policy

[English](./minimal-context-policy.md) | [繁體中文](./minimal-context-policy-zh-TW.md)

預設讀取：

- 使用者摘要與限制。
- 指名檔案。
- Diff 摘要或 changed file list。
- 相關 OpenSpec artifacts。
- 相鄰 code 與 tests。
- 相關 project rules。

預設排除：

- dependency directories
- generated files
- build output
- coverage output
- caches
- vendored code
- unrelated lockfiles

只有在契約衝突、安全邊界、架構不明、無法解釋的驗證失敗或上游設計問題時才擴大上下文。
