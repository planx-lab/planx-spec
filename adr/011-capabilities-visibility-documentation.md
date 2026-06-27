# ADR-011: Capabilities, Visibility & Documentation

**Status:** ACCEPTED
**Date:** 2026-06-27
**Deciders:** Runtime Protocol Architect

---

## Context

Two gaps in the descriptor: (1) capabilities must not be free-form strings (unqueryable), yet must express real runtime/operational traits; (2) there is no way to mark a component's maturity/visibility (Marketplace/Designer need to hide experimental, deprecate, or suppress internal components), and documentation is needed inline so Designer/Marketplace work offline and AI/search can consume it.

---

## Decision

### 1. Structured Capabilities (not free-form, not bitset)

A strongly-typed message. A bitset is semantically opaque; free-form strings are unqueryable. Structured fields are self-documenting and let Engine branch programmatically.

```proto
message Capabilities {
  DeliverySemantics delivery = 1;     // AT_LEAST_ONCE | AT_MOST_ONCE | EXACTLY_ONCE
  bool streaming = 2;                  // unbounded stream (never EOF)
  bool bounded = 3;                    // batch source (reaches EOF)
  bool stateful = 4;
  bool transactional = 5;
  bool supports_checkpoint = 6;        // declared now; Engine may ignore until implemented
  bool supports_resume = 7;
  ParallelismSpec parallelism = 8;
  bool configurable = 9;               // has config fields; false => Designer shows "No configuration required"
}
```

Capability fields are **declarations of what a plugin supports**. Planx today is at-least-once, fast-fail, no checkpoint/retry; Engine honors the implemented subset and ignores the rest. Adding a capability later is additive and non-breaking.

### 2. Visibility & Maturity (component-level)

Lifecycle/visibility metadata lives on the component, separate from runtime *capabilities* (they describe publication status, not behavior):

```proto
message ComponentStatus {
  Maturity maturity = 1;   // STABLE | EXPERIMENTAL | DEPRECATED
  bool hidden = 2;         // do not surface by default in Marketplace/Designer
}
enum Maturity { MATURITY_UNSPECIFIED = 0; STABLE = 1; EXPERIMENTAL = 2; DEPRECATED = 3; }
```

- `EXPERIMENTAL` — visible but flagged.
- `DEPRECATED` — visible with warning; signals removal.
- `hidden` — usable by reference but not listed (internal/white-label components).

### 3. Documentation: embedded summary + examples, large docs external

A `Documentation` message carries a **summary and concrete examples inline** so Designer/Marketplace work offline, search indexes them, and AI can use them directly. Large prose stays external via `homepage`.

```proto
message Documentation {
  string summary = 1;             // one-line purpose (cards, search)
  string homepage = 2;            // README/docs/website URL
  repeated Example examples = 3;  // shown beside the config form
}
message Example {
  string name = 1;
  string description = 2;
  bytes config = 3;               // example config matching the schema
}
```

**Why not URL-only:** a URL-only field forces Designer online to show anything, blocks Marketplace caching, defeats search, and is invisible to AI tooling. A summary + examples in the descriptor solves all four; deep prose still lives outside.

---

## Consequences

- **Engine:** reads `capabilities` for scheduling decisions (e.g. parallelism, delivery semantics); ignores unimplemented fields.
- **Designer:** `configurable=false` → "No configuration required" instead of an empty JSON panel; filters/sorts by `maturity`; hides `hidden`; renders `Documentation.summary` + `examples` beside the form.
- **Marketplace (future):** surfaces components with maturity badges, hides `hidden`, deprecation warnings, search over `summary`/`examples`/`keywords`.
- **Plugin authors:** declare capabilities + status + documentation in code via SDK builders (single source, no external file).
