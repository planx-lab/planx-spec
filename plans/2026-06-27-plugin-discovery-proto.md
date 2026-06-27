# Plugin Discovery Protocol — Plan 1: Proto Freeze

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite `planx-proto/v4/` into the self-describing Plugin Discovery Protocol (ADR-008..011): a `PluginService` (Discover/Health/ValidateConfig), a unified `SessionService`, kind-specific runtime services, and the canonical `PluginDescriptor`/`ConfigSchema`/`Capabilities` messages.

**Architecture:** Proto is the transport-independent contract (gRPC is the first mapping). One binary (a Plugin) exposes multiple Components; a pipeline node references a Component via `plugin_id`+`component_id`. Config stays opaque at runtime; `ConfigSchema` is a declaration-time proto object. This plan rewrites the proto only — SDK/Engine/Designer migrate in Plans 2–5.

**Tech Stack:** Protocol Buffers v3, Buf v2 (lint STANDARD, breaking FILE), `buf generate` → Go (`gen/go/planx/plugin/v4`).

**After this plan:** `planx-proto` generates clean Go for the new protocol. **SDK and Engine will NOT compile** until Plans 2–3 migrate them off the removed `SourcePlugin`/`ProcessorPlugin`/`SinkPlugin` services and the old `common.proto` session messages. This is intentional (backward compatibility is not required per the redesign).

**ADRs implemented:** [008](../adr/008-plugin-discovery-protocol.md), [009](../adr/009-component-model.md), [010](../adr/010-config-schema.md), [011](../adr/011-capabilities-visibility-documentation.md).

---

## File Structure

All under `planx-proto/v4/`. Each file has one responsibility:

| File | Responsibility |
|------|----------------|
| `common.proto` | `Empty` + shared enums (`ComponentKind`, `DeliverySemantics`, `Maturity`) |
| `schema.proto` | `ConfigSchema`, `ConfigField`, `ConfigValue`, `FieldValidation`, `FieldType` |
| `batch.proto` | `Batch` (opaque payload) |
| `plugin.proto` | `PluginDescriptor`, `ComponentDescriptor`, `Capabilities`, `ComponentStatus`, `Documentation`, `Example`, `HealthStatus`, `ConfigValidationRequest/Result` + `PluginService` |
| `session.proto` | `SessionCreateRequest/Response`, `SessionCloseRequest/Response`, `AckRequest/Response` + `SessionService` |
| `source.proto` | `StreamOpenRequest` + `SourceService` (OpenStream) |
| `processor.proto` | `ProcessorService` (Process) |
| `sink.proto` | `SinkService` (WriteBatch) |

The old `common.proto` session/stream/batch messages are **moved** into their focused files (`session.proto`, `source.proto`, `batch.proto`); `common.proto` keeps only `Empty` + enums.

---

## Task 1: `common.proto` — Empty + shared enums

**Files:**
- Modify: `planx-proto/v4/common.proto` (rewrite)

- [ ] **Step 1: Rewrite `common.proto`**

```proto
syntax = "proto3";

package planx.plugin.v4;

option go_package = "github.com/planx-lab/planx-proto/gen/go/planx/plugin/v4;pluginv4";

// Empty is the no-argument / no-return placeholder.
message Empty {}

// ComponentKind identifies the runtime role of a Component within a Plugin.
enum ComponentKind {
  COMPONENT_KIND_UNSPECIFIED = 0;
  COMPONENT_KIND_SOURCE = 1;
  COMPONENT_KIND_PROCESSOR = 2;
  COMPONENT_KIND_SINK = 3;
}

// DeliverySemantics declares the plugin's delivery guarantee.
enum DeliverySemantics {
  DELIVERY_SEMANTICS_UNSPECIFIED = 0;
  DELIVERY_SEMANTICS_AT_LEAST_ONCE = 1;
  DELIVERY_SEMANTICS_AT_MOST_ONCE = 2;
  DELIVERY_SEMANTICS_EXACTLY_ONCE = 3;
}

// Maturity declares the publication status of a Component.
enum Maturity {
  MATURITY_UNSPECIFIED = 0;
  MATURITY_STABLE = 1;
  MATURITY_EXPERIMENTAL = 2;
  MATURITY_DEPRECATED = 3;
}
```

- [ ] **Step 2: Verify buf lint passes for this file**

Run: `cd planx-proto && buf lint v4/common.proto`
Expected: no output (pass). If `ENUM_VALUE_PREFIX` errors appear, confirm each enum value is prefixed with its enum name (above it is).

- [ ] **Step 3: Commit**

```bash
cd planx-proto
git add v4/common.proto
git commit -m "proto(v4): common.proto — Empty + shared enums (ADR-008/009/011)"
```

---

## Task 2: `schema.proto` — ConfigSchema

**Files:**
- Create: `planx-proto/v4/schema.proto`

- [ ] **Step 1: Write `schema.proto`**

```proto
syntax = "proto3";

package planx.plugin.v4;

option go_package = "github.com/planx-lab/planx-proto/gen/go/planx/plugin/v4;pluginv4";

// ConfigSchema declares the shape of a Component's config. It is a
// declaration-time object: Designer renders a form from it; Engine never
// interprets config bytes at runtime. Semantics mirror a JSON-Schema subset.
// Nesting is UNLIMITED in the protocol (ADR-010); Designer support is phased.
message ConfigSchema {
  repeated ConfigField fields = 1;
}

// FieldType is the value type of a config field. SECRET is first-class so
// Designer masks it and Engine/Plugin never log it.
enum FieldType {
  FIELD_TYPE_UNSPECIFIED = 0;
  FIELD_TYPE_STRING = 1;
  FIELD_TYPE_INTEGER = 2;
  FIELD_TYPE_NUMBER = 3;
  FIELD_TYPE_BOOLEAN = 4;
  FIELD_TYPE_SECRET = 5;
  FIELD_TYPE_ENUM = 6;
  FIELD_TYPE_OBJECT = 7;
  FIELD_TYPE_ARRAY = 8;
}

message ConfigField {
  string name = 1;
  FieldType type = 2;
  string label = 3;
  string description = 4;
  bool required = 5;
  ConfigValue default = 6;
  repeated string enum_values = 7;       // populated when type == ENUM
  string placeholder = 8;
  string example = 9;
  repeated ConfigField properties = 10;  // nested fields (OBJECT / ARRAY item shape)
  FieldValidation validation = 11;
}

// FieldValidation captures simple scalar/range constraints.
message FieldValidation {
  optional int64 min = 1;
  optional int64 max = 2;
  optional string pattern = 3;           // regex
  optional int64 min_length = 4;
  optional int64 max_length = 5;
}

// ConfigValue is a typed scalar used for field defaults (oneof).
message ConfigValue {
  oneof kind {
    string string_value = 1;
    int64 int_value = 2;
    double double_value = 3;
    bool bool_value = 4;
  }
}
```

- [ ] **Step 2: Verify buf lint**

Run: `cd planx-proto && buf lint v4/schema.proto`
Expected: no output (pass). (`proto3` supports `optional` scalar fields since buf/protoc 3.15+; the project is on Go 1.25 / recent protoc, so this is fine.)

- [ ] **Step 3: Commit**

```bash
cd planx-proto
git add v4/schema.proto
git commit -m "proto(v4): schema.proto — ConfigSchema/ConfigField (ADR-010)"
```

---

## Task 3: `batch.proto` — Batch

**Files:**
- Create: `planx-proto/v4/batch.proto`

- [ ] **Step 1: Write `batch.proto`**

```proto
syntax = "proto3";

package planx.plugin.v4;

option go_package = "github.com/planx-lab/planx-proto/gen/go/planx/plugin/v4;pluginv4";

// Batch carries one unit of data. Payload is opaque and codec-defined;
// the Engine never inspects it. Lineage is optional runtime metadata
// (BatchID etc., frozen in ADR-005); left as opaque bytes here so the
// discovery protocol does not depend on runtime internals.
message Batch {
  bytes payload = 1;
  bytes lineage = 2;  // optional, opaque runtime lineage (ADB-005)
}
```

- [ ] **Step 2: Verify buf lint**

Run: `cd planx-proto && buf lint v4/batch.proto`
Expected: no output (pass).

- [ ] **Step 3: Commit**

```bash
cd planx-proto
git add v4/batch.proto
git commit -m "proto(v4): batch.proto — Batch (opaque payload + lineage)"
```

---

## Task 4: `plugin.proto` — Descriptor + PluginService

**Files:**
- Create: `planx-proto/v4/plugin.proto`

- [ ] **Step 1: Write `plugin.proto`**

```proto
syntax = "proto3";

package planx.plugin.v4;

option go_package = "github.com/planx-lab/planx-proto/gen/go/planx/plugin/v4;pluginv4";

import "common.proto";
import "schema.proto";

// PluginDescriptor is the canonical, self-described identity of a Plugin
// binary, returned by Discover(). The Plugin is the single source of truth.
message PluginDescriptor {
  string id = 1;                       // globally-unique kebab-case, e.g. "mysql"
  string version = 2;                  // semver of the binary
  string runtime_version = 3;          // protocol version spoken, e.g. "v4"
  string sdk_version = 4;              // SDK that built it (diagnostics)
  string display_name = 5;
  string description = 6;
  string vendor = 7;
  string license = 8;                  // SPDX identifier
  string homepage = 9;
  string icon = 10;                    // URL or data URI
  repeated string keywords = 11;
  Documentation documentation = 12;
  repeated ComponentDescriptor components = 13;  // >= 1
}

// ComponentDescriptor describes one selectable Component within a Plugin.
message ComponentDescriptor {
  string id = 1;                       // unique within the plugin, e.g. "source"
  ComponentKind kind = 2;              // SOURCE | PROCESSOR | SINK
  string version = 3;                  // component may version independently
  string display_name = 4;
  string description = 5;
  ConfigSchema config_schema = 6;      // may be empty (no config)
  Capabilities capabilities = 7;
  ComponentStatus status = 8;          // maturity / visibility
}

// Capabilities declares what a Component supports. Structured, not free-form.
// Fields the Engine does not yet implement are ignored (additive, non-breaking).
message Capabilities {
  DeliverySemantics delivery = 1;
  bool streaming = 2;                  // unbounded stream (never EOF)
  bool bounded = 3;                    // batch source (reaches EOF)
  bool stateful = 4;
  bool transactional = 5;
  bool supports_checkpoint = 6;
  bool supports_resume = 7;
  ParallelismSpec parallelism = 8;
  bool configurable = 9;               // false => Designer shows "No configuration required"
}

message ParallelismSpec {
  bool supported = 1;
  string note = 2;                     // e.g. "partition key required"
}

// ComponentStatus declares publication status / visibility (Marketplace/Designer).
message ComponentStatus {
  Maturity maturity = 1;
  bool hidden = 2;                     // do not surface by default
}

// Documentation carries an embedded summary + examples (offline-usable,
// searchable, AI-ready). Large prose stays external via homepage.
message Documentation {
  string summary = 1;
  string homepage = 2;
  repeated Example examples = 3;
}

message Example {
  string name = 1;
  string description = 2;
  bytes config = 3;                    // example config matching the schema
}

// HealthStatus is returned by Health() (Engine-pulled readiness probe).
message HealthStatus {
  enum State {
    STATE_UNSPECIFIED = 0;
    STATE_READY = 1;
    STATE_NOT_READY = 2;
  }
  State state = 1;
  string message = 2;
}

// ConfigValidationRequest is sent by Designer (via Engine) to ValidateConfig()
// for a live config check (connectivity, resource existence). The Plugin is
// authoritative; Engine only forwards.
message ConfigValidationRequest {
  string component_id = 1;
  bytes config = 2;                    // opaque, Designer-validated bytes
}

message ConfigValidationResult {
  bool ok = 1;
  string message = 2;
  repeated ValidationDetail details = 3;
}

message ValidationDetail {
  string key = 1;          // e.g. "latency", "db_version"
  string value = 2;
}

// PluginService is the discovery + metadata surface. Every Plugin implements it.
// Stateless methods, called once at load and cached.
service PluginService {
  rpc Discover(Empty) returns (PluginDescriptor);
  rpc Health(Empty) returns (HealthStatus);
  rpc ValidateConfig(ConfigValidationRequest) returns (ConfigValidationResult);
}
```

- [ ] **Step 2: Verify buf lint**

Run: `cd planx-proto && buf lint v4/plugin.proto`
Expected: no output (pass). Nested enum `HealthStatus.State` uses `STATE_` prefix, satisfying `ENUM_VALUE_PREFIX` within its scope.

- [ ] **Step 3: Commit**

```bash
cd planx-proto
git add v4/plugin.proto
git commit -m "proto(v4): plugin.proto — Descriptor + PluginService (ADR-008/009/011)"
```

---

## Task 5: `session.proto` — SessionService

**Files:**
- Create: `planx-proto/v4/session.proto`

- [ ] **Step 1: Write `session.proto`**

```proto
syntax = "proto3";

package planx.plugin.v4;

option go_package = "github.com/planx-lab/planx-proto/gen/go/planx/plugin/v4;pluginv4";

// SessionCreateRequest binds a session to one Component and applies config.
// config is opaque bytes, already Designer/schema-validated; the Plugin
// remains the runtime authority and re-validates here.
message SessionCreateRequest {
  string component_id = 1;
  bytes config = 2;
}

message SessionCreateResponse {
  string session_id = 1;
}

message SessionCloseRequest {
  string session_id = 1;
}

message SessionCloseResponse {}

// AckRequest is session-scoped flow control (backpressure). Today used by
// Source streams; kept general so future windowed processors/sinks reuse it.
message AckRequest {
  string session_id = 1;
  int32 processed = 2;
  int32 new_window = 3;
}

message AckResponse {}

// SessionService owns the stateful boundary + flow control. Every Plugin
// implements it; kind-specific execution lives in Source/Processor/SinkService.
service SessionService {
  rpc CreateSession(SessionCreateRequest) returns (SessionCreateResponse);
  rpc CloseSession(SessionCloseRequest) returns (SessionCloseResponse);
  rpc Ack(AckRequest) returns (AckResponse);
}
```

- [ ] **Step 2: Verify buf lint**

Run: `cd planx-proto && buf lint v4/session.proto`
Expected: no output (pass).

- [ ] **Step 3: Commit**

```bash
cd planx-proto
git add v4/session.proto
git commit -m "proto(v4): session.proto — SessionService (ADR-008/009)"
```

---

## Task 6: `source.proto` — SourceService

**Files:**
- Create: `planx-proto/v4/source.proto`

- [ ] **Step 1: Write `source.proto`**

```proto
syntax = "proto3";

package planx.plugin.v4;

option go_package = "github.com/planx-lab/planx-proto/gen/go/planx/plugin/v4;pluginv4";

import "batch.proto";

// StreamOpenRequest opens a server-streaming read for an already-created
// Source session. A clean stream end (plugin returns / gRPC closes) is the
// exhaustion signal; the Engine maps it to CodeEOF -> pipeline SUCCEEDED.
message StreamOpenRequest {
  string session_id = 1;
  int32 initial_window = 2;
}

// SourceService is implemented by any Plugin exposing a SOURCE component.
service SourceService {
  rpc OpenStream(StreamOpenRequest) returns (stream Batch);
}
```

- [ ] **Step 2: Verify buf lint**

Run: `cd planx-proto && buf lint v4/source.proto`
Expected: no output (pass).

- [ ] **Step 3: Commit**

```bash
cd planx-proto
git add v4/source.proto
git commit -m "proto(v4): source.proto — SourceService.OpenStream (ADR-008)"
```

---

## Task 7: `processor.proto` — ProcessorService

**Files:**
- Create: `planx-proto/v4/processor.proto`

- [ ] **Step 1: Write `processor.proto`**

```proto
syntax = "proto3";

package planx.plugin.v4;

option go_package = "github.com/planx-lab/planx-proto/gen/go/planx/plugin/v4;pluginv4";

import "batch.proto";

// ProcessorService is implemented by any Plugin exposing a PROCESSOR component.
// The session id travels in gRPC metadata (x-planx-session-id); Process is
// a stateless per-batch transform.
service ProcessorService {
  rpc Process(Batch) returns (Batch);
}
```

- [ ] **Step 2: Verify buf lint**

Run: `cd planx-proto && buf lint v4/processor.proto`
Expected: no output (pass).

- [ ] **Step 3: Commit**

```bash
cd planx-proto
git add v4/processor.proto
git commit -m "proto(v4): processor.proto — ProcessorService.Process (ADR-008)"
```

---

## Task 8: `sink.proto` — SinkService

**Files:**
- Create: `planx-proto/v4/sink.proto`

- [ ] **Step 1: Write `sink.proto`**

```proto
syntax = "proto3";

package planx.plugin.v4;

option go_package = "github.com/planx-lab/planx-proto/gen/go/planx/plugin/v4;pluginv4";

import "batch.proto";
import "session.proto";

// SinkService is implemented by any Plugin exposing a SINK component.
// WriteBatch acknowledges per batch (session-scoped). The session id travels
// in gRPC metadata (x-planx-session-id).
service SinkService {
  rpc WriteBatch(Batch) returns (AckResponse);
}
```

- [ ] **Step 2: Verify buf lint**

Run: `cd planx-proto && buf lint v4/sink.proto`
Expected: no output (pass).

- [ ] **Step 3: Commit**

```bash
cd planx-proto
git add v4/sink.proto
git commit -m "proto(v4): sink.proto — SinkService.WriteBatch (ADR-008)"
```

---

## Task 9: Delete obsolete proto, regenerate, verify build

The old `common.proto` session/stream/batch messages were moved out in Tasks 1–8. The old per-kind services (`SourcePlugin`/`ProcessorPlugin`/`SinkPlugin` in the original `source.proto`/`processor.proto`/`sink.proto`) are now replaced by `SourceService`/`ProcessorService`/`SinkService`. The old file *contents* were overwritten in Tasks 6–8 (same paths). Nothing else to delete — confirm the module is self-consistent.

**Files:**
- Verify: whole `planx-proto/v4/` module

- [ ] **Step 1: Lint the whole module**

Run: `cd planx-proto && buf lint`
Expected: no output (pass).

- [ ] **Step 2: Regenerate all Go code**

Run: `cd planx-proto && buf generate`
Expected: no output; `gen/go/planx/plugin/v4/` contains `plugin.pb.go`, `schema.pb.go`, `session.pb.go`, `source.pb.go` (with `_grpc.pb.go`), `processor.pb.go` (`_grpc.pb.go`), `sink.pb.go` (`_grpc.pb.go`), `batch.pb.go`, `common.pb.go`.

- [ ] **Step 3: Verify the generated package compiles**

Run: `cd planx-proto && go build ./gen/go/planx/plugin/v4/...`
Expected: no output (pass). This proves the proto contract is internally consistent.

- [ ] **Step 4: Confirm there are no leftover references to removed symbols**

Run: `cd planx-proto && grep -rn "SourcePlugin\|ProcessorPlugin\|SinkPlugin" v4/ || echo "clean"`
Expected: `clean`.

- [ ] **Step 5: Commit generated code**

```bash
cd planx-proto
git add gen/ v4/
git commit -m "proto(v4): regenerate — Plugin Discovery Protocol (ADR-008..011)

Plan 1 complete. planx-proto speaks the new protocol.
SDK/Engine migrate in Plans 2-3."
```

---

## Out of scope (later plans)

- **Plan 2 (SDK):** component registry + `PluginService.Discover`/`Health`/`ValidateConfig` implementation; `Transport` abstraction (gRPC now).
- **Plan 3 (Engine):** delete `manifest.yaml` loader; Discover-based registry; `GET /plugins` projection; `ValidateConfig` forwarding endpoint.
- **Plan 4:** migrate example plugins (source-hello / processor-passthrough / sink-stdout) to self-describe.
- **Plan 5 (Designer):** component palette + schema-driven config form.
- **Plan 6:** PipelineSpec node → `plugin_id`+`component_id` (frozen model, ADR-009).
- **Plan 7:** real connectors (mysql/csv/llm).
