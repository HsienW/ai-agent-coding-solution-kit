# Tool Schema & Routing Patterns

[English](./README.md) | [繁體中文](./README-zh-TW.md)

## Pattern 1: Specialized Boundaries, Shared Core

### Context

Several capabilities produce similar UI or artifacts but carry different domain
semantics, status models, authorization, or risk.

### Problem

A generic model-facing tool reduces code duplication but makes selection and
argument meaning ambiguous.

### Solution

- expose specialized schemas at the model boundary
- convert results through domain adapters
- reuse transport, validation, telemetry, fallback, and artifact conversion
- unify only after the domain output is validated

```text
Specialized Tool Schemas
        ↓
Domain Adapters
        ↓
Shared Tool Core
        ↓
Unified Artifact Model
```

### Consequences

Benefits:

- lower cross-domain confusion
- clearer authorization and ownership
- domain-specific evaluation
- shared internal implementation where semantics truly match

Costs:

- more schemas and tests
- explicit mapping code
- registry and version governance

### Do not use when

The capabilities are genuinely equivalent in semantics, risk, output contract,
and authorization. In that case a grouped tool may be simpler.

---

## Pattern 2: Stable Router Contract, Dynamic Registry

### Context

The tool catalog grows, and similar domains are added frequently.

### Problem

A router that hard-codes every domain into an enum becomes a central schema and
release bottleneck.

### Solution

- keep router input limited to generic evidence
- store domains, aliases, identifier patterns, surfaces, risk, and tool mapping in
  a versioned registry
- route deterministically when trusted metadata exists
- use a model classifier only for unresolved ambiguity
- return candidate tools, reason codes, and registry version

```text
Stable Router Contract
        ↓
Versioned Domain Registry
        ↓
Specialized or Grouped Tools
```

### Consequences

Benefits:

- low-risk domains can be added without router schema churn
- aliases and mappings are testable and reversible
- model sees a smaller candidate set

Costs:

- registry ownership and release process
- route regression tests
- risk of silent behavior changes if registry is not versioned

### Do not use when

The system has only a few stable tools and deterministic application logic can
select them directly.

---

## Pattern 3: High-risk Specialized, Low-risk Grouped

### Context

Creating one tool per subtype does not scale, while a single mega-tool destroys
semantic clarity.

### Problem

The system needs a principled way to choose tool granularity.

### Solution

Use a dedicated tool when a domain affects state, money, identity, eligibility,
policy, access, or irreversible actions. Group only read-only or display-only
subtypes that share the same meaning, output contract, and fallback policy.

```text
High-risk domain → Dedicated Tool
Low-risk long tail → Group Tool
```

### Consequences

Benefits:

- controlled catalog size
- clear safety boundaries
- lower token and review cost for long-tail display capabilities

Costs:

- risk classification must be maintained
- grouped tools need subtype validation
- a low-risk domain may later need promotion to a dedicated tool

### Do not use when

Risk is unknown. Treat unknown risk as requiring additional design review rather
than defaulting to a group tool.

---

## Pattern Selection Matrix

| Situation | Preferred pattern |
|---|---|
| Same UI, different meaning | Specialized Boundaries, Shared Core |
| Many changing domains | Stable Router Contract, Dynamic Registry |
| Catalog growth with mixed risk | High-risk Specialized, Low-risk Grouped |
| Trusted metadata identifies domain | Deterministic route; skip model router |
| Unknown or conflicting domain | Clarify or safe fallback |
