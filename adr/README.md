# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) for Planx 4.0.

## Index

| ADR | Title | Status |
|-----|-------|--------|
| [001](001-static-dag-pipeline.md) | Replace Linear Pipeline with Static DAG | ACCEPTED |
| [002](002-merge-batch-alignment.md) | Batch Alignment at Merge Nodes — FIFO Pairing | SUPERSEDED by ADR-005 |
| [003](003-broadcast-copy-semantics.md) | Broadcast Copy Semantics — Independent Copy | SUPERSEDED by ADR-005 |
| [004](004-runtime-foundation.md) | Runtime Foundation | ACCEPTED |
| [005](005-batch-identity-and-merge.md) | Batch Identity and Merge | ACCEPTED |
| [006](006-stage5b-runtime-architecture.md) | Stage 5B Runtime Architecture Freeze | ACCEPTED |
| [007](007-edge-channel-contract.md) | Edge Channel Contract | ACCEPTED |
| [008](008-plugin-discovery-protocol.md) | Plugin Discovery Protocol | ACCEPTED |
| [009](009-component-model.md) | Component Model & PipelineSpec Reference | ACCEPTED |
| [010](010-config-schema.md) | ConfigSchema | ACCEPTED |
| [011](011-capabilities-visibility-documentation.md) | Capabilities, Visibility & Documentation | ACCEPTED |

## Reading Order

For implementers, read in this order:

1. **ADR-001** — Why DAG? What topologies are supported?
2. **ADR-004** — How does the runtime execute? (execution, parallelism, delivery, failure, tenant, backpressure, ordering)
3. **ADR-005** — How does data flow through the DAG? (BatchID, broadcast, merge, ACK)
4. **ADR-006** — How is the runtime structured to deliver ADR-004/005? (Single Runtime, RuntimeBatch, MergeState, Scheduler, InFlight) — **read before Stage 5C implementation**
5. **ADR-007** — Edge channel element type, capacity, ownership — **read before writing any edge-channel code**
6. **ADR-008..011** — Plugin Discovery Protocol (self-describing plugins, Component model, ConfigSchema, Capabilities/Visibility/Documentation) — **read before implementing the new plugin protocol**

## Status Legend

| Status | Meaning |
|--------|---------|
| PROPOSED | Under discussion, not yet accepted |
| ACCEPTED | Approved and binding |
| SUPERSEDED | Replaced by a newer ADR; retained for historical record |
| DEPRECATED | No longer applicable |

## Format

Each ADR follows the format:

```
# ADR-NNN: Title

**Status:** ACCEPTED | PROPOSED | SUPERSEDED | DEPRECATED
**Date:** YYYY-MM-DD
**Deciders:** ...

## Context
## Decision
## Consequences
## References
```
