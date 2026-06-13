English | [中文版](./NPS-Node-L3.cn.md)

# NPS-Node-L3 Conformance Suite

**Status**: Draft
**Version**: 0.1
**Date**: 2026-06-12
**Applies-To**: [NPS-Node-Profile](../NPS-Node-Profile.md) — Level 3 (on-demand / FaaS), [NPS-CR-0007](../../cr/NPS-CR-0007-nop-l3-runtime-integration.md)
**Authors**: Ori Lynn / INNO LOTUS PTY LTD

> This document defines the test cases an **L3 runtime** (`nps-runner`) implementation MUST
> pass to claim NPS-Node-Profile L3 compliance, for the runner ↔ NOP orchestration interface
> introduced by [NPS-CR-0007](../../cr/NPS-CR-0007-nop-l3-runtime-integration.md) and
> summarised in [NPS-5-NOP §8](../../NPS-5-NOP.md).
>
> L3 is strictly additive over the orchestration semantics already covered by NOP (NPS-5) and
> the L1/L2 node suites; it adds the **task-claim protocol**, **`spawn_spec_ref` resolution**,
> and **idle/max-runtime lifecycle enforcement**.

---

## 1. How to Use This Document

1. The IUT (implementation under test) is an L3 runtime that claims `TaskFrame`s from a
   per-NID inbox and materialises ephemeral workers.
2. Each `TC-N3-*` case lists a precondition, an action, and the required observable outcome.
3. All cases MUST pass for an L3 self-certification (see a future `NPS-NODE-L3-CERTIFIED.md`).

---

## 2. Test Cases

### 2.1 Task-claim protocol (NPS-CR-0007 §4)

| Case | Precondition | Action | Required outcome |
|------|--------------|--------|------------------|
| **TC-N3-Claim-01** | One `TaskFrame` in the inbox | Two runners issue a claim concurrently | Exactly one claim is **granted**; the other returns `NOP-CLAIM-CONFLICT` (HTTP 409). No double execution. |
| **TC-N3-Claim-02** | A task is `LEASED` and the lease has expired | A second runner re-claims; the node had already reported terminal state | Claim is **granted** (reclaim); the already-terminal node is **not** re-executed (`dedup_key` match); execution resumes from the first non-terminal node. |
| **TC-N3-Claim-03** | A running task with an active lease | The owning runner renews the lease before expiry | Lease expiry is extended; no `NOP-CLAIM-CONFLICT` is raised against the owner. |

### 2.2 `spawn_spec_ref` resolution (NPS-CR-0007 §5)

| Case | Precondition | Action | Required outcome |
|------|--------------|--------|------------------|
| **TC-N3-Spawn-01** | AnnounceFrame carries an inline `spawnspec:` base64url-JSON `spawn_spec_ref` | Runner resolves and spawns | Worker constructed from the SpawnSpec `image`/`command`/`env`/`resource_limits`. |
| **TC-N3-Spawn-02** | `spawn_spec_ref` is an `https://`/`nwp://` URL returning valid SpawnSpec JSON | Runner fetches and spawns | Worker constructed; fetched body validated against the schema. |
| **TC-N3-Spawn-03** | `spawn_spec_ref` is unresolvable or fails schema validation | Runner attempts resolution | `NOP-SPAWN-SPEC-INVALID` (HTTP 400); node `FAILED`. |

### 2.3 Lifecycle enforcement (NPS-CR-0007 §6)

| Case | Precondition | Action | Required outcome |
|------|--------------|--------|------------------|
| **TC-N3-Life-01** | Worker with `idle_timeout_seconds = T` | Worker idles past `T` without completing the node | `NOP-RUNTIME-IDLE-TIMEOUT` (HTTP 504); node `FAILED`; worker reaped. |
| **TC-N3-Life-02** | Worker with `max_runtime_seconds = T` | Worker runs past `T` without completing | `NOP-RUNTIME-MAX-RUNTIME` (HTTP 504); node `FAILED`; worker reaped. |

### 2.4 End-to-end & saga

| Case | Precondition | Action | Required outcome |
|------|--------------|--------|------------------|
| **TC-N3-DAG-01** | A 3-node linear DAG TaskFrame in the inbox | Runner claims, resolves workers via NDP `ResolveFrame`, spawns and dispatches `DelegateFrame`s | All three nodes execute end-to-end; terminal status PATCHed to the orchestrator's task store; TaskFrame `COMPLETED`. |
| **TC-N3-Saga-01** | A DAG where an L3 node has a `compensate_action` and a downstream node FAILs | Runner runs the DAG; downstream node fails | Reverse-topological saga rollback (NPS-5 §3.1.6) compensates the completed L3 predecessor; states `COMPENSATING → COMPENSATED`. |

---

## 3. Out of Scope

Distributed lease-store backends (multi-replica coordination) are an implementation concern;
this suite validates only the **observable wire behaviour** of the claim/spawn/lifecycle
contract, which is store-agnostic (NPS-CR-0007 §10 OQ-1).

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
