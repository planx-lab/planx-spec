# ADR-008: Plugin Discovery Protocol

**Status:** ACCEPTED
**Date:** 2026-06-27
**Deciders:** Runtime Protocol Architect
**Supersedes:** manifest.yaml-based discovery (loader.go)

---

## Context

Plugin metadata is fragmented across `manifest.yaml`, an internal Go `PluginDescriptor`, the API layer, and Designer assumptions. There is no config schema. The four sources can drift, and no component can fully trust another.

Before first public release we want **one authoritative plugin protocol** where the binary describes itself, consumed identically by Engine, SDK, Designer, and a future Marketplace/Admin.

---

## Decision

Adopt a **self-describing plugin protocol** with a fixed lifecycle. The plugin binary is the single source of truth for its own identity; manifest.yaml is eliminated.

### Lifecycle (final)

```
Discover() → PluginDescriptor          (identity; cached; replaces manifest)
Health()    → HealthStatus             (Engine-pulled readiness probe)
ValidateConfig(component_id, config) → ValidationResult
                                       (Designer-initiated online check; Engine transparently forwards; Plugin authoritative)
CreateSession(component_id, config) → Session
  ├─ Source:  OpenStream(Session) → stream<Batch>; Ack(Session, window)
  ├─ Processor: Process(Session, Batch) → Batch
  └─ Sink:    WriteBatch(Session, Batch) → Ack
CloseSession(Session)
```

### Key rules

1. **Discover** is stateless, idempotent, called once at load and cached. It returns the canonical `PluginDescriptor` (identity + components + schemas + capabilities). This replaces `manifest.yaml`.
2. **Health is Engine-pulled.** Pull works across all transports (gRPC/stdio/embedded/wasm); push would be transport-dependent. Future scheduling, restart, and circuit-breaking are Engine responsibilities — the plugin does not proactively report.
3. **`ValidateConfig` is a first-class lifecycle RPC.** After a user fills the config form, Designer requests a live check (can the DB connect? does the Kafka topic / S3 bucket exist?). Validation **always belongs to the Plugin**; Engine only forwards the request and the boolean result — it never interprets config. This is strictly stronger than JSON-Schema-only validation. Plugins with nothing to validate return `ok`.
4. **Config stays opaque at runtime.** Engine transports already-validated `bytes`; the Plugin is the runtime authority and re-validates in `CreateSession`.
5. **Transport-independent.** The protocol is messages + RPC contracts. gRPC is the first mapping; future mappings (stdio / embedded / wasm) implement the identical lifecycle. The SDK abstracts transport; plugin authors implement SPI and never see transport.

### Explicitly rejected

- `Negotiate(version)` — folded into `Discover` (`runtime_version` field).
- `Shutdown()` — process-level (SIGTERM / connection close).

### Binary convention (no manifest)

Engine scans `plugins/<plugin-id>/`; the single executable is `plugins/<plugin-id>/plugin`. Engine starts it, connects, calls `Discover`. No external metadata file.

---

## Consequences

- **Engine:** delete `internal/plugin/registry/loader.go` and its `manifest` struct; rewrite load flow around `Discover`; gate on `runtime_version`; add `ValidateConfig` forwarding endpoint.
- **SDK:** implement `PluginService` (Discover/Health/ValidateConfig) + `SessionService` + a component registry; add a `Transport` abstraction with a gRPC implementation.
- **Designer:** consumes the cached descriptor; calls `ValidateConfig` via the Engine API for live config checks.
- **Proto:** `planx/plugin/v4/` gains `PluginService` and the discovery messages (see ADR-009/010/011).
- Every piece of plugin metadata now has exactly one author (the binary).
