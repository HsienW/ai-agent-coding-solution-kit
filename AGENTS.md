# ai-agent-coding-solution-kit Codex 規則

## 語言

- 與使用者互動一律使用繁體中文。
- 自然語言寫作任務至少支援繁體中文與英文。
- 不得輸出簡體中文，除非引用原文、程式碼、日誌或第三方內容。

## 自然語言寫作任務

當任務涉及以下內容時，視為自然語言寫作任務：

- README
- 文件
- 教學
- 架構說明
- 規格
- prompt
- changelog
- release note
- summary
- proposal
- review comment
- 繁體中文或英文內容改寫、潤稿、降 AI 味、語氣調整

執行自然語言寫作任務時，預設啟用 writing quality workflow，並優先評估使用以下 skills：

- `humanizer`
- `humanizer-zh-tw`
- `stop-slop`
- `de-AI-writing`
- `good-writing`

## Skill 使用規則

- 繁體中文寫作優先使用 `humanizer-zh-tw`。
- 英文寫作優先使用 `humanizer`。
- 所有自然語言輸出都應套用 `stop-slop` 類規則，避免空泛、模板化、過度行銷、AI 味明顯的文字。
- prompt、寫作指令、任務描述、內容生成流程，優先使用 `de-AI-writing`。
- 長文書寫(中文字符大於300個字)、寫作、改寫、終稿精修、風格審查、結構保真翻譯，優先使用 `good-writing`。
- 若某個 skill 不適用，需簡短說明略過原因。
