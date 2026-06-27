# ADR-012: Component Reference Propagation through the Runtime Plan

**Status:** ACCEPTED
**Date:** 2026-06-27
**Deciders:** Runtime Protocol Architect
**Touches frozen model:** YES — `RuntimePipelinePlan.RuntimeNode`

---

## Context

ADR-009 authorized the frozen `PipelineSpec.NodeSpec` change: a node references a
Component via **`plugin_id` + `component_id`** (two fields). ADR-009's "Engine
implications" already state: *"resolver reads `plugin_id` + `component_id`;
`CreateSession` is invoked per component; runtime dispatch uses component `kind`."*

However, ADR-009 formally listed **only PipelineSpec** under "Touches frozen
model." Between the spec and the executor sits the **resolved runtime plan** —
`RuntimePipelinePlan.RuntimeNode` (frozen, ADR-001) — which currently carries
only `PluginName` + `PluginKind`. For `component_id` to reach the executor's
`SessionService.CreateSession(component_id)` RPC, the runtime plan must carry
both ids end-to-end:

```
spec.NodeSpec (plugin_id, component_id)   [ADR-009]
  → resolver
  → RuntimeNode                            [FROZEN — this ADR]
  → bridge → runtime model.Node
  → executor.bootstrapNode → CreateSession(component_id)
```

Without this, the component reference is lost at the resolver boundary and
`CreateSession` cannot be invoked per component.

---

## Decision

`RuntimeNode` gains **`PluginID` + `ComponentID`**, replacing `PluginName`:

| Before (frozen) | After (this ADR) |
|------------------|------------------|
| `NodeID`         | `NodeID` |
| `PluginName`     | `PluginID` |
| —                | `ComponentID` (new) |
| `PluginKind`     | `PluginKind` (unchanged — declaration-time role) |
| `Config`         | `Config` |

`PluginKind` stays: it is the declaration-time role (`source|processor|sink`),
mirroring `NodeSpec.Kind`. The Engine confirms at runtime that the discovered
Component's kind matches `PluginKind` (referential-integrity check).

The resolver reads `plugin_id` + `component_id` from `NodeSpec` (ADR-009); the
bridge propagates them to the runtime `model.Node`; the executor passes
`component_id` to `SessionService.CreateSession`.

This is the direct, intended consequence of ADR-009 §"Engine implications" —
this ADR exists only to formally authorize touching the second frozen model
(`RuntimePipelinePlan`), per the project rule that each frozen-model change be
explicitly approved.

---

## Consequences

- **Plan 3 (engine migration)** now has full authorization to migrate the chain:
  resolver + `RuntimeNode` + bridge + `model.Node` + runtime client layer +
  registry + API. `CreateSession` uses the explicit `component_id`.
- **Referential-integrity validation** gains a runtime rule:
  `(plugin_id, component_id)` must resolve to a discovered Component whose kind
  matches the declared `PluginKind` (requires `Discover`, so it is a runtime —
  not declaration-time — check).
- `PluginName` is removed from `RuntimeNode`; all consumers (resolver, bridge,
  executor, `plan.Validate`) migrate in Plan 3. No backward compatibility
  (first release, per ADR-008).
- `RuntimeNode` is the second (and final) frozen model touched by the Component
  Model change set (ADR-009 touched PipelineSpec; this touches the runtime plan).
