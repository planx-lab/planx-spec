# Planx Specification & Governance Repository

This repository contains the **canonical, versioned specifications** for the Planx system.

It is the single source of truth for:
- Declarative pipeline specifications
- Cross-repository architectural contracts
- AI behavior constraints
- Repository layout rules

No runtime code lives here.

---

## Purpose

Planx is a multi-repository system composed of:
- A runtime engine
- SDKs
- Plugin templates
- Protocol definitions

To prevent drift, ambiguity, and undocumented behavior, **all normative definitions are centralized in this repository**.

Any document here is considered:
- **Normative**
- **Authoritative**
- **Binding** across all Planx repositories

---

## What Lives Here

### Canonical Specifications
- `planx-spec.md`  
  Defines the canonical Planx Pipeline Specification (v4).

### AI Governance
- `AI_CONTRACT.md`  
  Defines mandatory AI behavior rules for contributors and AI IDEs.
- `planx-ai-guardrails.md`  
  Additional system-level AI safety and scope constraints.

### Architecture
- `planx-architecture.md`  
  High-level architectural invariants and layering rules.

### Repository Layout
- `planx-repo-layout.lock`  
  Defines allowed repository structure and cross-layer import constraints.

---

## What Does NOT Live Here

This repository intentionally does NOT contain:

- Runtime execution logic
- Engine implementation details
- SDK code
- Plugin implementations
- Generated artifacts
- Test pipelines

Those belong in their respective repositories.

---

## Authority Model

- All other Planx repositories **MUST conform** to the documents in this repo.
- In case of conflict:
  **planx-spec always wins.**
- Changes to this repository should be rare and deliberate.

---

## AI Rule Discovery

AI IDEs (e.g., antigravity) discover rules via workspace-level `.agent/` directories.

For this reason:
- Canonical rules live here
- Read-only mirror files may exist in developer workspaces
- Canonical content MUST always be updated here first

---

## Status

This repository reflects the **frozen Planx 4.0 specification baseline**.

Any changes beyond clarification require explicit versioning and review.

## Specification Status

> [!IMPORTANT]
> **Planx 4.0 specification is frozen. Runtime work resumes.**
> No behavioral or structural changes are allowed without a new specification version.

---
