# Plugin Discovery Protocol — Architecture Proposal

**Status:** Proposal (awaiting approval)
**Date:** 2026-06-27
**Author:** Runtime Protocol Architect review
**Scope:** Redesign the plugin protocol so a plugin binary is **self-describing**; eliminate `manifest.yaml`; unify all plugin metadata into one authoritative source consumed by Engine, SDK, Designer, (future) Marketplace and Admin.
**Backward compatibility:** Not required (pre-first-public-release redesign).

---

## 1. Background & Goal

Today, plugin metadata is fragmented across four places that can drift:

| Location | Holds |
|----------|-------|
| `manifest.yaml` | name, type, version, protocol, binary, max_sessions |
| `internal/plugin/model.PluginDescriptor` (Go) | same fields, duplicated |
| API layer (`GET /plugins`) | a projection of the above |
| Designer assumptions | palette/config rendering with no schema |

There is **no config schema** anywhere, so Designer can only offer a raw JSON editor, and engine has no way to know what a plugin accepts.

**Goal:** ONE authoritative plugin protocol. The binary describes itself. No `manifest.yaml`. Every piece of metadata exists exactly once.

---

## 2. Core Architectural Decisions

### 2.1 The Plugin is the Single Source of Truth (not Engine, not manifest, not Designer)

A plugin binary is the sole authority over its own identity, its components, its config schema, and its capabilities. Everything else (Engine, Designer, Marketplace) **discovers and caches** — never authors.

### 2.2 Plugin = Container of Components (adopted)

A **Plugin** is the packaging/ownership/release unit (one binary). A **Component** is the selectable/runtime unit (a Source, a Processor, or a Sink). One plugin exposes one or more components:

```
mysql-plugin           llm-plugin            hospital-plugin
├── Source             └── Processor          ├── HIS Source
└── Sink                                      ├── LIS Source
                                              ├── EMR Source
                                              ├── Normalize Processor
                                              └── Archive Sink
```

**Designer selects Components, not Plugins.** A pipeline node references a Component.

### 2.3 Self-describing via a Discovery lifecycle stage

A new protocol stage — `Discover()` — precedes any runtime use. The plugin returns its full `PluginDescriptor`. Engine calls it once at load and caches; Designer/Marketplace consume the cached projection. This replaces `manifest.yaml`.

### 2.4 Transport-independent protocol

The protocol is defined as messages + RPC contracts (the semantic lifecycle). gRPC is the **first transport mapping**, not the protocol. Future mappings (stdio like Hashicorp, embedded in-process, wasm host calls) implement the identical lifecycle. The SDK abstracts transport; plugin authors implement SPI and never see transport.

### 2.5 Config stays opaque at runtime

Engine never interprets config bytes. `ConfigSchema` is a **declaration-time** object: Designer uses it for form generation + validation; Engine only transports already-validated opaque bytes; the plugin remains the runtime authority (re-validates on `CreateSession`).

---

## 3. Lifecycle

```
            ┌──────────────────────────────────────────────┐
            │              Engine loads plugin              │
            └──────────────────────┬───────────────────────┘
                                   ▼
              Discover() ──────────► PluginDescriptor  (id, version, components[], schemas, capabilities)
                                   │        (cached; identity frozen for process lifetime)
                                   ▼
              Health()  ──────────► HealthStatus   (optional; readiness before scheduling)
                                   │
                                   ▼
   CreateSession(component_id, config) ──► Session   (config = opaque, already-validated bytes)
                                   │
                ┌──────────────────┼──────────────────┐
                ▼                  ▼                  ▼
        Source: OpenStream    Processor: Process   Sink: WriteBatch
        (stream<Batch>)       (Batch→Batch)        (Batch→Ack)
                │
                ▼
            Ack(Session, window)          (flow control / backpressure; session-scoped)
                                   │
                                   ▼
              CloseSession(Session)
```

**Why these stages and no more:**

- **Discover** — stateless, idempotent, cheap. Returns identity. Required (replaces manifest).
- **Health** — optional readiness probe. Engine decides whether to route traffic. Cheap insurance for remote/wasm transports where process-up ≠ ready.
- **CreateSession / CloseSession** — stateful boundary; config applied here. One session binds exactly one component.
- **Kind-specific execute** (OpenStream / Process / WriteBatch) — semantics differ enough (streaming vs unary) that a unified `Execute(oneof)` would obscure them. Strong-typed per-kind services are clearer.
- **Ack** — session-scoped flow control. Kept general (not source-only) so future windowed processors/sinks can reuse it.

**Explicitly rejected lifecycle methods:**
- `Negotiate(version)` — folded into Discover (`runtime_version` field; Engine checks compatibility). A separate call duplicates responsibility.
- `Shutdown()` — process-level (SIGTERM / connection close). No RPC needed.
- `Validate(config)` — Designer validates via ConfigSchema client-side; plugin re-validates in CreateSession. A separate Validate RPC would split the authority.

---

## 4. PluginDescriptor (canonical)

Two layers: **PluginDescriptor** (binary-level) contains **ComponentDescriptor[]** (component-level).

### 4.1 Field table

| Field | Level | Required | Justification |
|-------|-------|----------|---------------|
| `id` | Plugin | **yes** | Globally-unique kebab-case identity (`mysql`). Used for registration + node reference. |
| `version` | Plugin | **yes** | Semver of the plugin binary. Marketplace/upgrades. |
| `runtime_version` | Plugin | **yes** | Protocol version this binary speaks (`v4`). Engine compatibility gate. |
| `sdk_version` | Plugin | recommended | Which SDK built it. Diagnostics. |
| `display_name` | Plugin | recommended | Human label for Designer/Marketplace. |
| `description` | Plugin | recommended | One-line purpose. |
| `vendor` | Plugin | optional | Org/owner. |
| `license` | Plugin | optional | SPDX identifier. |
| `homepage` | Plugin | optional | Docs/repo URL. |
| `documentation` | Plugin | optional | Longer doc reference (Markdown URL). |
| `icon` | Plugin | optional | Visual identity (URL or data URI). |
| `keywords` | Plugin | optional | Search/Marketplace tags. |
| `components[]` | Plugin | **yes (≥1)** | The components this binary exposes. |
| — `id` | Component | **yes** | Unique within the plugin (`source`, `sink`, `his-source`). Node references are `(plugin_id, component_id)`. |
| — `kind` | Component | **yes** | `SOURCE \| PROCESSOR \| SINK`. Determines which runtime service Engine calls. |
| — `version` | Component | **yes** | Component may version independently of plugin. |
| — `display_name` | Component | recommended | e.g. "MySQL Source". |
| — `description` | Component | recommended | |
| — `config_schema` | Component | yes (may be empty) | `ConfigSchema`. Empty = no config. |
| — `capabilities` | Component | yes (may be empty) | `Capabilities`. |

**Rule:** a field exists in exactly one layer. `kind` lives on Component (a plugin has no single kind). `version` exists on both because they version at different cadences (binary release vs component contract).

---

## 5. ConfigSchema

### 5.1 Decision: proto-native, aligned to a JSON Schema subset (not raw JSON Schema text)

Considered:
- **Raw JSON Schema (draft 2020-12)** — standard, ecosystem of validators + form generators. But full JSON Schema is extremely expressive (`$ref`, `if/then/else`, dynamic composition) — far beyond plugin-config needs, hard to render into predictable forms, hard to reimplement across languages/transport, and Engine/Designer would each need a JSON Schema engine.
- **Custom free-form** — reinvents the wheel, loses shared semantics.
- **Proto-native restricted profile** ✅ — a first-class protocol object whose *semantics* mirror a JSON Schema subset (object + typed properties + required/default/enum/description) but whose *representation* is proto. Strong-typed, transport-independent, trivially rendered into a form, and serializable to JSON Schema text if a tool wants it.

**Why proto-native wins for a multi-year protocol:** the protocol is transport-independent (§2.4). Raw JSON Schema text ties us to one encoding and a heavy parser in every consumer. A proto object is the contract; JSON Schema is one possible projection.

### 5.2 Shape

```proto
message ConfigSchema {
  repeated ConfigField fields = 1;     // top-level object properties
}
message ConfigField {
  string name = 1;                      // "host"
  FieldType type = 2;                   // STRING|INTEGER|NUMBER|BOOLEAN|SECRET|ENUM|OBJECT|ARRAY
  string label = 3;                     // "Host"
  string description = 4;
  bool required = 5;
  Value default = 6;                    // typed default
  repeated string enum_values = 7;      // for ENUM (string-typed)
  string placeholder = 8;
  string example = 9;
  repeated ConfigField properties = 10; // nested fields for OBJECT / ARRAY(item shape)
  Validation validation = 11;           // min/max/pattern (optional)
}
message Validation {
  optional int64 min = 1;  optional int64 max = 2;
  optional string pattern = 3;          // regex
  optional int64 min_length = 4; optional int64 max_length = 5;
}
```

- A dedicated **`SECRET`** type (not just a boolean flag) so Designer renders masked inputs and Engine never logs/echoes these bytes — first-class credential handling.
- `OBJECT`/`ARRAY` nest via `properties`, keeping rendering recursive and predictable (no arbitrary `$ref` graph).
- `example` powers inline help + generated docs (serves the "documentation" goal).

---

## 6. Capabilities (structured)

**Decision: structured message, not free-form strings, not a bitset.**

A bitset is compact but semantically opaque (Designer/Engine must agree on bit positions). Free-form strings are unqueryable. A structured message is self-documenting, strongly-typed, and Engine can branch on programmatically.

```proto
message Capabilities {
  DeliverySemantics delivery = 1;       // AT_LEAST_ONCE | AT_MOST_ONCE | EXACTLY_ONCE
  // Source character
  bool streaming = 2;                   // unbounded stream (never EOF)
  bool bounded = 3;                     // batch source (reaches EOF)
  // Processor character
  bool stateful = 4;
  // Cross-cutting (declared now; Engine may ignore until implemented)
  bool transactional = 5;
  bool supports_checkpoint = 6;
  bool supports_resume = 7;
  ParallelismSpec parallelism = 8;
}
enum DeliverySemantics { DELIVERY_UNSPECIFIED = 0; AT_LEAST_ONCE = 1; AT_MOST_ONCE = 2; EXACTLY_ONCE = 3; }
message ParallelismSpec {
  bool supported = 1;
  string note = 2;                      // e.g. "partition key required"
}
```

**Discipline:** capability fields are **declarations of what a plugin supports**. Planx today is at-least-once, fast-fail, no checkpoint/retry (per guardrails). Engine honors the subset it implements and ignores the rest — but the protocol is ready. Adding a capability later is additive and non-breaking.

---

## 7. Proto Organization

```
planx/plugin/v4/
├── plugin.proto       # PluginService: Discover, Health
│                      #   PluginDescriptor, ComponentDescriptor, Capabilities, HealthStatus
├── schema.proto       # ConfigSchema, ConfigField, Value, Validation, FieldType
├── session.proto      # SessionService: CreateSession, CloseSession, Ack
│                      #   Session, SessionCreateRequest, AckRequest
├── source.proto       # SourceService: OpenStream (server-stream)
├── processor.proto    # ProcessorService: Process (unary)
├── sink.proto         # SinkService: WriteBatch (unary)
└── batch.proto        # Batch, Empty (opaque payload + lineage)
```

### 7.1 Service contracts

```proto
// ---- Discovery & health (all plugins implement) ----
service PluginService {
  rpc Discover(google.protobuf.Empty) returns (PluginDescriptor);
  rpc Health(google.protobuf.Empty) returns (HealthStatus);
}

// ---- Session lifecycle + flow control (all plugins implement) ----
service SessionService {
  rpc CreateSession(SessionCreateRequest) returns (SessionCreateResponse);
  rpc CloseSession(SessionCloseRequest) returns (SessionCloseResponse);
  rpc Ack(AckRequest) returns (AckResponse);            // session-scoped backpressure
}
message SessionCreateRequest {
  string component_id = 1;   // which component within the plugin
  bytes  config = 2;         // opaque, Designer-validated bytes
}
message SessionCreateResponse { string session_id = 1; }

// ---- Kind-specific execution (plugin implements the services its components need) ----
service SourceService {
  rpc OpenStream(StreamOpenRequest) returns (stream Batch);  // EOF closes the stream
}
service ProcessorService {
  rpc Process(Batch) returns (Batch);                         // metadata carries session_id
}
service SinkService {
  rpc WriteBatch(Batch) returns (AckResponse);
}
```

**Why split into these services:** `PluginService` is identity (called once, cached). `SessionService` is state boundary + flow control (called per execution, kind-agnostic). The three kind services carry the genuinely different execution semantics. A plugin implements only the kind services its components declare — a pure `mysql-source` plugin implements `SourceService` but not `SinkService`. Engine learns which to call from `Discover`.

**Session binding:** `component_id` at `CreateSession` binds the session to one component; subsequent kind-service calls carry the session id in metadata (`x-planx-session-id`, unchanged). This is how one binary serves multiple components without per-component RPCs.

---

## 8. Migration Impact

### 8.1 Engine
- **Delete** `manifest.yaml` loading (`internal/plugin/registry/loader.go`) and its `manifest` struct.
- **New load flow:** scan `plugins/<plugin-id>/` → start the binary (convention: `plugins/<plugin-id>/plugin`, a single executable per dir) → connect transport → `Discover()` → cache `PluginDescriptor` → gate on `runtime_version`.
- `model.PluginDescriptor` becomes the proto-generated type (single definition, no duplication).
- `GET /plugins` returns the descriptor projection (components flattened for Designer).
- Runtime dispatch reads cached component `kind` to choose Source/Processor/Sink service.

### 8.2 SDK
- Add a **component registration API**: plugin `main` registers components with their SPI impl + `ConfigSchema` + `Capabilities` + metadata. SDK aggregates these into the `PluginDescriptor` served by `PluginService.Discover`.
- SDK implements `PluginService` + `SessionService` once; wires each registered component to the matching kind service.
- Transport abstraction (`Transport` interface) with a gRPC implementation now; stdio/embedded/wasm later. Plugin SPI stays unchanged across transports.

### 8.3 Designer
- Palette lists **components** grouped by kind; each shows its owning plugin.
- `ConfigPanel` renders a form from `component.config_schema` (replaces the CodeMirror JSON editor; JSON editor remains as an advanced/raw fallback).
- A pipeline node references `(plugin_id, component_id)`.

### 8.4 PipelineSpec (FROZEN model — requires ADR)
- `NodeSpec.plugin` → `NodeSpec { plugin_id, component_id }` (or a single `component` ref `"mysql/source"`).
- This is a frozen-model change → must go through ADR approval before implementation.

### 8.5 Existing example plugins
- `source-hello`, `processor-passthrough`, `sink-stdout` migrate to self-describing: each registers one component + (empty or trivial) schema. They become the reference implementations of the new protocol.

---

## 9. ADRs Required

| ADR | Title | Frozen impact |
|-----|-------|---------------|
| ADR-008 | Plugin Discovery Protocol — self-describing plugins, eliminate manifest | No (new protocol) |
| ADR-009 | Component Model — plugin as container of components | No (new concept) |
| ADR-010 | ConfigSchema — proto-native, JSON-Schema-subset semantics | No |
| ADR-011 | Structured Capabilities | No |
| ADR-012 | PipelineSpec node → component reference | **Yes** (frozen PipelineSpec) |
| ADR-013 | Transport Independence — proto is the contract, transport is a mapping | No |

ADRs 008–011, 013 are additive. **ADR-012 touches the frozen PipelineSpec** and needs explicit approval before any code.

---

## 10. Implementation Roadmap

Each stage is independently verifiable; nothing is built on speculation.

1. **Protocol freeze** — author the `.proto` files (plugin/schema/session/source/processor/sink/batch) + ADRs 008–013. *Verify:* proto compiles; design review.
2. **SDK: discovery + component registration** — `Describe()`/component registry, `ConfigSchema` builder, gRPC transport. *Verify:* a mock plugin returns a descriptor; unit tests.
3. **Engine: Discover-based registry** — delete manifest loader; new load flow; `GET /plugins` projection. *Verify:* engine discovers example plugins, no manifest present.
4. **Example plugins migrate** — source-hello/processor-passthrough/sink-stdout self-describe. *Verify:* end-to-end DAG still runs SUCCEEDED.
5. **Designer: component palette + schema form** — replace JSON editor for schema'd components. *Verify:* build a DAG configuring a component via form.
6. **PipelineSpec migration (ADR-012)** — node → component ref. *Verify:* engine + Designer round-trip; existing tests rewritten.
7. **Real connectors** — mysql (source+sink), csv (source), llm (processor) using the new protocol. *Verify:* each runs a real pipeline.

---

## 11. Single-Source-of-Truth Audit

For each piece of metadata, exactly one authority:

| Metadata | Authority | Consumers (read-only) |
|----------|-----------|----------------------|
| Plugin id/version/vendor/... | Plugin binary (Discover) | Engine, Designer, Marketplace |
| Component list + kinds | Plugin binary (Discover) | Engine (dispatch), Designer (palette) |
| ConfigSchema | Plugin binary (Discover) | Designer (form), submit-time validation |
| Capabilities | Plugin binary (Discover) | Engine (scheduling decisions) |
| Runtime config bytes | PipelineSpec (user-authored, schema-validated) | Engine (transport only), Plugin (runtime authority) |

No row has two possible authors. The design invariant — "if two components could be the source of the same information, the design is wrong" — holds.

---

## 12. Review Outcome (2026-06-27) — Resolved

All four open questions decided, plus two additions raised in review. Frozen in **ADR-008..011**:

1. **Component reference in PipelineSpec** → **`plugin_id` + `component_id` as two separate fields** (NOT a `"plugin/component"` string). A Component is a runtime unit, not a sub-path; this supports component rename and natural Marketplace surfacing. → ADR-009 (authorizes the frozen PipelineSpec change).
2. **ConfigSchema nesting** → **protocol allows UNLIMITED nesting**; Designer supports types in phases (Alpha: object/string/bool/int/enum/secret; Beta: array/nested/map). UI capability never constrains the protocol. → ADR-010.
3. **Health** → **Engine-pulled** (transport-agnostic; Engine owns scheduling/restart/circuit-breaking, plugin does not push). → ADR-008.
4. **Documentation** → **embedded `summary` + `examples`** in the descriptor; large prose external via `homepage`. Offline-usable, cacheable, searchable, AI-ready. → ADR-011.
5. **`ValidateConfig()` lifecycle RPC** *(added in review)* → Designer-initiated live config check (DB / topic / bucket connectivity); Plugin-authoritative, Engine transparently forwards. Strictly stronger than schema-only validation. → ADR-008.
6. **Capability / visibility additions** *(added in review)* → `configurable` (false ⇒ Designer shows "No configuration required"); `maturity` (STABLE/EXPERIMENTAL/DEPRECATED) + `hidden` for Marketplace/Designer. → ADR-011.
