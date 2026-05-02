English | [中文版](./NPS-Node-L2.cn.md)

# NPS-Node-L2 Conformance Suite

**Status**: Draft
**Version**: 0.2
**Date**: 2026-05-01
**Applies-To**: [NPS-AaaS-Profile §4.3](../NPS-AaaS-Profile.md) — Level 2 Standard
**Authors**: Ori Lynn / INNO LOTUS PTY LTD

> This document defines the test cases an Anchor Node implementation MUST pass to claim
> NPS-AaaS-Profile L2 compliance for the **L2-08 topology read-back** requirement
> introduced by [NPS-CR-0002](../../cr/NPS-CR-0002-anchor-topology-queries.md).
>
> The remaining L2 requirements (L2-01 through L2-07 — NOP orchestration, OpenTelemetry
> tracing, CGN Token Budget, preflight, retry/timeout, async actions, AlignStream
> back-pressure) have their conformance test cases tracked in follow-up CRs and are
> **out of scope for this document**. A future CR will collect them under additional
> §3.x sub-sections in this same file.

---

## 1. How to Use This Document

1. The IUT MUST already pass [NPS-Node-L1](./NPS-Node-L1.md) — L2 is strictly additive.
2. Build or install the implementation under test (**IUT**).
3. Start a **peer** — any implementation that already passes L2, or the .NET reference
   SDK at `impl/dotnet/src/NPS.NWP.Anchor/` running the `AnchorNodeClient` (CR-0002 §4).
4. Run every test case in §3 against the IUT paired with the peer.
5. A test case passes if and only if **all** of its acceptance criteria hold.
6. All cases in §3 MUST pass for the relevant L2 requirement (L2-08 in this document)
   to be claimed; partial claims are not allowed.
7. Copy [`NPS-NODE-L2-CERTIFIED.md`](./NPS-NODE-L2-CERTIFIED.md) to the IUT repository
   root, fill it in, and sign the attestation with the IUT's root key.

The paired-peer methodology is identical to [NPS-Node-L1 §1](./NPS-Node-L1.md). The
peer requirement upgrades: an L2-passing peer is required wherever L1 used an
L1-passing peer.

Self-certification is sufficient at this release. Third-party certification (NPS Cloud CA)
is targeted for L3 in 2027 Q1+ and is out of scope here.

---

## 2. Test Environment

| Requirement | Value |
|-------------|-------|
| Network | Loopback only; cases do not require external DNS or inter-host routing |
| Peer | Any L2-passing NPS implementation (reference: `NPS.NWP.Anchor` + `NPS.NDP` at NPS v1.0.0-alpha.4 or later) |
| Clock | Wall clock within ±5 s of the peer (ISO 8601 UTC timestamps) |
| File system | Writable directory for the IUT's key store and (if applicable) topology persistence |
| Wire encoding | Tier-1 JSON MUST be exercised; Tier-2 MsgPack SHOULD be exercised |
| Cluster fixture | The IUT MUST act as the **Anchor Node** of a single cluster; member nodes are simulated by the peer issuing NDP `Announce` frames carrying `cluster_anchor` = IUT's NID |

Every case runs in a fresh IUT state: clear the IUT's topology registry between cases
unless the case is explicitly continuation-oriented.

---

## 3. Test Cases

Each case lists the requirement ID from [NPS-AaaS-Profile §4.3](../NPS-AaaS-Profile.md),
the fixture, the actions, and the acceptance criteria.

### 3.1 Anchor Topology — `topology.snapshot` / `topology.stream`

These cases validate L2-08: implementation of the reserved query types defined in
[NPS-2 §12](../../NPS-2-NWP.md). All twelve cases MUST pass for L2-08 to be claimed:
TC-N2-AnchorTopo-01 through TC-N2-AnchorStream-04 cover happy paths; TC-N2-AnchorTopo-04
through TC-N2-AnchorTopo-08 cover required negative paths (one MUST-reject per error code).

#### TC-N2-AnchorTopo-01 — Snapshot of a 3-member cluster
**Req**: L2-08 (`topology.snapshot`)
**Fixture**: IUT acting as Anchor with no prior topology state.
**Action**:
1. Peer announces three member nodes (`M1` Memory, `M2` Action, `M3` Complex) via NDP
   `Announce` frames with `cluster_anchor` set to the IUT's NID.
2. Peer waits 200 ms for ingestion.
3. Peer sends a `QueryFrame` with `type = "topology.snapshot"`, `topology.scope = "cluster"`,
   default `topology.include`.
**Pass**:
- IUT responds with a `CapsFrame` carrying `anchor_ref = "nps:system:topology:snapshot"`.
- Response payload `cluster_size` equals 3.
- Response payload `members` lists exactly the three NIDs `M1`, `M2`, `M3` in some order.
- Each member object carries `nid`, `node_roles`, `activation_mode`.
- `version` is a positive integer.
- Response includes the IUT's NID as `anchor_nid`.

#### TC-N2-AnchorTopo-02 — Version monotonicity across joins
**Req**: L2-08 (version semantics, [NPS-2 §12.3](../../NPS-2-NWP.md))
**Fixture**: IUT acting as Anchor with no prior topology state.
**Action**:
1. Peer announces member `M1`.
2. Peer takes `snapshot_a` (records its `version: V1`).
3. Peer announces member `M2`.
4. Peer takes `snapshot_b` (records its `version: V2`).
**Pass**:
- `V2 > V1` strictly.
- `snapshot_b.cluster_size = snapshot_a.cluster_size + 1`.
- The IUT's `version` counter is monotonic across the run.

#### TC-N2-AnchorTopo-03 — Sub-Anchor member surfaces with `child_anchor` and `member_count`
**Req**: L2-08 (sub-Anchor member representation, [NPS-2 §12.1](../../NPS-2-NWP.md))
**Fixture**: IUT acting as Anchor; peer-simulated child Anchor `CA` with 2 of its own members.
**Action**:
1. Peer announces child Anchor `CA` to the IUT — `node_roles = ["anchor"]`, `cluster_anchor` = IUT NID, with metadata indicating `CA` itself has 2 members.
2. Peer takes a snapshot of the IUT with default `topology.depth = 1`.
**Pass**:
- The member object for `CA` carries `child_anchor: true`.
- The member object for `CA` carries `member_count: 2`.
- The snapshot's `truncated` field is absent or `false` (depth 1 is the default and is not exceeded).

#### TC-N2-AnchorStream-01 — `member_joined` on NDP Announce
**Req**: L2-08 (`topology.stream` join event, [NPS-2 §12.2](../../NPS-2-NWP.md))
**Fixture**: IUT acting as Anchor with no prior topology state.
**Action**:
1. Peer subscribes via `SubscribeFrame` with `type = "topology.stream"`, `topology.scope = "cluster"`.
2. Peer waits for the `subscribed` ack.
3. Peer announces a new member `M1`.
**Pass**:
- Within 1 s of the announce, the IUT pushes a `DiffFrame` with `event_type = "member_joined"`.
- The pushed event's `payload` is a full member object whose `nid` matches `M1`.
- The event's `seq` is greater than the `last_seq` returned in the subscribed ack.

#### TC-N2-AnchorStream-02 — `member_left` on NDP TTL expiry
**Req**: L2-08 (`topology.stream` leave event, [NPS-2 §12.2](../../NPS-2-NWP.md))
**Fixture**: IUT acting as Anchor with one announced member `M1` (TTL configured short — e.g. 2 s).
**Action**:
1. Peer subscribes to `topology.stream`.
2. Peer stops refreshing `M1`'s announce.
3. Peer waits past the TTL.
**Pass**:
- Within `TTL + 1 s` of the last refresh, the IUT pushes a `DiffFrame` with `event_type = "member_left"`.
- The pushed event's `payload.nid` matches `M1`'s NID.
- The event's `seq` is strictly greater than the most recent `member_joined` event.

#### TC-N2-AnchorStream-03 — Resume from `topology.since_version`
**Req**: L2-08 (replay semantics, [NPS-2 §12.2 / §12.3](../../NPS-2-NWP.md))
**Fixture**: IUT acting as Anchor with no prior topology state.
**Action**:
1. Peer subscribes to `topology.stream` and announces `M1`. Peer records the resulting `seq = V1`.
2. Peer announces `M2` and `M3` (yielding `V2`, `V3`).
3. Peer disconnects the subscription.
4. Peer re-subscribes with `topology.since_version = V1`.
**Pass**:
- The IUT's replay yields exactly two events: `member_joined` for `M2` and `M3`, in that order, with `seq` values `V2` and `V3`.
- No event for `M1` is replayed (its `seq = V1` is the boundary; replay starts strictly after).
- No `resync_required` event is emitted.

#### TC-N2-AnchorTopo-04 — Unauthorized topology access (missing `topology:read`) → `NWP-TOPOLOGY-UNAUTHORIZED`
**Req**: L2-08 (authorization gate, [NPS-2 §12.4](../../NPS-2-NWP.md); M6)
**Fixture**: IUT acting as Anchor with one announced member.
**Action**:
1. Peer presents an IdentFrame **without** `topology:read` in `capabilities` (all other required capabilities present).
2. Peer sends a `QueryFrame` with `type = "topology.snapshot"`.
**Pass**:
- IUT responds with an `ErrorFrame` carrying error code `NWP-TOPOLOGY-UNAUTHORIZED`.
- IUT does NOT return any snapshot payload or partial membership data.
- IUT does NOT silently drop the request — an error response MUST be sent.

#### TC-N2-AnchorTopo-05 — Depth cap exceeded → `NWP-TOPOLOGY-DEPTH-UNSUPPORTED`
**Req**: L2-08 (depth enforcement, [NPS-2 §12.1](../../NPS-2-NWP.md))
**Fixture**: IUT acting as Anchor, with its maximum `topology.depth` documented or configurable for the test; use `max_depth = 3` if not otherwise specified.
**Action**:
1. Peer (with `topology:read`) sends a `QueryFrame` with `type = "topology.snapshot"` and `topology.depth = max_depth + 1` (e.g. `4` for `max_depth = 3`).
**Pass**:
- IUT responds with an `ErrorFrame` carrying error code `NWP-TOPOLOGY-DEPTH-UNSUPPORTED`.
- IUT does NOT silently truncate or return a partial snapshot without an error.

#### TC-N2-AnchorTopo-06 — Unrecognized `topology.scope` value → `NWP-TOPOLOGY-UNSUPPORTED-SCOPE`
**Req**: L2-08 (scope validation, [NPS-2 §12.1](../../NPS-2-NWP.md))
**Fixture**: IUT acting as Anchor with one announced member.
**Action**:
1. Peer (with `topology:read`) sends a `QueryFrame` with `type = "topology.snapshot"` and `topology.scope = "nonexistent_scope"`.
**Pass**:
- IUT responds with an `ErrorFrame` carrying error code `NWP-TOPOLOGY-UNSUPPORTED-SCOPE`.
- IUT does NOT silently fall back to a default scope and return data.

#### TC-N2-AnchorTopo-07 — Unrecognized `topology.filter` key → `NWP-TOPOLOGY-FILTER-UNSUPPORTED`
**Req**: L2-08 (filter key validation, [NPS-2 §12.1](../../NPS-2-NWP.md))
**Fixture**: IUT acting as Anchor with one announced member.
**Action**:
1. Peer (with `topology:read`) sends a `QueryFrame` with `type = "topology.snapshot"` and `topology.filter = { "nonexistent_key": "value" }`.
**Pass**:
- IUT responds with an `ErrorFrame` carrying error code `NWP-TOPOLOGY-FILTER-UNSUPPORTED`.
- IUT does NOT silently ignore the unknown key and return unfiltered data.

#### TC-N2-AnchorTopo-08 — Unrecognized reserved `type` value → `NWP-RESERVED-TYPE-UNSUPPORTED`
**Req**: L2-08 (reserved-type validation, [NPS-2 §12](../../NPS-2-NWP.md); M4)
**Fixture**: IUT acting as Anchor.
**Action**:
1. Peer (with `topology:read`) sends a `QueryFrame` with `type = "topology.nonexistent_operation"`.
**Pass**:
- IUT responds with an `ErrorFrame` carrying error code `NWP-RESERVED-TYPE-UNSUPPORTED`.
- IUT does NOT respond with `NWP-ACTION-NOT-FOUND` (these codes are explicitly distinct — see NPS-2 §13).
- IUT does NOT silently ignore the unknown type.

#### TC-N2-AnchorStream-04 — `resync_required` when version is too old
**Req**: L2-08 (`resync_required` semantics, [NPS-2 §12.2](../../NPS-2-NWP.md))
**Fixture**: IUT acting as Anchor with retention buffer configured to 5 events.
**Action**:
1. Peer announces 10 members in sequence (yielding `seq` values `V1..V10`).
2. Peer subscribes to `topology.stream` with `topology.since_version = 1`.
**Pass**:
- The IUT's first pushed event has `event_type = "resync_required"` with `payload.reason = "version_too_old"`.
- No member-event payloads are sent before the `resync_required` event.
- The peer can recover by issuing a fresh `topology.snapshot` and resubscribing without `topology.since_version`.

---

## 4. Results Manifest

A conformance run produces a manifest (JSON) summarizing per-case outcomes. The
manifest is embedded into [`NPS-NODE-L2-CERTIFIED.md`](./NPS-NODE-L2-CERTIFIED.md):

```json
{
  "profile": "NPS-Node-L2",
  "profile_version": "0.1",
  "scope": ["L2-08"],
  "iut": {
    "name": "example-anchor",
    "version": "0.1.0",
    "nid": "urn:nps:node:example.com:anchor-01"
  },
  "peer": {
    "name": "nps-dotnet-reference",
    "version": "1.0.0-alpha.4"
  },
  "run": {
    "date": "2026-04-27T00:00:00Z",
    "environment": "linux-x64 / 1 vCPU / 1 GB"
  },
  "cases": [
    { "id": "TC-N2-AnchorTopo-01", "result": "pass" },
    { "id": "TC-N2-AnchorTopo-02", "result": "pass" },
    { "id": "TC-N2-AnchorTopo-03", "result": "pass" },
    { "id": "TC-N2-AnchorTopo-04", "result": "pass" },
    { "id": "TC-N2-AnchorTopo-05", "result": "pass" },
    { "id": "TC-N2-AnchorTopo-06", "result": "pass" },
    { "id": "TC-N2-AnchorTopo-07", "result": "pass" },
    { "id": "TC-N2-AnchorTopo-08", "result": "pass" },
    { "id": "TC-N2-AnchorStream-01", "result": "pass" },
    { "id": "TC-N2-AnchorStream-02", "result": "pass" },
    { "id": "TC-N2-AnchorStream-03", "result": "pass" },
    { "id": "TC-N2-AnchorStream-04", "result": "pass" }
  ],
  "summary": { "pass": 12, "fail": 0, "skip": 0, "na": 0 }
}
```

Certification of L2-08 is granted when **all 12 cases are `pass`**. There are no
optional cases at this scope; an Anchor Node either implements `topology.snapshot`
and `topology.stream` per [NPS-2 §12](../../NPS-2-NWP.md) or it does not.

When future CRs add §3.2 onward to cover L2-01..L2-07, the `scope` array above will
expand and the `summary` totals will update accordingly.

---

## 5. Reference Suite Location

| Language | Path | Status |
|----------|------|--------|
| .NET 10 (xUnit) | `impl/dotnet/tests/NPS.Tests/Daemons/Npsd/AnchorTopologyConformanceTests.cs` | Implemented alongside this CR |
| Python | `impl/python/tests/conformance/node_l2/` | TODO (Phase 2) |
| TypeScript | `impl/typescript/tests/conformance/node-l2/` | TODO (Phase 2) |

The reference suite's test names MUST align with the `TC-N2-*` IDs above so a
test-run report maps 1:1 onto the §4 manifest.

---

## 6. Change Log

| Version | Date | Changes |
|---------|------|---------|
| 0.2 | 2026-05-01 | Added 5 negative-path test cases (TC-N2-AnchorTopo-04 through -08) to enforce the "every MUST-reject clause has a failure-path TC" standard: unauthorized access (M6 capability gate, `NWP-TOPOLOGY-UNAUTHORIZED`), depth cap exceeded (`NWP-TOPOLOGY-DEPTH-UNSUPPORTED`), unrecognized scope (`NWP-TOPOLOGY-UNSUPPORTED-SCOPE`), unrecognized filter key (`NWP-TOPOLOGY-FILTER-UNSUPPORTED`), unrecognized reserved type (`NWP-RESERVED-TYPE-UNSUPPORTED`). Total cases: 7 → 12. Fixed `node_kind` → `node_roles` in TC-N2-AnchorTopo-01 and -03 (M1 consistency). |
| 0.1 | 2026-04-27 | Initial draft: 7 test cases covering L2-08 (`topology.snapshot` / `topology.stream`) per [NPS-CR-0002](../../cr/NPS-CR-0002-anchor-topology-queries.md). Paired-peer methodology inherited from L1. |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
