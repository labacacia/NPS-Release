<!--
Copyright 2026 INNO LOTUS PTY LTD
Developed under LabAcacia Open Source Initiative
Licensed under the Apache License, Version 2.0

Saved verbatim as received from author 2026-04-25.
This is the planning artifact; per Q1 (pre-1.0 mode) it is implemented
directly without becoming a formal RFC. The §1/§2 motivation framing
is corrected during implementation per Q2 — the actual current spec
defines Gateway Node as stateless-only, so this is rename + new role
rather than role split.

STATUS: Implemented in v1.0-alpha.3 (2026-04-26).
Implementation notes (corrections to the framing in §1/§2 below):
  - "Gateway Node" was historically the *cluster entry point + NOP
    routing* role — the role that the CR's §1.1 "cluster control
    plane and external entrypoint" describes. So the existing impl
    is renamed Gateway -> Anchor.
  - "Bridge Node" is the genuinely new role (NPS<->non-NPS protocol
    translation). Phase 1 ships only the spec definition + a
    skeleton NPS.NWP.Bridge package; concrete adapters per protocol
    are deferred to follow-up CRs.
  - The pre-existing compat/{mcp,a2a,grpc}-bridge packages carry the
    INVERSE direction (external -> NPS) and have been renamed
    compat/{mcp,a2a,grpc}-ingress to free the "Bridge" word; sibling
    GitHub/Gitee repos NPS-{mcp,a2a,grpc}-bridge are scheduled to be
    renamed *-ingress as part of the v1.0-alpha.3 release sync (the
    repository rename itself must be performed manually on each
    platform - see tools/cr-0001-rename-instructions.md).
-->

# NPS Change Request: Split Gateway Node into Anchor Node + Bridge Node

**CR ID**: NPS-CR-0001
**Target version**: v1.0-alpha.3
**Status**: Draft, pending review
**Type**: Breaking change (spec-level role split)
**Author**: Ori, LabAcacia
**Affected components**: NWP spec, .NET SDK, conformance tests, public docs

---

## 1. Summary

The current NWP node taxonomy includes a single `Gateway Node` type whose definition has, on inspection, been carrying two distinct roles that should not share one name:

1. **Cluster control plane and external entrypoint** — accepts inbound traffic addressed to a cluster, maintains topology of member nodes, dispatches tasks, aggregates responses. State-aware.
2. **Protocol translator** — bridges NPS frames to and from non-NPS protocols (HTTP, gRPC, MCP, A2A). Stateless per-request.

This CR splits `Gateway Node` into two distinct node types:
- **`Anchor Node`** — replaces the cluster-control-plane semantics
- **`Bridge Node`** — replaces the protocol-translation semantics

The original term `Gateway Node` is retired.

## 2. Motivation

The conflation produces concrete harm even at alpha stage:

- **Implementation ambiguity**: An implementer reading "Gateway Node" cannot tell whether a stateful cluster registry is required, or only stateless translation logic. The two have radically different complexity, failure semantics, and resource profiles.
- **Deployment confusion**: A `Gateway Node` doing protocol translation is per-request scalable; one acting as cluster control plane is a singleton with HA concerns. Operators cannot reason about scale-out without the role being specified.
- **Conformance untestable**: Profile L1/L2/L3 cannot meaningfully require "Gateway Node support" because the two underlying capabilities are independent — an implementation might support one without the other.
- **Naming collision with deployment layer**: A separate naming registry (`docs/services.md`) is introducing a process named `nps-gateway` for Internet ingress duty. With `Gateway Node` retired, this collision is preempted.

The cost of fixing this now (alpha.3, no production users) is near zero. The cost of fixing it after v1.0 freeze would be measured in deprecation cycles, downstream SDK breakage, and ecosystem confusion.

## 3. Specification changes

### 3.1 New node type: Anchor Node

Add to `spec/NPS-2-NWP.md`, Node Types section:

> **Anchor Node**
>
> An Anchor Node is the control plane and external entrypoint of an NPS cluster. It MUST:
>
> 1. Maintain a topology of member nodes within its cluster, including their NIDs, declared capabilities, and `activation_mode`.
> 2. Accept inbound NWP `Action` and `Query` frames addressed to the cluster (rather than to a specific member NID).
> 3. Dispatch frames to appropriate member nodes based on capability declaration and current load.
> 4. Aggregate outbound responses from member nodes into single response streams toward the originating caller.
>
> Member nodes register with their Anchor Node on cluster join via NDP `Announce` frames carrying a `cluster_anchor` field referencing the Anchor Node's NID. Deregistration follows standard NDP offline semantics.
>
> A cluster MUST have at least one Anchor Node. High-availability deployments MAY operate multiple Anchor Nodes for the same cluster; consensus protocol between Anchor Nodes is implementation-defined and outside this specification (deferred to NPS-AaaS Profile L3).
>
> An Anchor Node MAY simultaneously carry other node-type roles (e.g. Memory Node) in deployments where role separation is unnecessary.

### 3.2 New node type: Bridge Node

Add to `spec/NPS-2-NWP.md`, Node Types section:

> **Bridge Node**
>
> A Bridge Node translates between NPS frames and non-NPS protocols. It MUST:
>
> 1. Accept inbound NWP frames carrying a `bridge_target` parameter identifying the external protocol and endpoint.
> 2. Produce outbound requests in the target protocol's format.
> 3. Translate target protocol responses back into NWP frames.
>
> Bridge Nodes are stateless per request and do not participate in cluster topology. A single Bridge Node MAY translate to multiple distinct external protocols; deployments MAY operate dedicated Bridge Nodes per protocol for isolation.
>
> Standard external protocols expected to be supported by reference Bridge Node implementations:
> - HTTP/HTTPS (REST and streaming)
> - gRPC (unary and streaming)
> - MCP (Model Context Protocol)
> - A2A (Agent-to-Agent protocol)
>
> Additional protocol adapters MAY be registered through future CRs.

### 3.3 Removal of Gateway Node

In `spec/NPS-2-NWP.md`:
- Delete the `Gateway Node` section in its entirety.
- Add a "Removed types" subsection under Node Types containing:
  > **Gateway Node** (removed in v1.0-alpha.3) — Split into Anchor Node and Bridge Node. See NPS-CR-0001.

### 3.4 Wire format changes

In NDP `Announce` frame, the `node_kind` field:

| Old wire value | Status | New wire value(s) |
|---|---|---|
| `"gateway"` | Removed | `"anchor"` and/or `"bridge"` (a node MAY declare multiple) |

The `node_kind` field is redefined to accept either a string (single role) or an array of strings (multiple roles). Implementations MUST support both forms when parsing.

New optional fields in `Announce`:
- `cluster_anchor` (string, NID): For non-Anchor nodes joining a cluster, identifies the Anchor Node they register with. Absent for standalone nodes and for Anchor Nodes themselves.
- `bridge_protocols` (array of strings): For nodes declaring `"bridge"` in `node_kind`, lists supported external protocols. Standard values: `"http"`, `"grpc"`, `"mcp"`, `"a2a"`.

### 3.5 Specification files affected

| File | Change |
|---|---|
| `spec/NPS-2-NWP.md` | Add Anchor Node section, add Bridge Node section, remove Gateway Node section, update wire format tables, update examples |
| `spec/NPS-4-NDP.md` | Update `Announce` frame schema with `node_kind` array form, `cluster_anchor`, `bridge_protocols` |
| `spec/services/NPS-AaaS-Profile.md` | Update L1/L2/L3 conformance requirements to reference Anchor and Bridge separately |
| `spec/services/conformance/NPS-Node-L1.md` | Add separate test items for Anchor Node basic registry and Bridge Node basic translation (where applicable to L1 scope) |
| `spec/services/conformance/L2.md`, `L3.md` | Same alignment |
| `README.md` | Update node type list |
| `CHANGELOG.md` | Record breaking change under v1.0-alpha.3 |

## 4. SDK changes (.NET reference SDK)

**Type renames:**

| Old | New |
|---|---|
| `GatewayNode` (class) | Removed; replaced by `AnchorNode` and `BridgeNode` classes |
| `NodeKind.Gateway` (enum) | Removed; replaced by `NodeKind.Anchor` and `NodeKind.Bridge` |
| `[Flags]` semantics on `NodeKind` | Newly required — a node may declare multiple kinds |

**New types:**

```csharp
[Flags]
public enum NodeKind
{
    None = 0,
    Memory = 1 << 0,
    Action = 1 << 1,
    Complex = 1 << 2,
    Anchor = 1 << 3,
    Bridge = 1 << 4,
}

public sealed record AnchorNodeDescriptor(
    Nid Nid,
    IReadOnlyList<Nid> ClusterMembers,
    /* HA group, dispatch policy etc. — TBD per L3 spec */
);

public sealed record BridgeNodeDescriptor(
    Nid Nid,
    IReadOnlySet<string> SupportedProtocols /* "http", "grpc", "mcp", "a2a" */
);
```

**Serialization:**

`NodeKind` flag combinations serialize to JSON arrays in NDP `Announce`:
- `NodeKind.Anchor` → `"node_kind": "anchor"` (single string preserved when only one flag set)
- `NodeKind.Anchor | NodeKind.Memory` → `"node_kind": ["anchor", "memory"]`

Deserializer MUST accept both string and array forms.

**Deprecation aids:**

For one alpha release window (alpha.3 only), the SDK SHOULD include:
- `[Obsolete("Gateway Node has been split. Use AnchorNode for cluster control plane, BridgeNode for protocol translation. See NPS-CR-0001.")]` on a stub `GatewayNode` type that throws on instantiation.
- A wire-level deserializer that, on encountering `"node_kind": "gateway"`, throws with a clear message referencing this CR.

This stub is removed in alpha.4.

## 5. Conformance test changes

**`conformance/L1-test/`**:

- Remove: any test asserting `gateway` wire value support.
- Add: `AnchorNode_BasicRegistry_Test` — verifies an Anchor Node accepts `Announce` from member nodes and responds to topology queries.
- Add: `BridgeNode_HttpTranslation_Test` — verifies a Bridge Node correctly translates a simple NWP Action to an HTTP GET and back.
- Update: any test referencing `node_kind` field must accept both string and array forms.

**`L1-CERTIFIED.md` template**: split the Gateway Node checkbox into two:
- [ ] Implements Anchor Node (basic cluster topology)
- [ ] Implements Bridge Node (HTTP minimum)

Implementations MAY claim L1 with one but not both, and MUST declare which.

## 6. Migration impact

**External impact**: None known. v1.0-alpha.2 has no production deployments. No third-party SDKs or implementations exist as of CR submission.

**Internal impact**:
- `nps-daemon` (in development under `labacacia/nps-daemon`): not yet implementing Gateway Node logic, so no migration burden. Updates align with new CR before first L1 release.
- `nps-claude-bridge` (planned): unaffected — does not depend on Gateway Node.
- NeuronHub design documents: TO UPDATE — replace Gateway Node references with Anchor Node where the cluster-entrypoint role is intended.
- LabCubes presentation materials: TO UPDATE if Gateway Node was mentioned.

**Migration window**: Single release. alpha.3 introduces the split with the deprecation stub described in §4. alpha.4 removes the stub.

## 7. Out of scope (explicit non-changes)

To prevent scope creep, the following are explicitly NOT part of this CR:

- **Anchor Node HA / consensus protocol** — deferred to a future CR aligned with NPS-AaaS Profile L3 work. This CR specifies the role; HA is implementation-defined for now.
- **Bridge Node protocol adapter specifications** — only protocol names are reserved here. Detailed bridging semantics for each protocol (HTTP, gRPC, MCP, A2A) live in separate CRs or annexes.
- **Process-level naming** — the `nps-gateway` process name in `docs/services.md` is a separate decision tracked in a different document. Anchor Node and `nps-gateway` are orthogonal layers (logical role vs deployment form) and may co-locate in a single process or be split across processes; that is a deployment concern, not a spec concern.
- **NDP Discovery semantics** — Anchor Node maintains cluster-internal topology; cross-cluster discovery remains NDP's responsibility. This CR does not modify NDP discovery semantics beyond the `Announce` field additions in §3.4.

## 8. Acceptance criteria

This CR is considered accepted and ready to merge when:

- [ ] All affected spec files updated per §3
- [ ] .NET SDK changes per §4 implemented and `dotnet test` passes
- [ ] L1 conformance tests per §5 updated and passing against the reference daemon implementation
- [ ] `CHANGELOG.md` entry written
- [ ] Migration impact items in §6 confirmed (NeuronHub design doc updated, etc.)
- [ ] At least one independent reviewer (besides the author) signs off

## 9. CHANGELOG entry (proposed text)

```markdown
## [v1.0-alpha.3] - YYYY-MM-DD

### Breaking changes

- **NWP**: Removed `Gateway Node` type. Split into `Anchor Node` (cluster
  control plane + external entrypoint) and `Bridge Node` (NPS↔non-NPS
  protocol translation). The two roles previously conflated under
  Gateway Node have distinct semantics, state requirements, and
  conformance criteria, and are now independently declarable. See
  NPS-CR-0001 for full rationale and migration notes.

- **NDP**: `Announce` frame `node_kind` field now accepts both string
  (single role) and array (multiple roles) forms. New optional fields
  `cluster_anchor` and `bridge_protocols` introduced. Wire value
  `"gateway"` removed; `"anchor"` and `"bridge"` introduced.

- **.NET SDK**: `NodeKind` enum updated with `[Flags]` semantics.
  `GatewayNode` type removed (deprecation stub present in alpha.3, to
  be removed in alpha.4). New `AnchorNodeDescriptor` and
  `BridgeNodeDescriptor` types.

### Migration

No production deployments affected. Implementations consuming alpha.2
must update wire field values and SDK type references before
upgrading to alpha.3. The deprecation stub in alpha.3 will throw
informative errors on legacy usage to aid migration.
```

## 10. Open questions

These are noted for reviewer attention; CR can merge with reasonable defaults but discussion welcome:

1. **Should `node_kind` array form be the canonical form going forward, with single-string accepted only as a parsing courtesy?** — Argues for: cleaner spec long-term, avoids two equally-valid forms. Argues against: most nodes carry one role, single-string is more readable in human-facing JSON.

2. **Should `cluster_anchor` field be mandatory for non-standalone nodes, or optional with implicit "no cluster" default?** — Affects whether single-node deployments must declare anything special.

3. **Should Bridge Node's `bridge_protocols` be open-ended or restricted to a registered enum?** — Open-ended allows third-party protocol adapters without spec changes; enum gives stronger conformance guarantees.

Default proposal in this CR: array form is canonical (single-string accepted), `cluster_anchor` is optional with implicit standalone, `bridge_protocols` is open-ended with reserved standard values. Reviewer may overrule.
