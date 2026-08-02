English | [中文版](./NPS-Node-L2.cn.md)

# NPS-Node-L2 Conformance Suite

**Status**: Draft
**Version**: 0.5
**Date**: 2026-08-01
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
| Peer | Any L2-passing NPS implementation (reference: `NPS.NWP.Anchor` + `NPS.NDP` at NPS v1.0.0-alpha.11 or later) |
| Clock | Wall clock within ±5 s of the peer (ISO 8601 UTC timestamps) |
| File system | Writable directory for the IUT's key store and (if applicable) topology persistence |
| Wire encoding | Tier-1 JSON MUST be exercised; Tier-2 MsgPack SHOULD be exercised |
| Cluster fixture | The IUT MUST act as the **Anchor Node** of a single cluster; member nodes are simulated by the peer issuing NDP `Announce` frames carrying `cluster_anchor` = IUT's NID |
| Multi-Anchor fixture | §3.4 (`TC-N2-HA-*`) only: the peer additionally simulates one or more **standby Anchors** of the same `cluster_anchor` cluster, and for `TC-N2-HA-07` / `-08` the IUT is exercised in its **NDP Registry** role rather than its Anchor role |

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

### 3.2 NCP-over-TLS Ingress (NPS-RFC-0006 §6)

These cases apply to an IUT that terminates native-mode NCP-over-TLS at an L2 ingress
(e.g. `nps-ingress`). They validate the transport-security admission gate added by
[NPS-RFC-0006 §6](../../rfcs/NPS-RFC-0006-ncp-native-transport.md) and NCP v0.8 §7.5.

#### TC-N2-Tls-01 — ALPN `nps/1.0` negotiated over TLS 1.3
**Req**: NPS-RFC-0006 §6.1–§6.2
**Fixture**: IUT acting as L2 terminator with a server certificate configured.
**Action**:
1. Peer opens a TLS 1.3 connection offering ALPN `nps/1.0` and a valid client certificate.
**Pass**:
- The handshake completes with negotiated ALPN `nps/1.0`.
- A peer offering no/unknown ALPN is failed with TLS alert `no_application_protocol` (120).

#### TC-N2-Tls-02 — Mutual TLS required
**Req**: NPS-RFC-0006 §6.3 (mTLS)
**Fixture**: IUT with `RequireClientCert = true`.
**Action**:
1. Peer attempts to connect without presenting a client certificate.
**Pass**:
- The IUT refuses the connection (no NCP frames are forwarded to the backend).

#### TC-N2-Tls-03 — Client cert validates to a trust anchor and binds the NID
**Req**: NPS-RFC-0006 §6.3
**Fixture**: IUT with a configured trust anchor; peer holds a NIP leaf cert chaining to it.
**Action**:
1. Peer connects with the valid client certificate and sends the preamble + HelloFrame + IdentFrame.
**Pass**:
- The certificate validates to the trust anchor; the certificate subject NID is bound to the session.
- The terminated NCP byte stream is proxied to the backend verbatim (preamble + frames replayed).

#### TC-N2-Tls-04 — IdentFrame/certificate NID mismatch → `NCP-NID-MISMATCH`
**Req**: NPS-RFC-0006 §6.3, error code `NCP-NID-MISMATCH`
**Fixture**: IUT as above.
**Action**:
1. Peer presents a valid certificate for NID `urn:nps:agent:A` but its IdentFrame declares NID `urn:nps:agent:B`.
**Pass**:
- The IUT closes the session with `NCP-NID-MISMATCH` and does NOT forward the IdentFrame to the backend.

### 3.3 Bridge Node Inbound ([NPS-2 §16.1.2](../../NPS-2-NWP.md), [NPS-CR-0010](../../cr/NPS-CR-0010-bridge-bidirectional.md))

These cases apply to an IUT that declares a non-empty `bridge_inbound_protocols`
(NPS-4 §3.1) — i.e. claims the inbound Bridge conformance profile. The existing
outbound `bridge_target` round-trip vectors ([NPS-2 §16.4](../../NPS-2-NWP.md))
remain the outbound profile's suite and are unchanged; NPS-CR-0010 §4 refers to
the two families as `TC-N2-BRIDGE-OUT-*` / `TC-N2-BRIDGE-IN-*`, realized here as
the §16.4 vectors and `TC-N2-BridgeIn-01..06` respectively.

#### TC-N2-BridgeIn-01 — MCP inbound serves the full required method set
**Req**: NPS-2 §16.1.2 MUST-3
**Fixture**: IUT declaring `bridge_inbound_protocols: ["mcp"]`, fronting one Memory Node and one Action Node.
**Action**:
1. A plain MCP client (JSON-RPC 2.0, no NPS libraries) calls, in order: `initialize`, `ping`,
   `tools/list`, `tools/call`, `resources/list`, `resources/read`.
**Pass**:
- All six methods return successful JSON-RPC results.
- `tools/list` surfaces the Action Node's actions with qualified `node__action` names (CR-0010 §5.1).
- `resources/list` / `resources/read` project the Memory Node's read surface; an IUT fronting no
  Memory Node MUST still serve both methods (empty set), not return "method not found".

#### TC-N2-BridgeIn-02 — gRPC inbound round-trip
**Req**: NPS-2 §16.1.2 MUST-2, MUST-3
**Fixture**: IUT declaring `bridge_inbound_protocols: ["grpc"]`, fronting one Action Node; client uses the published `nwp_ingress.proto` contract.
**Action**:
1. A plain gRPC client calls the unary invoke RPC with a valid action id and JSON payload.
**Pass**:
- The call succeeds and the response payload equals the Action Node's NWP result body.
- The client supplied no NID, frame, or NPS addressing knowledge.

#### TC-N2-BridgeIn-03 — A2A inbound round-trip
**Req**: NPS-2 §16.1.2 MUST-2, MUST-3
**Fixture**: IUT declaring `bridge_inbound_protocols: ["a2a"]`, fronting one Action Node.
**Action**:
1. A plain A2A client fetches the AgentCard, then submits `tasks/send` naming a listed skill.
**Pass**:
- The AgentCard lists the fronted actions as skills with qualified names (CR-0010 §5.1).
- `tasks/send` completes and returns the action result as the task artifact.

#### TC-N2-BridgeIn-04 — Bare action id resolves while ambiguity is rejected
**Req**: NPS-CR-0010 §5.1 (forgiving input)
**Fixture**: IUT fronting two nodes where exactly one defines action `orders_lookup` and both define action `status`.
**Action**:
1. Client calls `tools/call` (MCP) with bare name `orders_lookup`.
2. Client calls `tools/call` with bare name `status`.
**Pass**:
- Call 1 resolves and succeeds (unambiguous bare id).
- Call 2 is rejected with a deterministic JSON-RPC error naming both qualified candidates; it MUST NOT
  silently pick one.

#### TC-N2-BridgeIn-05 — Error mapping matches the §16.3 table
**Req**: NPS-2 §16.1.2 MUST-4, §16.3
**Fixture**: IUT declaring `bridge_inbound_protocols: ["mcp"]`, fronting an Action Node that can be made to fail.
**Action**:
1. Client triggers, in separate calls: an authorization failure, an unknown-action failure, and an
   upstream timeout.
**Pass**:
- Each failure surfaces as a **JSON-RPC error** whose code equals the §16.3 mapping row for the
  underlying NPS status — never as a successful result with `isError: true` for auth-class errors.
- Distinct NPS status classes map to distinct foreign codes (§16.1.2 SHOULD-2 observability).

#### TC-N2-BridgeIn-06 — Undeclared protocol/direction is refused
**Req**: NPS-2 §16.1.2 MUST-5, error code `NWP-BRIDGE-DIRECTION-UNSUPPORTED`
**Fixture**: IUT declaring `bridge_inbound_protocols: ["mcp"]` only.
**Action**:
1. Client sends a well-formed A2A `tasks/send` to the IUT's inbound surface.
**Pass**:
- The request is rejected with `NWP-BRIDGE-DIRECTION-UNSUPPORTED`.
- The response `hint` SHOULD carry both declared arrays (`bridge_protocols`,
  `bridge_inbound_protocols`).

### 3.4 Multi-Anchor High Availability ([NPS-2 §12.2](../../NPS-2-NWP.md), [NPS-4 §9](../../NPS-4-NDP.md), [NPS-CR-0009](../../cr/NPS-CR-0009-multi-anchor-ha.md))

These cases realize the `TC-N2-HA-*` family promised by [NPS-CR-0009](../../cr/NPS-CR-0009-multi-anchor-ha.md) §4.
They validate the cluster-ownership contract that CR adds: the `cluster_epoch` fence
([NPS-2 §12.2](../../NPS-2-NWP.md)), the two finalised `anchor_state` sub-types
`anchor_failover` / `anchor_quorum_lost`, and the highest-epoch resolution rule
([NPS-4 §9](../../NPS-4-NDP.md)). Multi-Anchor HA **operation** is an AaaS Profile L3
requirement; an L1/L2 single-Anchor cluster remains fully conformant and is covered by
TC-N2-HA-09 instead.

The contract spans two node roles, so §4 certifies the family as three groups:
`TC-N2-HA-01..06` exercise the IUT as an **Anchor**; `TC-N2-HA-07..08` exercise it as an
**NDP Registry**; `TC-N2-HA-09` is the single-Anchor mirror of the first group and is
mutually exclusive with it.

Fixture conventions for this sub-section: `C` is the cluster's stable `cluster_anchor` NID;
`A1` / `A2` are Anchors of that cluster; `E` is the `cluster_epoch` the active owner
currently holds. How ownership is acquired, leased, and relinquished is
implementation-defined — CR-0009 §1 deliberately does not specify the consensus transport —
so each case names only the **observable** trigger, and the IUT MUST document the control
surface used to drive it.

> **Underspecified in CR-0009, pinned here for testability**: (a) an inbound frame carrying
> a *lower* `cluster_epoch` than the receiver's own is not covered by CR-0009 §3.1; these
> cases assert nothing about it, and implementations MAY accept or reject it until a
> follow-up CR settles the point. (b) CR-0009 §3.3 says quorum recovery is signalled by
> "a normal `anchor_state` event with a fresh `cluster_epoch`" without naming a sub-type,
> so TC-N2-HA-04 asserts the fresh epoch only, not a particular `field` value.

#### TC-N2-HA-01 — `cluster_epoch` carried on every topology read surface
**Req**: NPS-CR-0009 §3.1 (ownership fence), [NPS-2 §12.2](../../NPS-2-NWP.md)
**Fixture**: IUT acting as the **active** Anchor of cluster `C` at `cluster_epoch = E`; peer simulates standby `A2`; one announced member.
**Action**:
1. Peer (with `topology:read`) sends a `QueryFrame` with `type = "topology.snapshot"`.
2. Peer subscribes via `SubscribeFrame` with `type = "topology.stream"` and reads the `subscribed` ack.
**Pass**:
- The snapshot response carries `cluster_epoch` as a uint64 ≥ 1.
- The `subscribed` ack reports the same `cluster_epoch` as the snapshot.
- Both values equal the epoch under which the IUT currently holds ownership of `C`; the IUT does not report a different epoch on the two surfaces.

#### TC-N2-HA-02 — `anchor_failover` wire shape on planned handover
**Req**: NPS-CR-0009 §3.2, [NPS-2 §12.2](../../NPS-2-NWP.md) `anchor_state` sub-type `anchor_failover`
**Fixture**: IUT acting as the active Anchor of `C` at `cluster_epoch = E`; peer-simulated standby `A2` ready to take ownership.
**Action**:
1. Peer subscribes to `topology.stream` and waits for the `subscribed` ack.
2. Operator triggers a **graceful** ownership handover from the IUT to `A2` through the IUT's documented control surface.
**Pass**:
- Within 1 s the IUT pushes a `DiffFrame` with `event_type = "anchor_state"` and `payload.field = "anchor_failover"`.
- `payload.details` carries all three required fields `successor_nid`, `cluster_epoch`, `reason`.
- `successor_nid` equals `A2`'s NID; `cluster_epoch` is a uint64 **strictly greater** than `E`; `reason = "planned"`.
- The event is not emitted as a bare `member_updated` or `resync_required` — the `anchor_state` envelope with the `field` discriminator is required.

#### TC-N2-HA-03 — `anchor_failover` on active loss is terminal
**Req**: NPS-CR-0009 §3.2 (fenced prior leader sends a terminal `anchor_failover`, then closes its streams)
**Fixture**: IUT acting as the active Anchor of `C` at `cluster_epoch = E` with one attached subscriber; peer-simulated `A2` will claim ownership at `E + 1` once the IUT's lease lapses.
**Action**:
1. Peer subscribes to `topology.stream`.
2. The IUT's ownership lease is made to lapse (e.g. its lease peers are partitioned away); `A2` claims ownership at `E + 1` and the fact is made observable to the IUT.
**Pass**:
- The IUT pushes an `anchor_state` event with `payload.field = "anchor_failover"`, `payload.details.reason = "active_lost"`, `successor_nid` = `A2`'s NID, and `cluster_epoch = E + 1`.
- That event is the **last** event on the stream: the IUT closes the subscription after it and pushes no further topology events.
- After the handover the IUT no longer behaves as the owner — a topology write sent to it is rejected per TC-N2-HA-05.
- The IUT does NOT simply drop the subscription without the terminal event.

#### TC-N2-HA-04 — `anchor_quorum_lost` wire shape and degraded read-only operation
**Req**: NPS-CR-0009 §3.3, [NPS-2 §12.2](../../NPS-2-NWP.md) `anchor_state` sub-type `anchor_quorum_lost`
**Fixture**: IUT acting as the active Anchor of a three-Anchor cluster with `quorum_size = 2`; one announced member; one attached subscriber.
**Action**:
1. Peer subscribes to `topology.stream`.
2. Both peer-simulated Anchors are made unreachable, leaving the IUT alone (`available = 1`, below `quorum_size = 2`).
3. Peer issues a `topology.snapshot` (read) and, separately, a topology-mutating write.
4. The two peer Anchors are restored.
**Pass**:
- The IUT pushes a `DiffFrame` with `event_type = "anchor_state"` and `payload.field = "anchor_quorum_lost"`.
- `payload.details` carries both required fields `quorum_size` and `available` as uint32, with `quorum_size = 2` and `available = 1` matching the fixture.
- The read in step 3 still succeeds (degraded reads are retained) and the IUT's own NDP `AnnounceFrame` reports `health: "degraded"` ([NPS-4 §3.2](../../NPS-4-NDP.md)).
- The write in step 3 is rejected with `NWP-ANCHOR-NOT-LEADER`.
- On recovery the IUT emits an `anchor_state` event whose `cluster_epoch` is strictly greater than the pre-loss epoch; it does NOT resume accepting writes at the pre-loss epoch.

#### TC-N2-HA-05 — Standby Anchor rejects a topology write → `NWP-ANCHOR-NOT-LEADER`
**Req**: NPS-CR-0009 §3.1, error code `NWP-ANCHOR-NOT-LEADER`
**Fixture**: IUT configured as a **standby** Anchor of `C`; the peer-simulated Anchor `A1` holds ownership at `cluster_epoch = E`.
**Action**:
1. Peer (holding `topology:read` and write authority) sends the IUT a topology-mutating push — a `DiffFrame` / `AnnounceFrame` that would change cluster membership — carrying `cluster_epoch = E`.
2. Peer then sends the IUT a `topology.snapshot`.
**Pass**:
- Step 1 is rejected with an `ErrorFrame` carrying `NWP-ANCHOR-NOT-LEADER`, which maps to `NPS-CLIENT-CONFLICT` (HTTP 409 in HTTP mode).
- The IUT's topology is unchanged by the rejected write.
- The IUT does NOT silently forward the write to `A1` and report success — a standby MUST reject, not proxy.
- Step 2 MAY succeed as a stale read; if it does, the response carries the IUT's own last-known `cluster_epoch` (≤ `E`). A standby MUST NOT report an epoch it has never observed.

#### TC-N2-HA-06 — Superseded leader is fenced → `NWP-ANCHOR-EPOCH-FENCED`
**Req**: NPS-CR-0009 §3.1, error code `NWP-ANCHOR-EPOCH-FENCED`
**Fixture**: IUT acting as the active Anchor of `C`, believing itself to hold `cluster_epoch = E`.
**Action**:
1. Peer sends the IUT an inbound topology frame carrying `cluster_epoch = E + 1` — i.e. it comes from a peer that has already taken ownership under a higher epoch.
2. Peer sends a second, otherwise identical frame carrying `cluster_epoch = E`.
**Pass**:
- Step 1 is rejected with an `ErrorFrame` carrying `NWP-ANCHOR-EPOCH-FENCED` (→ `NPS-CLIENT-CONFLICT`).
- The IUT does NOT answer step 1 with `NWP-ANCHOR-NOT-LEADER` — the codes are distinct: `NOT-LEADER` means "you wrote to a non-owner", `EPOCH-FENCED` means "the receiver is a superseded owner".
- The IUT does NOT apply the step-1 frame's effects before rejecting it.
- Step 2 (equal epoch) is NOT fenced: only a **strictly greater** inbound `cluster_epoch` fences the receiver.

#### TC-N2-HA-07 — Registry resolves the highest-`cluster_epoch` Anchor
**Req**: NPS-CR-0009 §3.4, [NPS-4 §9](../../NPS-4-NDP.md) multi-Anchor cluster resolution
**Fixture**: IUT acting as an **NDP Registry** (reference: `nps-registry`) with no prior entries.
**Action**:
1. Peer announces Anchor `A1` with `cluster_anchor = C` and `cluster_epoch = 4`.
2. Peer announces Anchor `A2` with `cluster_anchor = C` and `cluster_epoch = 7`; both entries are live and within TTL.
3. Peer sends a `ResolveFrame` for `C`.
4. Peer re-announces `A1` at `cluster_epoch = 4` (a stale leader still beating) and resolves `C` again.
5. Peer announces Anchor `A3` for a second cluster `C2` **omitting** `cluster_epoch`, alongside an `A4` for `C2` at `cluster_epoch = 2`, and resolves `C2`.
**Pass**:
- Step 3 resolves to `A2` — the live entry with the highest `cluster_epoch`.
- The resolved entry echoes `cluster_epoch = 7`.
- Step 4 still resolves to `A2`: the Registry MUST NOT downgrade cluster `C` to epoch 4 (monotonic per cluster).
- Step 5 resolves to `A4`: an announcement omitting `cluster_epoch` is treated as epoch `1` and therefore loses to any explicit epoch greater than 1.

#### TC-N2-HA-08 — Equal-epoch split-brain → `NDP-CLUSTER-SPLIT`
**Req**: NPS-CR-0009 §3.4, error code `NDP-CLUSTER-SPLIT`
**Fixture**: IUT acting as an NDP Registry with no prior entries.
**Action**:
1. Peer announces Anchor `A1` with `cluster_anchor = C` and `cluster_epoch = 5`.
2. Peer announces Anchor `A2` with `cluster_anchor = C` and `cluster_epoch = 5`; both entries are live and within TTL.
3. Peer sends a `ResolveFrame` for `C`.
4. Peer lets `A1`'s entry age past its TTL and resolves `C` again.
**Pass**:
- Step 3 responds with an `ErrorFrame` carrying `NDP-CLUSTER-SPLIT` (→ `NPS-CLIENT-CONFLICT`).
- The IUT does NOT pick either Anchor arbitrarily and does NOT return a partial or "best effort" resolution.
- Resolution of any **other** cluster is unaffected — the split is scoped to `C`.
- Step 4 resolves to `A2` and succeeds: the split clears without operator intervention once only one live entry remains (or one side is superseded by a higher epoch).

#### TC-N2-HA-09 — Single-Anchor cluster stays at `cluster_epoch = 1` and emits no HA events
**Req**: NPS-CR-0009 §4 / §5 (backward compatibility), [NPS-2 §12.2](../../NPS-2-NWP.md)
**Fixture**: IUT acting as the **only** Anchor of its cluster with no multi-Anchor HA configured — the default L1/L2 deployment shape.
**Action**:
1. Peer subscribes to `topology.stream` and takes a `topology.snapshot`.
2. Peer drives a full member-churn sequence (join, update, leave) as in TC-N2-AnchorTopo-01 and TC-N2-AnchorStream-01/-02.
3. Peer sends a topology-mutating write **omitting** `cluster_epoch` entirely, as a pre-alpha.17 peer would.
4. Peer restarts the IUT and repeats step 1.
**Pass**:
- Every snapshot and `subscribed` ack in steps 1 and 4 either omits `cluster_epoch` or reports `cluster_epoch = 1`; the value never increases, including across the restart.
- No `anchor_state` event with `payload.field = "anchor_failover"` or `"anchor_quorum_lost"` is pushed at any point in the run.
- The step-3 write is accepted and treated as epoch 1 — the IUT MUST NOT reject a peer for the missing optional field.
- The IUT never emits `NWP-ANCHOR-NOT-LEADER` or `NWP-ANCHOR-EPOCH-FENCED` during the run.
- All twelve §3.1 cases still pass unchanged against this IUT.

---

## 4. Results Manifest

A conformance run produces a manifest (JSON) summarizing per-case outcomes. The
manifest is embedded into [`NPS-NODE-L2-CERTIFIED.md`](./NPS-NODE-L2-CERTIFIED.md):

```json
{
  "profile": "NPS-Node-L2",
  "profile_version": "0.5",
  "scope": ["L2-08"],
  "iut": {
    "name": "example-anchor",
    "version": "0.1.0",
    "nid": "urn:nps:node:example.com:anchor-01"
  },
  "peer": {
    "name": "nps-dotnet-reference",
    "version": "1.0.0-alpha.11"
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
    { "id": "TC-N2-AnchorStream-04", "result": "pass" },
    { "id": "TC-N2-Tls-01", "result": "pass" },
    { "id": "TC-N2-Tls-02", "result": "pass" },
    { "id": "TC-N2-Tls-03", "result": "pass" },
    { "id": "TC-N2-Tls-04", "result": "pass" },
    { "id": "TC-N2-BridgeIn-01", "result": "na" },
    { "id": "TC-N2-BridgeIn-02", "result": "na" },
    { "id": "TC-N2-BridgeIn-03", "result": "na" },
    { "id": "TC-N2-BridgeIn-04", "result": "na" },
    { "id": "TC-N2-BridgeIn-05", "result": "na" },
    { "id": "TC-N2-BridgeIn-06", "result": "na" },
    { "id": "TC-N2-HA-01", "result": "na" },
    { "id": "TC-N2-HA-02", "result": "na" },
    { "id": "TC-N2-HA-03", "result": "na" },
    { "id": "TC-N2-HA-04", "result": "na" },
    { "id": "TC-N2-HA-05", "result": "na" },
    { "id": "TC-N2-HA-06", "result": "na" },
    { "id": "TC-N2-HA-07", "result": "na" },
    { "id": "TC-N2-HA-08", "result": "na" },
    { "id": "TC-N2-HA-09", "result": "pass" }
  ],
  "summary": { "pass": 17, "fail": 0, "skip": 0, "na": 14 }
}
```

The example above is a **single-Anchor** Anchor-only IUT: it passes the 12 topology and
4 TLS cases, is not a Bridge, does not run a Registry, and declares no multi-Anchor HA —
so `TC-N2-HA-01..08` are `na` and only the backward-compatibility case `TC-N2-HA-09`
is exercised.

Certification is granted **per case family, all-or-nothing within the family**:

- **L2-08 topology** (`TC-N2-AnchorTopo-*`, `TC-N2-AnchorStream-*`) — all 12 MUST `pass`
  for any L2 claim. There are no optional cases; an Anchor Node either implements
  `topology.snapshot` and `topology.stream` per [NPS-2 §12](../../NPS-2-NWP.md) or it does not.
- **NCP-over-TLS ingress** (`TC-N2-Tls-*`) — all 4 MUST `pass` for an IUT that terminates
  native-mode NCP-over-TLS; `na` otherwise.
- **Bridge inbound** (`TC-N2-BridgeIn-*`) — all 6 MUST `pass` for an IUT declaring a
  non-empty `bridge_inbound_protocols`; `na` for outbound-only or non-Bridge IUTs.
- **Multi-Anchor HA, Anchor side** (`TC-N2-HA-01`..`TC-N2-HA-06`) — all 6 MUST `pass` for an
  IUT that declares multi-Anchor HA (an AaaS Profile L3 capability, [NPS-CR-0009](../../cr/NPS-CR-0009-multi-anchor-ha.md) §4);
  `na` for a single-Anchor IUT.
- **Multi-Anchor HA, Registry side** (`TC-N2-HA-07`, `TC-N2-HA-08`) — both MUST `pass` for an
  IUT that implements the NDP Registry resolve surface; `na` for an Anchor-only IUT.
- **Single-Anchor backward compatibility** (`TC-N2-HA-09`) — MUST `pass` for a single-Anchor
  IUT; `na` for an IUT that declares multi-Anchor HA. Exactly one of this case and the
  Anchor-side family is `na` in a valid manifest — they are mutually exclusive, never both
  `na` and never both `pass`.

A family marked `na` in full does not block certification of the others; a family
partially `na` is a manifest error.

When future CRs cover the remaining L2 requirements (L2-01..L2-07), the `scope`
array above will expand and the `summary` totals will update accordingly.

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
| 0.5 | 2026-08-01 | New §3.4 **Multi-Anchor High Availability** (`TC-N2-HA-01..09`) realizing the `TC-N2-HA-*` family promised by [NPS-CR-0009](../../cr/NPS-CR-0009-multi-anchor-ha.md) §4: `cluster_epoch` present on both topology read surfaces, `anchor_failover` wire shape on planned handover and on active loss (terminal event then stream close), `anchor_quorum_lost` wire shape plus degraded read-only operation and recovery at a fresh epoch, `NWP-ANCHOR-NOT-LEADER` on a standby write, `NWP-ANCHOR-EPOCH-FENCED` on a superseded leader, NDP highest-epoch resolution with no downgrade, equal-epoch split-brain → `NDP-CLUSTER-SPLIT`, and single-Anchor backward compatibility at `cluster_epoch = 1`. §2 gains a multi-Anchor fixture row; §4 manifest now enumerates four case families (12 topology + 4 TLS + 6 bridge + 9 HA) and defines the mutually exclusive HA-Anchor-side / single-Anchor-backward-compat pairing. Two points left open by CR-0009 (lower-epoch inbound frames; the `anchor_state` sub-type used to signal quorum recovery) are called out in §3.4 rather than asserted. |
| 0.4 | 2026-07-23 | New §3.3 **Bridge Node Inbound** (`TC-N2-BridgeIn-01..06`) realizing the `TC-N2-BRIDGE-IN-*` family promised by [NPS-CR-0010](../../cr/NPS-CR-0010-bridge-bidirectional.md) §4: full MCP method set incl. `resources/*`, gRPC + A2A inbound round-trips, bare-vs-qualified name resolution, §16.3 error-mapping fidelity, and `NWP-BRIDGE-DIRECTION-UNSUPPORTED` refusal. §4 manifest now enumerates all three case families (12 topology + 4 TLS + 6 bridge) with per-family certification and `na` semantics. Also retro-adds the missing 0.3 changelog row. |
| 0.3 | 2026-06-12 | (Retro-added row — change shipped in alpha.13 without a changelog entry.) New §3.2 **NCP-over-TLS Ingress** (`TC-N2-Tls-01..04`) validating the NPS-RFC-0006 §6 admission gate: ALPN `nps/1.0`, mTLS requirement, trust-anchor validation + session NID binding, and `NCP-NID-MISMATCH` on IdentFrame/certificate mismatch. |
| 0.2 | 2026-05-01 | Added 5 negative-path test cases (TC-N2-AnchorTopo-04 through -08) to enforce the "every MUST-reject clause has a failure-path TC" standard: unauthorized access (M6 capability gate, `NWP-TOPOLOGY-UNAUTHORIZED`), depth cap exceeded (`NWP-TOPOLOGY-DEPTH-UNSUPPORTED`), unrecognized scope (`NWP-TOPOLOGY-UNSUPPORTED-SCOPE`), unrecognized filter key (`NWP-TOPOLOGY-FILTER-UNSUPPORTED`), unrecognized reserved type (`NWP-RESERVED-TYPE-UNSUPPORTED`). Total cases: 7 → 12. Fixed `node_kind` → `node_roles` in TC-N2-AnchorTopo-01 and -03 (M1 consistency). |
| 0.1 | 2026-04-27 | Initial draft: 7 test cases covering L2-08 (`topology.snapshot` / `topology.stream`) per [NPS-CR-0002](../../cr/NPS-CR-0002-anchor-topology-queries.md). Paired-peer methodology inherited from L1. |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
