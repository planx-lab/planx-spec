# Planx Architecture – v4.0

## 0. Purpose

Planx 4.0 is a **high-performance, multi-tenant, plugin-first iPaaS**.

This document defines the **non-negotiable architectural truths** of Planx.
Any implementation, human or AI-generated, MUST comply.

---

## 1. Core Architecture Principles

1. **Plugin-First**
   - All data ingress, egress, and transformation logic lives in plugins.
   - The engine is generic and domain-agnostic.

2. **Process Isolation**
   - Every plugin runs as an independent OS process.
   - No in-process plugins. No shared memory.

3. **SDK-Managed Runtime**
   - Plugins do NOT implement gRPC, sessions, concurrency, or flow control.
   - All runtime complexity lives in SDKs.

4. **Batch-Only Transport**
   - All data moves as `Batch`.
   - No single-record APIs.

5. **Zero-Copy Payload**
   - Payload is opaque `[]byte`.
   - Engine never parses payloads.

6. **Multi-Tenant by Default**
   - A single plugin process may serve multiple tenants concurrently.
   - Tenant isolation is enforced via sessions.

---

## 2. System Components

### 2.1 planx-engine
Responsibilities:
- Pipeline orchestration (DAG)
- Plugin process lifecycle
- Session orchestration
- Flow-control decisions
- YAML / TOML pipeline parsing

MUST NOT:
- Import plugin code
- Parse payloads
- Contain business logic

---

### 2.2 planx-sdk-*

Responsibilities:
- gRPC server implementation
- Session registry
- Flow control enforcement (window blocking)
- Batch packing / unpacking
- Observability (metrics, logs, tracing)

SDK is the **only runtime authority**.

---

### 2.3 Plugins (Source / Processor / Sink)

Characteristics:
- Standalone executable artifacts
- Use SDK only
- Implement minimal SPI interfaces
- Contain business logic only

Plugins MUST NOT:
- Implement gRPC services
- Manage sessions
- Control concurrency
- Access engine internals

---

## 3. Plugin Discovery Model

Planx 4.0 uses a **Manifest-First Plugin Model**.

- Engine discovers plugins via filesystem directories.
- Each plugin directory contains:
  - One executable artifact
  - One `manifest.yaml`

Runtime handshake exists internally but is NOT the discovery mechanism.

---

## 4. Polyglot Strategy

- Plugins MAY be written in Go, Rust, or Python.
- SDKs provide identical semantics across languages.
- gRPC protocol is the only contract.

---

## 5. Architectural Non-Goals

- No dynamic linking (`.so`, `dlopen`)
- No in-process plugins
- No implicit discovery
- No engine-specific plugin assumptions

---

## 6. Authority

If any document, code, or AI output conflicts with this file,
**this file takes precedence**.
