# Execution Summary

## Change

- Change ID: `<change-id>`
- Run ID: `<run-id>`
- Final Phase: `<phase>`
- Final Verdict: `<ready | not-ready | incomplete>`

## What Changed

- `<short description>`

## Accepted Scope

- `<scope item>`

## Main Artifacts

- Coordinator Result: `<artifact ref>`
- Implementation Result: `<artifact ref>`
- Review Result: `<artifact ref>`
- Readiness Result: `<artifact ref>`

## Validation Evidence

| Check | Status | Evidence |
|---|---|---|
| `<check>` | `<passed / failed / skipped>` | `<evidence ref>` |

## Review Outcome

- Verdict: `<approve | request_changes | incomplete>`
- Remaining blocker: `<none or list>`

## Deviations From Original Plan

- `<none or deviation>`

## Accepted Risks

- `<risk and mitigation>`

## Human Gate

Commit, push, deploy, migration, deletion, and other irreversible operations remain controlled by a human approver.

## Recommended Commit Message

```text
<type(scope): message>
```
