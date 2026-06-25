# Minimal Context Policy

[English](./minimal-context-policy.md) | [繁體中文](./minimal-context-policy-zh-TW.md)

Default reads:

- User summaries and constraints.
- Named files.
- Diff summaries or changed file lists.
- Relevant OpenSpec artifacts.
- Adjacent code and tests.
- Relevant project rules.

Default exclusions:

- dependency directories
- generated files
- build output
- coverage output
- caches
- vendored code
- unrelated lockfiles

Expand only for contract conflicts, security boundaries, unclear architecture, unexplained validation failures, or upstream design issues.
