# ADR-010: ConfigSchema

**Status:** ACCEPTED
**Date:** 2026-06-27
**Deciders:** Runtime Protocol Architect

---

## Context

There is currently no config schema anywhere. Designer offers only a raw JSON editor; users must memorize field names; there is no validation, no defaults, no credential handling, and no documentation of what a plugin accepts.

We need a schema that (a) generates Designer forms, (b) validates configs, (c) documents connectors, and (d) carries defaults/constraints — while keeping config **opaque at runtime** (Engine never interprets it).

---

## Decision

### 1. Proto-native, aligned to a JSON-Schema subset

`ConfigSchema` is a first-class **protocol object** (proto message), not raw JSON Schema text. Its *semantics* mirror a JSON-Schema subset (object + typed properties + required/default/enum/description); its *representation* is proto.

**Why not raw JSON Schema:** full draft 2020-12 is far beyond plugin-config needs (`$ref`, `if/then/else`, dynamic composition), forces every consumer to ship a JSON-Schema engine, renders unpredictably into forms, and is transport-coupled. A proto object is the cross-language, transport-independent contract; JSON Schema text can be a *projection* of it when a tool wants one.

### 2. Protocol allows UNLIMITED nesting; Designer supports in phases

`OBJECT`/`ARRAY` nest via `properties` with **no depth limit in the protocol**. UI capability is a separate concern:

- **Protocol:** unlimited `OBJECT`/`ARRAY` nesting.
- **Designer Alpha:** `object`, `string`, `bool`, `int`, `enum`, `secret`.
- **Designer Beta:** adds `array`, nested `object`, `map`.

**Principle: the protocol is never constrained by what a UI can render today.** A field type the current Designer cannot render is still valid on the wire; Designer degrades gracefully (e.g. falls back to raw JSON for that field).

### 3. Dedicated SECRET type

A `SECRET` field type (not a boolean flag) so Designer renders masked inputs and Engine/Plugin **never log or echo** these bytes — credential handling is first-class.

### 4. Shape

```proto
message ConfigSchema {
  repeated ConfigField fields = 1;
}
message ConfigField {
  string name = 1;
  FieldType type = 2;       // STRING|INTEGER|NUMBER|BOOLEAN|SECRET|ENUM|OBJECT|ARRAY
  string label = 3;
  string description = 4;
  bool required = 5;
  Value default = 6;
  repeated string enum_values = 7;   // for ENUM
  string placeholder = 8;
  string example = 9;
  repeated ConfigField properties = 10;  // nested (OBJECT / ARRAY item shape)
  Validation validation = 11;
}
message Validation {
  optional int64 min = 1; optional int64 max = 2;
  optional string pattern = 3;
  optional int64 min_length = 4; optional int64 max_length = 5;
}
```

---

## Consequences

- **Designer:** schema-driven form rendering replaces the JSON editor as the primary UX (JSON editor retained as advanced/raw fallback); phased type support per above.
- **Submit-time validation** uses the schema client-side before `CreateSession`.
- **`ValidateConfig` (ADR-008)** complements schema validation with live checks the schema cannot express (connectivity, resource existence).
- **SDK:** provides a `ConfigSchema`/`ConfigField` builder so plugin authors declare schemas in code.
- Empty `ConfigSchema` (no fields) is valid → a component with `capabilities.configurable = false` renders "No configuration required" (see ADR-011).
