# Role Adapters

[English](./role-adapters.md) | [繁體中文](./role-adapters-zh-TW.md)

The workflow is tool-neutral.

| Generic Role | Responsibility |
| --- | --- |
| Planner | Creates or updates change plans. |
| Reviewer | Reviews plans and implementation results. |
| Implementer | Edits code, tests, and specs. |
| Coordinator | Checks readiness and gates. |
| Archivist | Archives changes and drafts commit messages. |

Example:

```text
Planner = CCR
Reviewer = Qwen
Implementer = Codex
Coordinator = CCR
Archivist = Codex plus human git owner
```
