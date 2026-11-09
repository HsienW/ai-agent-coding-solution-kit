# Schemas

[![English](https://img.shields.io/badge/English-0969da?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-555555?style=for-the-badge)](./README-zh-CN.md)

This directory provides three core contracts for multi-agent coding workflows.

- `agent-result.schema.json` defines the shared output envelope for Coordinator, Implementer, Reviewer, Readiness, and Archive.
- `handoff-envelope.schema.json` defines the input references, expected output, and transition policy passed between roles.
- `current-state.schema.json` defines the current workflow state, owner, gates, blockers, and next step.

Recommended approach

1. Validate the data structure against the schema.
2. Check context invariants in code, such as whether `changeId`, `runId`, `producer`, and `kind` match the current task.
3. Handle access control, secret scanning, and Git Gate decisions outside the schema.
