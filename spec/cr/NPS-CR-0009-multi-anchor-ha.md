English

# NPS-CR-0009: Multi-Anchor High Availability

**Status**: Implemented  
**Target**: v1.0.0-alpha.17  
**Date**: 2026-07-05  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  
**Touches**: NPS-2 NWP (§12.2 topology events, §7 authorization), NPS-4 NDP (§5 GraphFrame, §9 federation), error-codes, services/NPS-AaaS-Profile (L3), NWP/NDP conformance vectors

---

## 1. Summary

Define a minimum, wire-level model for running **more than one Anchor Node per
NWP cluster** so a cluster survives the loss of its active Anchor without a
topology-wide outage. This CR:

1. Promotes the two topology sub-types reserved as Phase-3 placeholders in NWP
   §12.2 — `anchor_failover` and `anchor_quorum_lost` — from placeholder slots to
   a **finalised wire format**, gated behind a `cluster_epoch` fence.
2. Adds a lease-based **single-writer + fencing** ownership model (not full Raft):
   exactly one Anchor holds cluster ownership at a time, identified by a
   monotonically increasing `cluster_epoch`; standby Anchors serve reads.
3. Defines how NDP registries agree on the **current active Anchor per cluster**
   using `cluster_epoch` (highest epoch wins), extending NDP §9 federation.

This CR deliberately does **not** specify the intra-cluster consensus transport
(Raft/Paxos/etc.) — that remains implementation-defined. It specifies only the
**observable wire contract**: the epoch fence, the two failover events, and the
NDP resolution rule.

## 2. Motivation

NWP §7.1 requires "a cluster MUST have at least one Anchor Node" and allows
multiple Anchors for HA, but the consensus between them is "implementation-defined
and deferred to NPS-AaaS Profile L3." Consequently:

- `anchor_failover` / `anchor_quorum_lost` sit in §12.2 as reserved slots that
  implementations **MUST NOT emit** — so a subscriber has no defined way to learn
  that cluster ownership moved.
- Two Anchors for the same `cluster_anchor` NID can both accept topology writes
  (split-brain), and NDP `Resolve` has no rule for which one is authoritative.

alpha.16's "Edge" theme stands up L2/L3 daemons; multi-Anchor HA is the topology
half of production readiness. This CR gives the smallest wire contract that makes
failover observable and split-brain writes rejectable, without mandating a
particular consensus algorithm.

## 3. Specification Changes

### 3.1 Cluster ownership and the epoch fence (NWP §7.2, new)

- Every NWP cluster has a stable `cluster_anchor` NID (already defined). At any
  instant **at most one** Anchor Node is the *active* owner of that cluster; all
  others are *standby*.
- Ownership carries a `cluster_epoch` (uint64, starts at 1). Each time ownership
  changes, the incoming active Anchor MUST use a strictly greater `cluster_epoch`
  than any it has observed. `cluster_epoch` is the fencing token.
- Every topology write (`DiffFrame` / `AnnounceFrame` push that mutates cluster
  topology) and every `topology.snapshot` / `topology.stream` response MUST carry
  the current `cluster_epoch`.
- A standby Anchor MUST reject topology **writes** with `NWP-ANCHOR-NOT-LEADER`
  (→ `NPS-CLIENT-CONFLICT`, HTTP 409) and MAY serve stale reads marked with its
  last-known `cluster_epoch`. An active Anchor MUST reject any inbound frame whose
  `cluster_epoch` is **greater** than its own with `NWP-ANCHOR-EPOCH-FENCED`
  (→ `NPS-CLIENT-CONFLICT`) — this fences a superseded leader.

### 3.2 Finalise `anchor_failover` (NWP §12.2)

Emitted on `topology.stream` when cluster ownership transfers. Removes the
"Phase 3 placeholder — MUST NOT emit" restriction.

| Field | Type | Req | Description |
|---|---|---|---|
| `successor_nid` | string (NID) | required | The Anchor NID that has taken ownership. |
| `cluster_epoch` | uint64 | required | The new (strictly greater) epoch. |
| `reason` | string | required | `planned` (graceful handover) / `active_lost` (health/lease loss). |

On receipt a subscriber MUST update its notion of the active Anchor and MAY
reconnect its stream to `successor_nid`. The prior active Anchor, once fenced,
MUST send a terminal `anchor_failover` then close its streams.

### 3.3 Finalise `anchor_quorum_lost` (NWP §12.2)

Emitted when the Anchor cluster cannot establish/maintain the ownership quorum;
the cluster operates **read-only (degraded)**.

| Field | Type | Req | Description |
|---|---|---|---|
| `quorum_size` | uint32 | required | Number of Anchors required for ownership. |
| `available` | uint32 | required | Number currently reachable. |

While in this state, Anchors MUST reject topology **writes** with
`NWP-ANCHOR-NOT-LEADER` and SHOULD keep serving reads with a `degraded` health
marker (NDP §3.2 `health: "degraded"`). Recovery is signalled by a normal
`anchor_state` event with a fresh `cluster_epoch`.

### 3.4 NDP resolution rule (NPS-4 §9, extended)

- Anchors announce their cluster membership via `AnnounceFrame.cluster_anchor`
  (existing). AnnounceFrames from a cluster's Anchors MUST include the
  `cluster_epoch` they were issued under (new optional field, uint64).
- When a Registry resolves a `cluster_anchor` NID, it MUST return the Anchor with
  the **highest `cluster_epoch`** among live entries; ties (equal epoch, >1 live
  active) are a fault and MUST be reported via `NDP-CLUSTER-SPLIT` (→
  `NPS-CLIENT-CONFLICT`) rather than picking arbitrarily.
- Federated Registries (NDP §9) propagate the `(cluster_anchor, cluster_epoch,
  active_nid)` tuple; a Registry MUST prefer a higher epoch received from a peer,
  and MUST NOT downgrade a cluster to a lower epoch (monotonic per cluster).

### 3.5 Error codes (error-codes.md)

| Code | NPS status | Meaning |
|---|---|---|
| `NWP-ANCHOR-NOT-LEADER` | `NPS-CLIENT-CONFLICT` | Topology write sent to a standby / read-only Anchor. |
| `NWP-ANCHOR-EPOCH-FENCED` | `NPS-CLIENT-CONFLICT` | Inbound frame carries a higher `cluster_epoch`; the receiver is a superseded leader. |
| `NDP-CLUSTER-SPLIT` | `NPS-CLIENT-CONFLICT` | Two live active Anchors advertise the same `cluster_epoch` for one cluster. |

## 4. Conformance

New `TC-N2-HA-*` vectors (L2 topology): (1) standby rejects writes; (2) fenced
leader rejects higher-epoch frames; (3) `anchor_failover` wire shape; (4)
`anchor_quorum_lost` wire shape; (5) NDP resolve returns highest-epoch Anchor;
(6) equal-epoch split → `NDP-CLUSTER-SPLIT`. Multi-Anchor HA operation is an
**AaaS Profile L3** requirement; L1/L2 single-Anchor clusters remain conformant
and MUST simply never emit the two events (epoch stays 1).

## 5. Backward compatibility

Additive and Phase-gated:

- Single-Anchor clusters keep `cluster_epoch = 1` forever and never emit the two
  events — no behaviour change; existing subscribers already MUST ignore unknown
  `anchor_state` sub-types (§12.2 forward-compat rule).
- `cluster_epoch` on frames is optional-on-the-wire and defaults to `1` when
  absent, so pre-alpha.16 peers interoperate as single-Anchor.
- No existing error code changes meaning; three new codes added.

## 6. Open questions

- **OQ-1**: minimum `quorum_size` recommendation (RECOMMENDED `⌊N/2⌋+1`) — leave as
  SHOULD or make MUST for `public-federated`? (Proposed: SHOULD.)
- **OQ-2**: whether `cluster_epoch` should also fence NDP `GraphFrame` topology
  snapshots (§5) or only AnnounceFrame resolution. (Proposed: AnnounceFrame only
  for alpha.16; GraphFrame in a follow-up.)
