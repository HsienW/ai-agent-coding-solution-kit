# Context Engineering Patterns

[![English](https://img.shields.io/badge/English-0969da?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-555555?style=for-the-badge)](./README-zh-CN.md)

This directory is reserved for small, reusable architecture patterns extracted from the longer reference documents.

[Back to module overview](../README.md)

## Pattern Format

```md
# Pattern Name

## Problem
## Context
## Forces
## Solution
## Structure
## Example
## Trade-offs
## Failure Modes
## When Not to Use
## Evaluation
```

## Planned Pattern Catalog

| Pattern | Problem It Solves |
|---|---|
| Route Before Assembly | Prevent every source and tool from being loaded |
| Progressive Disclosure | Delay expensive context until evidence is insufficient |
| Source Authority Ladder | Prevent stale memory from overriding live state |
| Hybrid Memory | Separate profile, summary, window, vector, and graph memory |
| Tool Result Pruning | Prevent large tool payloads from dominating prompts |
| Client Context Pack | Standardize client-provided context |
| Runtime Event Protocol | Make agent execution observable to a client |
| Structured Artifact Rendering | Render validated structures without inferring UI from prose |
| Checkpoint and Resume | Recover long-running workflows |
| Shadow Route Evaluation | Compare route policies without changing visible behavior |

Patterns should remain domain-neutral, use synthetic examples, define trade-offs, identify failure modes, and include measurable acceptance criteria.
