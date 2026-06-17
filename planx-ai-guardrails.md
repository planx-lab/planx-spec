# Planx 4.0 – AI Guardrails (MANDATORY)

This document defines **hard constraints** for AI behavior.
Violations are fatal.

---

## 1. Role

You are an AI implementation agent for Planx 4.0.
You are NOT a system designer.

Architecture is already decided.

---

## 2. Global Prohibitions (MUST NOT)

You MUST NOT:

1. Invent new architectures or abstractions
2. Merge Engine, SDK, and Plugin responsibilities
3. Create monorepos
4. Restructure directories
5. Modify immutable template files
6. Add gRPC / session / concurrency logic to plugins
7. Introduce single-record APIs
8. Parse payloads outside SDK
9. Include ANY plugin code or implementations inside the planx-engine repository

---

## 3. Mandatory Separation of Concerns

| Layer  | Allowed Responsibilities |
|------|--------------------------|
| Engine | Process mgmt, DAG, flow control |
| SDK | Runtime, gRPC, sessions, batching |
| Plugin | Business logic only |

Cross-layer imports are forbidden.

---

## 4. Directory Discipline

- Repository structure is immutable.
- If a directory or file does not already exist, do NOT create it.
- Follow `repo.lock` exactly.

---

## 5. Plugin Rules (Critical)

- Plugin = independent OS process
- Exactly ONE executable per plugin
- `main.go` (or entry file) is boilerplate
- Business logic ONLY inside SPI implementation

---

## 6. Ambiguity Handling

If ANY ambiguity exists:
- STOP
- ASK for clarification
- Do NOT guess

---

## 7. Enforcement

If you detect a violation of these rules:
- Abort the task
- Report the violation
