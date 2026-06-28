# Agent Design

[English](./README.md) | [繁體中文](./README-zh-TW.md)

A framework-neutral collection of design methods, contracts, patterns, and
operational practices for building reliable AI agents.

> This pattern documents reference architectures.
> All scenarios, identifiers, policies, payloads, and metrics are synthetic.
> Production adoption requires domain-specific validation, security review,
> evaluation, and operational controls.

## What Belongs Here

`agent-design` focuses on the runtime and contract boundaries of an agent system:

- tool schemas and tool contracts
- routing and candidate selection
- state and workflow boundaries
- human-in-the-loop controls
- guardrails and authorization
- planning and execution policies
- evaluation, observability, release, and rollback

It complements other repository areas:

- `context-engineering` decides what information the model should see.
- `agent-workflows` describes how coding agents collaborate during development.
- `agent-design` defines how an application agent selects capabilities and acts
  safely at runtime.

## Patterns

### [Tool Schema & Routing](./tool-schema-routing/README.md)

A reference architecture for designing model-facing tool contracts, separating
similar domains, routing across large tool catalogs, and governing tool behavior.

Core ideas:

- specialized boundaries, shared core
- stable router contracts, dynamic registries
- high-risk specialized tools, low-risk grouped tools
- structured facts before model inference
- evaluation and observability built into the design
