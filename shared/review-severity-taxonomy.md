# Review Severity Taxonomy

[English](./review-severity-taxonomy.md) | [繁體中文](./review-severity-taxonomy-zh-TW.md)

## Blocker

Prevents implementation handoff, readiness, or archive. Examples: security flaw, data corruption risk, contract breakage, build failure, core workflow failure, direct spec conflict.

## Major

Should be fixed before archive. Examples: broken edge case, incomplete error/timeout/cancel handling, missing regression tests, cross-subsystem responsibility drift.

## Minor

Does not block archive by itself. Examples: naming, wording, readability, low-risk duplication.

Each finding should include severity, location, problem, trigger, impact, suggested fix, and related requirement/task/contract.
