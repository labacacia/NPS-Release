<!--
Copyright 2026 INNO LOTUS PTY LTD
Developed under LabAcacia Open Source Initiative
Licensed under the Apache License, Version 2.0

Saved verbatim as received from author 2026-04-25.
Depends on CR-0001 (Anchor Node must exist as a node type before its
topology query surface can be specified). Target alpha.4.

Implementation status flipped 2026-04-27 — see Implementation Notes at the
bottom of this file for what shipped and what remains as follow-up.
-->

# NPS Change Request: Standard Topology Query Types for Anchor Node

**CR ID**: NPS-CR-0002
**Target version**: v1.0-alpha.4 (after CR-0001 lands)
**Status**: Implemented (2026-04-27) — see [§11 Implementation Notes](#11-implementation-notes)
**Type**: Additive (new reserved query types, no wire breakage)
**Author**: Ori, LabAcacia
**Affected components**: NWP spec, Anchor Node implementations, conformance tests

---

## 1. Summary

CR-0001 introduced the Anchor Node type and stated that Anchor Nodes maintain cluster topology. It did not specify how that topology is read. This CR fills that gap by reserving two standard query types that all Anchor Nodes MUST implement:

- **`topology.snapshot`** — one-shot full topology retrieval, served via NWP `Query`.
- **`topology.stream`** — continuous topology change feed, served via NWP `Subscribe`.

Together they enable any NPS client (SDKs, CLIs, web consumers, audit tools, NeuronHub dashboards) to read cluster structure in a uniform way without per-implementation invention.

## 2. Motivation

Without standard topology queries, every Anchor Node implementation invents its own way to expose membership, leading to:

- **Tooling fragmentation** — `nps-starmap`, NeuronHub admin UI, future CLI tools each need bespoke adapters per Anchor Node implementation.
- **Conformance gap** — Profile L2 declares Anchor Node support but cannot meaningfully test it without specifying what queries an Anchor Node answers.
- **No on-ramp for ecosystem builders** — third parties wanting to build dashboards or monitors have no documented contract.

The first concrete consumer driving this CR is the `nps-starmap` 3D topology visualizer (LabAcacia demo project). Designing the queries with a real consumer in hand prevents the "spec written without users" trap.

## 3. Specification changes

### 3.1 Query type registry

Add new section to `spec/NPS-2-NWP.md`: **Reserved Query Types**.

> NWP defines a set of reserved query type identifiers that have specification-defined semantics. Implementations MUST handle reserved query types according to this specification when they declare the relevant node role. Identifiers in the `topology.*` namespace are reserved for cluster topology operations and are mandatory for Anchor Nodes.

### 3.2 `topology.snapshot`

**Frame**: NWP `Query`
**Required of**: All Anchor Nodes (mandatory at Profile L2 and above)
**Idempotent**: Yes
**Caching**: Responses MAY be cached client-side; Anchor Node SHOULD include a `version` field for cache validation against `topology.stream` events.

**Request payload**:
```json
{
  "type": "topology.snapshot",
  "scope": "cluster",
  "include": ["members", "capabilities"],
  "depth": 1
}
```

Fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `scope` | string | yes | `"cluster"` for the Anchor's own cluster; `"member"` with `target_nid` for a single member's metadata |
| `include` | array of strings | no | Subset of `["members", "capabilities", "tags", "metrics"]`. Default: `["members"]`. `"metrics"` is implementation-defined and may be empty. |
| `depth` | integer | no | For sub-Anchor members, controls recursion. `1` (default) = list sub-Anchors as references only. `2+` = recurse into sub-Anchor topology. Anchor Nodes MAY cap depth and respond with `truncated: true` when exceeded. |
| `target_nid` | string | conditional | Required when `scope = "member"`. |

**Response payload**:
```json
{
  "version": 142,
  "anchor_nid": "urn:nps:node:labacacia:host-abc123",
  "cluster_size": 23,
  "members": [
    {
      "nid": "urn:nps:agent:labacacia:host-abc123-sess-aaa",
      "node_roles": ["memory"],
      "activation_mode": "ephemeral",
      "tags": ["dev", "library"],
      "joined_at": "2026-04-15T10:23:00Z",
      "last_seen": "2026-04-26T14:55:00Z"
    },
    {
      "nid": "urn:nps:node:labacacia:host-def456",
      "node_roles": ["anchor"],
      "child_anchor": true,
      "member_count": 7,
      "tags": ["sub-cluster", "training"]
    }
  ],
  "truncated": false
}
```

Fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `version` | integer | yes | Monotonically increasing topology version. Used to correlate snapshot with subsequent `topology.stream` events. |
| `anchor_nid` | string | yes | NID of the responding Anchor Node. |
| `cluster_size` | integer | yes | Total members, regardless of `depth` truncation. |
| `members` | array of member objects | yes | See member object schema below. |
| `truncated` | bool | no | True if `depth` cap was hit. |

**Member object schema**:

| Field | Type | Required | Description |
|---|---|---|---|
| `nid` | string | yes | Member NID. |
| `node_roles` | array of strings | yes | Per CR-0001 node role values. |
| `activation_mode` | string | yes | Per existing NDP definition. |
| `child_anchor` | bool | no | True if this member is itself an Anchor Node of a sub-cluster. Implies `member_count` field. |
| `member_count` | integer | conditional | For sub-Anchor members, count of their direct members. |
| `tags` | array of strings | no | NDP-declared tags. |
| `joined_at`, `last_seen` | RFC 3339 timestamps | no | Implementation-provided. |
| `capabilities`, `metrics` | objects | no | Returned only if requested via `include`. Schema implementation-defined for now (may be standardized in future CR). |

### 3.3 `topology.stream`

**Frame**: NWP `Subscribe`
**Required of**: All Anchor Nodes (mandatory at Profile L2 and above)
**Cancelable**: Yes, via standard NWP `Unsubscribe`.

**Subscribe request**:
```json
{
  "type": "topology.stream",
  "scope": "cluster",
  "filter": { "tags_any": ["dev", "library"] },
  "since_version": 142
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `scope` | string | yes | `"cluster"` (default for Anchor's own); future scopes reserved. |
| `filter` | object | no | Reduces event volume. Supported keys: `tags_any` (array, match-any), `tags_all` (array, match-all), `node_roles` (array). Anchor Node MAY reject unsupported filter keys with an error. |
| `since_version` | integer | no | Resume from a previous version. Anchor Node MUST replay missed events when feasible; if the version is too old, MUST respond with a `resync_required` event and the client MUST issue a fresh `topology.snapshot`. |

**Event types pushed by Anchor Node**:

```json
{ "kind": "event", "version": 143, "type": "member_joined", "member": { ... full member object ... } }
{ "kind": "event", "version": 144, "type": "member_left", "nid": "urn:nps:..." }
{ "kind": "event", "version": 145, "type": "member_updated", "nid": "urn:nps:...", "changes": { "tags": ["new"], "activation_mode": "resident" } }
{ "kind": "event", "version": 146, "type": "anchor_state", "field": "version_rebased", "details": { ... } }
{ "kind": "event", "type": "resync_required", "reason": "version_too_old" }
```

| Event | When emitted |
|---|---|
| `member_joined` | New NDP `Announce` from a node naming this Anchor as `cluster_anchor`. |
| `member_left` | Member explicitly leaves OR exceeds liveness TTL. |
| `member_updated` | Existing member's metadata changes (tags, activation_mode, etc.). |
| `anchor_state` | Rare. Anchor Node internal state change relevant to clients (e.g. version counter rebase after restart). |
| `resync_required` | Subscriber's `since_version` is no longer replayable; client must re-snapshot. |

Every event except `resync_required` carries a `version` matching the post-event topology version. Versions strictly increase per Anchor Node lifetime.

### 3.4 Versioning and consistency model

**Stated guarantees**:

- A snapshot at `version: V` reflects the cluster state after exactly `V` topology mutations.
- An event with `version: V` reflects the cluster state after exactly `V` mutations.
- Combining a snapshot at `V` with all subsequent stream events `V+1, V+2, ...` yields a consistent live view.

**Not guaranteed**:

- Real-time delivery latency (events MAY batch).
- Event ordering across multiple Anchor Nodes (each Anchor has its own version counter).
- Total ordering with non-topology events (Action / Query traffic).

### 3.5 Errors

Standard NWP error responses; topology-specific error codes:

| Code | Meaning |
|---|---|
| `topology.unauthorized` | Caller lacks permission to read this cluster's topology. |
| `topology.unsupported_scope` | Scope value not implemented. |
| `topology.depth_unsupported` | Requested depth exceeds Anchor's max. |
| `topology.filter_unsupported` | Filter contains unrecognized key. |

### 3.6 Affected files

| File | Change |
|---|---|
| `spec/NPS-2-NWP.md` | Add Reserved Query Types section; full schemas for `topology.snapshot` and `topology.stream` |
| `spec/services/NPS-AaaS-Profile.md` | Update L2 to require both query types from any Anchor Node implementation |
| `spec/services/conformance/L2.md` | Add test items |
| `CHANGELOG.md` | v1.0-alpha.4 entry |

## 4. SDK changes (.NET reference SDK)

Add to the SDK:

```csharp
public sealed class AnchorNodeClient
{
    public Task<TopologySnapshot> GetSnapshotAsync(
        TopologyScope scope = TopologyScope.Cluster,
        TopologyInclude include = TopologyInclude.Members,
        int depth = 1,
        CancellationToken ct = default);

    public IAsyncEnumerable<TopologyEvent> SubscribeAsync(
        TopologyFilter? filter = null,
        long? sinceVersion = null,
        CancellationToken ct = default);
}

public sealed record TopologySnapshot(
    long Version,
    Nid AnchorNid,
    int ClusterSize,
    IReadOnlyList<MemberInfo> Members,
    bool Truncated);

public abstract record TopologyEvent(long Version);
public sealed record MemberJoined(long Version, MemberInfo Member) : TopologyEvent(Version);
public sealed record MemberLeft(long Version, Nid Nid) : TopologyEvent(Version);
public sealed record MemberUpdated(long Version, Nid Nid, MemberChanges Changes) : TopologyEvent(Version);
public sealed record AnchorState(long Version, string Field, object Details) : TopologyEvent(Version);
public sealed record ResyncRequired(string Reason) : TopologyEvent(0);
```

Implementation reference goes in the `nps-daemon` Anchor Node module.

## 5. Conformance tests (L2)

Add to `conformance/L2-test/`:

- `Anchor_TopologySnapshot_BasicCluster_Test` — Anchor with 3 members responds to `topology.snapshot` with all 3.
- `Anchor_TopologySnapshot_VersionMonotonic_Test` — Two snapshots taken across a member-join show version strictly increasing.
- `Anchor_TopologyStream_MemberJoin_Test` — Subscriber receives `member_joined` event when a new member registers.
- `Anchor_TopologyStream_MemberLeave_Test` — Subscriber receives `member_left` event on disconnect.
- `Anchor_TopologyStream_ResumeFromVersion_Test` — Resubscribe with `since_version` replays missed events.
- `Anchor_TopologyStream_ResyncRequired_Test` — Resubscribe with too-old version yields `resync_required`.
- `Anchor_TopologySnapshot_SubAnchorMembers_Test` — Sub-Anchor member appears with `child_anchor: true` and `member_count`.

## 6. Migration impact

**External impact**: None. No production users.

**Internal impact**:
- `nps-daemon`: Anchor Node implementation must add these query handlers before claiming L2 conformance. Not blocking for L1 release.
- `nps-starmap`: Built specifically against this contract. Will inform spec revisions during prototyping.
- NeuronHub design: future admin UI consumes these directly. Document as a dependency.

## 7. Out of scope

- **Capabilities and metrics schema standardization**: This CR reserves the field names but leaves contents implementation-defined. A separate CR can standardize once enough implementations exist to know what's needed.
- **Cross-cluster federation queries**: Querying topology across multiple Anchor Nodes is a Profile L3 / NPS Cloud concern. This CR is single-Anchor only.
- **Authorization model**: This CR uses an opaque `topology.unauthorized` error code but does not define when authorization is required. NeuronHub's commercial deployment will need this; a separate CR will handle it.
- **Browser transport (WebSocket)**: Whether `npsd` exposes a WebSocket endpoint for browser clients is tracked separately as a likely NPS-CR-0003. Topology query semantics in this CR are transport-independent.

## 8. Acceptance criteria

- [x] Spec changes per §3 merged (NWP §12 + AaaS-Profile L2-08 + Node-L2 conformance suite)
- [x] .NET SDK types per §4 implemented and tested (`NPS.NWP.Anchor.Topology` + `NPS.NWP.Anchor.Client.AnchorNodeClient`)
- [x] L2 conformance tests per §5 passing on the `NPS.NWP.Anchor` reference Anchor Node implementation (10/10 in `tests/NPS.Tests/Nwp/Anchor/AnchorTopologyTests.cs` — 7 TC-N2-* cases + 3 negative path cases)
- [ ] L2 conformance tests passing on the `nps-daemon` Anchor Node implementation — **deferred**: npsd today is `node_type: "memory"`; promoting it (or adding a sibling Anchor daemon) is tracked as follow-up work
- [ ] `nps-starmap` demo successfully renders a snapshot and updates from stream events — **deferred**: out-of-tree project; the wire contract it consumes is now stable
- [x] CHANGELOG entry written (v1.0-alpha.4 unreleased section)
- [ ] At least one independent reviewer signs off — **pending PR review**

## 9. CHANGELOG entry (proposed)

```markdown
## [v1.0-alpha.4] - YYYY-MM-DD

### Added

- **NWP**: Reserved query type namespace `topology.*`. Defined two
  mandatory queries for Anchor Nodes: `topology.snapshot` (one-shot
  cluster topology) and `topology.stream` (live change feed).
  Mandatory at Profile L2 and above. See NPS-CR-0002.

- **.NET SDK**: `AnchorNodeClient` with `GetSnapshotAsync` and
  `SubscribeAsync` methods; supporting types `TopologySnapshot`,
  `TopologyEvent` hierarchy.

### Notes

- Capability and metrics field schemas remain implementation-defined.
- Cross-cluster federation queries deferred to L3.
```

## 10. Open questions

1. **Should sub-Anchor recursion via `depth: 2+` be mandatory or optional at L2?** — Mandatory simplifies clients but burdens small Anchor implementations. Default proposal: optional; clients can recurse manually by issuing one snapshot per sub-Anchor.

2. **Should `member_updated` events carry the full member object or only the diff?** — Diff is bandwidth-efficient but pushes reassembly complexity to clients. Default proposal: diff (`changes` object), with field-level granularity.

3. **Should there be a separate `topology.health` query for liveness/metrics, or fold into snapshot's `metrics` include?** — Folded means one fewer query type. Separate means health queries can have different cache / push semantics. Default proposal: fold into `metrics`, revisit if real consumers complain.

---

## 11. Implementation Notes

This section was added 2026-04-27 when the CR was flipped to **Implemented**. It records what shipped, what was deliberately reduced in scope, and the open OQ resolutions that became implementation defaults.

### 11.1 OQ resolutions

All three OQ defaults from §10 were accepted at the author's recommendation:

1. **OQ-1 — sub-Anchor recursion at depth ≥ 2 is OPTIONAL at L2.** §12.1 of the spec records "clients SHOULD recurse manually by issuing one snapshot per sub-Anchor." `InMemoryAnchorTopologyService` does not implement depth-2+ recursion; it returns `truncated = false` because at depth 1 the cap is never exceeded. A future implementation MAY add recursion without re-opening this CR.
2. **OQ-2 — `member_updated` carries `changes` (field-level diff), not the full member object.** Encoded by the typed `MemberChanges` record on the .NET side and the `payload.changes` schema in §12.2 on the wire side. The diff is computed in `InMemoryAnchorTopologyService.DiffMembers`.
3. **OQ-3 — health/metrics fold into snapshot's `metrics` include rather than getting their own query type.** No separate `topology.health` is reserved; `topology.include = ["metrics"]` is the documented hook. Schema of the `metrics` payload remains implementation-defined per §12.4.

### 11.2 What landed in this PR

**Spec**:
- `spec/NPS-2-NWP.md` (+`.cn.md`) v0.7 → v0.8: new §12 Reserved Query Types; §6.1 / §8.1 `type` field; §8.2 `event_type` extension; new error codes; new §14.7 Topology Read-back security section; sections renumbered §12→§13 / §13→§14 / §14→§15.
- `spec/services/NPS-AaaS-Profile.md` (+`.cn.md`) v0.3 → v0.4: §4.3 L2-08 row; §2 placeholder removed; §6 NWP version bumped.
- `spec/services/conformance/NPS-Node-L2.md` (+`.cn.md`) v0.1: 7 `TC-N2-*` test cases.
- `spec/services/conformance/NPS-NODE-L2-CERTIFIED.md` v0.1: self-attestation template (mirrors L1 layout).
- `CHANGELOG.md` (+`.cn.md`): new v1.0-alpha.4 unreleased section.

**.NET reference implementation** (in `impl/dotnet/src/NPS.NWP.Anchor`):
- `Topology/TopologyTypes.cs` — public records (`TopologySnapshot` / `MemberInfo` / `TopologyEvent` hierarchy / `TopologyFilter`).
- `Topology/IAnchorTopologyService.cs` — server contract.
- `Topology/InMemoryAnchorTopologyService.cs` — reference implementation: thread-safe member map, monotonic version counter, ring buffer (default 256 events), `RebaseVersion(...)` for restart-and-rebase, `Channel<T>`-based fan-out.
- `Topology/NwpTopologyErrorCodes.cs` — `NWP-TOPOLOGY-*` constants + `TopologyProtocolException`.
- `AnchorNodeMiddleware.cs` — extended with `/query` (type=topology.snapshot) and `/subscribe` (type=topology.stream) routing; topology service resolved from DI (optional).
- `AnchorServiceExtensions.cs` — new `AddInMemoryAnchorTopology(...)` helper.
- `Client/AnchorNodeClient.cs` — typed client over `HttpClient`: `GetSnapshotAsync(...)` + `IAsyncEnumerable<TopologyEvent> SubscribeAsync(...)`; NDJSON wire transport for the stream.

**Tests** (in `impl/dotnet/tests/NPS.Tests/Nwp/Anchor/AnchorTopologyTests.cs`):
- 7 tests mapping 1:1 to `TC-N2-AnchorTopo-01..03` + `TC-N2-AnchorStream-01..04`.
- 3 negative-path tests covering `NWP-TOPOLOGY-UNSUPPORTED-SCOPE`, `NWP-TOPOLOGY-FILTER-UNSUPPORTED`, and unknown reserved-type rejection.
- All 10 pass; full suite remains 602/602 green.

### 11.3 Deferred work (tracked as follow-up)

- **`nps-daemon` Anchor adoption.** `tools/daemons/npsd/` today is `node_type: "memory"`. Two possible paths: (a) extend npsd with an Anchor mode behind a config switch, (b) introduce a separate `nps-anchord` daemon. Either way the integration is mechanical now that `IAnchorTopologyService` + `AddInMemoryAnchorTopology(...)` exist; it is deferred to a follow-up branch to keep this PR's blast radius proportionate to the wire-contract change.
- **`nps-starmap` demo.** Out-of-tree LabAcacia project; will be wired up against this CR's contract once npsd Anchor mode lands.
- **Multi-language SDK clients.** The Python / TypeScript / Java / Rust / Go SDKs do not yet implement `AnchorNodeClient`. Each is a self-contained port and is tracked alongside their existing publish cadences (per `MEMORY.md` "npsd publish lag at alpha.3" item).
- **Authorization model** for `NWP-TOPOLOGY-UNAUTHORIZED` (§12.4 / §3.5 OQ "out of scope"). The error code is wired; the policy is not. Will land in a NeuronHub-driven follow-up CR.
