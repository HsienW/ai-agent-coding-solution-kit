# Prompts

[![English](https://img.shields.io/badge/English-0969da?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-555555?style=for-the-badge)](./README-zh-CN.md)

This directory provides tool-neutral role prompts. Adapt them to the CLI or Agent Framework in use.

Role responsibilities

- `coordinator.md` handles planning, scope, and arbitration.
- `implementer.md` handles implementation, testing, and evidence.
- `reviewer.md` performs read-only reviews and outputs only `review_result`.
- `readiness-and-archive.md` checks gates and produces the final summary.

Usage recommendations

1. Each role reads only the artifacts required by the handoff.
2. Each role outputs JSON that conforms to the schema.
3. Keep authorization for Git Commit, Push, and Deploy with a human.
