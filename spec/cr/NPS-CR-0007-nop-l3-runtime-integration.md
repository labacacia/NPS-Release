<!--
Copyright 2026 INNO LOTUS PTY LTD
Developed under LabAcacia Open Source Initiative
Licensed under the Apache License, Version 2.0
-->

# NPS Change Request: NOP ↔ L3 Runtime Integration (nps-runner)

**CR ID**: NPS-CR-0007
**Target version**: v1.0.0-alpha.13
**Status**: Implemented (v1.0.0-alpha.13, 2026-06-13)
**Type**: Backward-compatible addition (new NOP §8, `spawn_spec_ref` content-schema standardization, 4 new error codes, new L3 conformance suite)
**Author**: Ori, LabAcacia
**Affected components**: NOP spec (NPS-5) §8 (new), NDP spec (NPS-4) `spawn_spec_ref` field note, `spec/error-codes.md`, `spec/status-codes.md`, `spec/services/conformance/NPS-Node-L3.md` (new), `nps-runner` daemon, all 6 SDKs (NOP frame field additions)
**Issue**: GitHub NPS-Dev#46

---

## 1. Summary

The NPS-Node Profile defines an **L3 (on-demand / FaaS)** activation tier in which an agent
process does not exist until a task arrives. The runtime that materialises such agents is the
`nps-runner` daemon. NOP v0.6 specified task dispatch (TaskFrame / DelegateFrame / SyncFrame)
but left the **runner ↔ orchestration interface** implementation-defined, and NDP marked
`spawn_spec_ref` content as *"standardized at NPS-Node Profile L3 (future NPS-Daemon-Spec)."*

This CR closes that gap by defining a normative interface between `nps-runner` and the NOP
orchestration layer:

- **Task-claim protocol** — how a runner leases a `TaskFrame` from a per-NID inbox without
  two runners executing the same task (`NOP-CLAIM-CONFLICT`).
- **`spawn_spec_ref` content schema** — the OCI/command/resource descriptor a runner resolves
  to construct an ephemeral worker.
- **Lifecycle enforcement** — `idle_timeout_seconds` / `max_runtime_seconds` and their
  terminal error codes.
- **Result reporting** — how the runner reports terminal node state back to the orchestrator.

It adds NOP §8 (normative summary), four error codes, and the `NPS-Node-L3` conformance suite.
No existing wire format changes; all additions are backward compatible.

---

## 2. Motivation

`nps-runner` shipped as a Phase-1 skeleton (alpha.4) with L3 deferred. Standing up the L3
runtime for alpha.13 requires a canonical contract so that:

1. Multiple runner replicas can compete for the same inbox safely (no double-execution).
2. `spawn_spec_ref` is interoperable across runner implementations rather than per-vendor.
3. Idle / max-runtime semantics are uniform, so cost and liveness behave predictably.
4. An L3 deployment can be conformance-tested (`TC-N3-*`) the way L1/L2 already are.

This CR also unblocks daemon work item 2 of the alpha.13 roadmap (`nps-runner` L3 FaaS runtime).

---

## 3. The L3 Runtime Model

```
 Orchestrator                nps-runner (L3)               Worker (ephemeral)
      │                           │                              │
      │── TaskFrame → inbox ─────▶│  (per-NID inbox queue)       │
      │                           │── claim(lease) ─────────────▶│  (atomic lease)
      │                           │  resolve workers (NDP Resolve)│
      │                           │── spawn(spawn_spec_ref) ─────▶│  cold start
      │                           │── DelegateFrame ────────────▶│  execute node
      │                           │◀──── SyncFrame / result ──────│
      │◀── status PATCH (terminal)│  enforce idle/max-runtime     │
```

A runner is **stateless beyond the active lease set**: a crash releases its leases after the
lease TTL, allowing another runner to reclaim the task (at-least-once execution with a dedup
key — see §4.3).

---

## 4. Task-Claim Protocol

### 4.1 Claim request

A runner claims the head of a per-NID inbox by issuing an atomic lease. The claim carries:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `task_id` | string | Required | The `TaskFrame.task_id` being claimed |
| `runner_nid` | string | Required | The claiming runner's NID |
| `lease_seconds` | uint32 | Required | Requested lease duration; server clamps to `[10, 600]` |
| `dedup_key` | string | Required | `sha256(task_id ‖ dag_hash)` — guards against duplicate execution on reclaim |

### 4.2 Claim outcomes

- **Granted** — the inbox marks the task `LEASED` with `(runner_nid, lease_expiry)`. The runner
  MUST renew the lease (heartbeat) before expiry while the task is running.
- **Conflict** — the task is already `LEASED` by a live lease ⇒ `NOP-CLAIM-CONFLICT`
  (`NPS-CLIENT-CONFLICT`). The runner backs off and polls the next inbox item.
- **Reclaim** — if the prior lease has expired, a new claim succeeds; the `dedup_key` ensures
  any side-effect-bearing node is not re-run if it already reported terminal state.

### 4.3 Idempotency

Result reporting (§6) is keyed by `(task_id, node_id, dedup_key)`. A reclaiming runner that
finds a node already in a terminal state MUST skip re-execution of that node and resume from
the first non-terminal node.

---

## 5. `spawn_spec_ref` Content Schema

`spawn_spec_ref` (NDP AnnounceFrame, NPS-4 §3.1) is an opaque string the runner resolves to a
**SpawnSpec** object. The standardized content schema is:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `image` | string | Required | OCI image reference (e.g. `registry.example.com/agent:1.4.2`) |
| `command` | array[string] | Optional | Entrypoint override; defaults to the image's `CMD` |
| `env` | object | Optional | Environment variables injected into the worker (string→string) |
| `resource_limits` | object | Optional | `{ "cpu": "500m", "memory": "512Mi", "cgn_budget": 50000 }` |
| `idle_timeout_seconds` | uint32 | Optional | Per-worker idle cap; defaults to the runner's policy |
| `max_runtime_seconds` | uint32 | Optional | Per-worker hard runtime cap; defaults to the runner's policy |

`spawn_spec_ref` MAY be (a) an inline `spawnspec:` data URI carrying base64url-encoded JSON of
the above, or (b) a `https://`/`nwp://` URL the runner fetches and whose body MUST validate
against the schema. Resolution failure ⇒ `NOP-SPAWN-SPEC-INVALID` (`NPS-CLIENT-BAD-PARAM`).

---

## 6. Lifecycle Enforcement & Result Reporting

| Limit | Source (precedence high→low) | Terminal on breach |
|-------|------------------------------|--------------------|
| Idle timeout | SpawnSpec `idle_timeout_seconds` → runner policy | `NOP-RUNTIME-IDLE-TIMEOUT` |
| Max runtime | SpawnSpec `max_runtime_seconds` → runner policy | `NOP-RUNTIME-MAX-RUNTIME` |

On any terminal condition (worker `done` / `failed`, or a limit breach) the runner PATCHes the
node's status back to the orchestrator's task store. Terminal states map to the existing NOP
state machine (§5): `done → COMPLETED`, `failed → FAILED`, idle/max-runtime breach → `FAILED`
with the corresponding code; a `FAILED` node with a `compensate_action` triggers saga rollback
(§3.1.6) exactly as for L1/L2 workers — L3 introduces no new saga path.

---

## 7. New Error Codes

| Code | NPS status | HTTP | Meaning |
|------|------------|------|---------|
| `NOP-CLAIM-CONFLICT` | `NPS-CLIENT-CONFLICT` | 409 | TaskFrame already leased by a live runner lease |
| `NOP-SPAWN-SPEC-INVALID` | `NPS-CLIENT-BAD-PARAM` | 400 | `spawn_spec_ref` could not be resolved or failed SpawnSpec schema validation |
| `NOP-RUNTIME-IDLE-TIMEOUT` | `NPS-SERVER-TIMEOUT` | 504 | Worker exceeded its idle timeout before completing the node |
| `NOP-RUNTIME-MAX-RUNTIME` | `NPS-SERVER-TIMEOUT` | 504 | Worker exceeded its max runtime before completing the node |

`NPS-CLIENT-CONFLICT` is an existing NPS status (HTTP 409); no new status code is introduced.

---

## 8. Conformance (NPS-Node-L3)

This CR introduces `spec/services/conformance/NPS-Node-L3.md` with `TC-N3-*` cases:

| Case | Requirement |
|------|-------------|
| TC-N3-Claim-01 | Concurrent claim by two runners — exactly one is granted, the other gets `NOP-CLAIM-CONFLICT` |
| TC-N3-Claim-02 | Expired lease is reclaimable; `dedup_key` prevents re-running a terminal node |
| TC-N3-Spawn-01 | Valid `spawn_spec_ref` (inline + URL) resolves to a worker; invalid ⇒ `NOP-SPAWN-SPEC-INVALID` |
| TC-N3-Life-01 | Idle timeout breach ⇒ `NOP-RUNTIME-IDLE-TIMEOUT`, node `FAILED` |
| TC-N3-Life-02 | Max-runtime breach ⇒ `NOP-RUNTIME-MAX-RUNTIME`, node `FAILED` |
| TC-N3-DAG-01 | 3-node linear DAG executes end-to-end via spawned workers; result PATCHed to orchestrator |
| TC-N3-Saga-01 | `FAILED` L3 node with `compensate_action` triggers reverse-topological compensation |

---

## 9. SDK Impact

The runner↔orchestrator interface is daemon-side; SDKs gain only the additive NOP frame
fields and error-code constants:

| Language | `SpawnSpec` type + claim fields | 4 new error constants |
|----------|--------------------------------|-----------------------|
| .NET | reference | reference |
| Python / TypeScript / Go / Java / Rust | alpha.13 | alpha.13 |

---

## 10. Open Questions

None blocking. OQ-1 (distributed lease store backend for multi-replica runners) is an
implementation concern deferred to the `nps-runner` design doc; the wire contract is
store-agnostic.

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
