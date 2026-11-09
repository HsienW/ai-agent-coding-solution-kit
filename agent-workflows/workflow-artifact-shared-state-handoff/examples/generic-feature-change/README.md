# Generic Feature Change Example

[![English](https://img.shields.io/badge/English-0969da?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-555555?style=for-the-badge)](./README-zh-CN.md)

This example shows a complete, sanitized artifact flow for a generic feature change.

It intentionally avoids:

- Company names
- Product names
- Domain-specific business logic
- Model or CLI vendor names
- Credentials or private URLs

Suggested reading order:

1. `current-state.json`
2. `handoff-envelope.json`
3. `coordinator-result.json`
4. `implementation-result.json`
5. `review-result.json`
6. `readiness-result.json`
7. `archive-result.json`
8. `evidence/validation-summary.json`

The example is designed to validate against the schemas in `../../schemas/`.
