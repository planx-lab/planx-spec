# ADR-009: Component Model & PipelineSpec Reference

**Status:** ACCEPTED
**Date:** 2026-06-27
**Deciders:** Runtime Protocol Architect
**Touches frozen model:** YES — PipelineSpec `NodeSpec`

---

## Context

The current model is "one plugin = one component = one kind (source|processor|sink)". This blocks realistic connectors:

- `mysql` is both a **Source** and a **Sink**
- `hospital` could expose **HIS/LIS/EMR Sources**, a **Normalize Processor**, and an **Archive Sink** in one binary

It also forces a single `plugin` string on every node, which cannot distinguish which component of a multi-component plugin a node uses.

---

## Decision

### 1. Plugin is a Package; Component is a Runtime Unit

- **Plugin** = the packaging/ownership/release unit (one binary, one `PluginDescriptor`).
- **Component** = the selectable/runtime unit (exactly one kind: `SOURCE | PROCESSOR | SINK`). One plugin exposes **one or more** components via `PluginDescriptor.components[]`.
- **Designer selects Components, not Plugins.** A pipeline node references a Component.

```
mysql-plugin           llm-plugin            hospital-plugin
├── Source             └── Processor          ├── HIS Source
└── Sink                                      ├── LIS Source
                                              ├── EMR Source
                                              ├── Normalize Processor
                                              └── Archive Sink
```

### 2. PipelineSpec references a Component with TWO fields

A node carries **`plugin_id` and `component_id` as separate fields** — never a composite `"plugin/component"` string.

```yaml
spec:
  nodes:
    - id: src-1
      kind: source
      plugin_id: mysql
      component_id: source
      config: { ... }
    - id: snk-1
      kind: sink
      plugin_id: mysql
      component_id: sink
      config: { ... }
    - id: norm-1
      kind: processor
      plugin_id: hospital
      component_id: normalize
      config: { ... }
```

**Why two fields, not a string:**

1. A Component is not a sub-path of a Plugin — it is a first-class runtime unit. Splitting `"mysql/source"` in the UI is a smell.
2. Marketplace surfaces a Plugin's components naturally (Source ✓ / Sink ✓ / CDC ✓); Designer drills Plugin → Component. No string parsing.
3. **A component can be renamed** (`source` → `read`) without rewriting every pipeline that references it — only the descriptor changes.
4. Components carry their own metadata (displayName/icon/tags/capabilities); separate fields let that metadata live on the component cleanly.

### 3. Frozen-model change

`NodeSpec.plugin` (single string) → `NodeSpec { plugin_id, component_id }`. This is a frozen PipelineSpec change, authorized by this ADR.

---

## Consequences

- **Engine:** resolver reads `plugin_id` + `component_id`; `CreateSession` is invoked per component; runtime dispatch uses component `kind`.
- **Designer:** palette lists components grouped by kind, each annotated with its owning plugin; node creation captures both ids.
- **Marketplace (future):** a plugin page lists its components as first-class selectable items.
- **Referential integrity** validation gains a rule: `(plugin_id, component_id)` must resolve to a discovered component of matching `kind`.
- Engine/SDK/Designer tests referencing the old single `plugin` field are rewritten.
