# Templates

這些模板可作為專案導入 Artifact-based Shared State + Structured Handoff 的起點。

- `current-state.template.json`：建立一個 Change / Run 的目前狀態。
- `handoff-envelope.template.json`：定義某一階段交給下一角色的工作契約。
- `workflow-policy.template.yaml`：集中描述角色權限、狀態轉移與保留策略。
- `execution-summary.template.md`：Change 完成後才進 Git 的長期摘要。
- `integration-checklist-zh-TW.md`：導入前檢查表。
- `gitignore.snippet`：建議加入 `.agent-runtime/` 忽略規則。
