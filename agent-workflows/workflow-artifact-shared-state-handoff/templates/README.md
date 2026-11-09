# Templates

[![English](https://img.shields.io/badge/English-0969da?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-555555?style=for-the-badge)](./README-zh-CN.md)

Use these templates as a starting point for adopting Artifact-based Shared State + Structured Handoff in a project.

- `current-state.template.json` records the current state of a Change / Run.
- `handoff-envelope.template.json` defines the work contract passed from one stage to the next role.
- `workflow-policy.template.yaml` collects role permissions, state transitions, and retention policies.
- `execution-summary.template.md` provides a durable summary added to Git after the Change is complete.
- `integration-checklist-zh-TW.md` provides a checklist for adoption.
- `gitignore.snippet` provides a suggested ignore rule for `.agent-runtime/`.
