English

# NPS-CR-0010: Bridge Node is Bidirectional

**Status**: Proposed  
**Target**: v1.0.0-alpha.16  
**Date**: 2026-07-12  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  
**Touches**: NPS-2 NWP (§2.1 node taxonomy, Bridge Node semantics, §16 conformance), NPS-4 NDP (§3.1 AnnounceFrame), error-codes, `NPS.NWP.Bridge`, `compat/{mcp,grpc,a2a}-ingress`, client SDK `BridgeNodeDescriptor`

---

## 1. Summary

Resolve a contradiction inside the current spec: **NWP defines Bridge Node as
bidirectional in its taxonomy and as outbound-only in its normative semantics.**
This CR settles it as **bidirectional** — the reading the taxonomy, NPS-CR-0001,
and the reference implementation all already take — and pays off the structural
debt that the outbound-only reading created.

1. Restates Bridge Node as **NPS ↔ non-NPS protocol translation** in both
   directions, and deletes the outbound-only "Direction note" whose premise this
   CR dissolves.
2. Makes direction **declarable on the wire**: `bridge_protocols` retains its
   current meaning (protocols this node translates *outbound*), and a new
   `bridge_inbound_protocols` declares the protocols it *serves inbound*. The
   role name `bridge` identifies the translation family; the two arrays then fix
   the exact obligation per protocol and direction, each with its own §16
   conformance profile. This is the structural difference from `Gateway Node`,
   whose ambiguity had no field to resolve it — an implementer could not tell
   from the role, or from anything else on the wire, whether a stateful cluster
   registry was required. Here the obligation is never hidden: `bridge` plus the
   declared arrays state it completely.
3. Splits §16 Bridge conformance into **outbound** and **inbound** profiles.
4. Consolidates the two duplicate implementations of the inbound half — the
   reference SDK's `McpServerBridge`/`A2aServerBridge` and the published
   `compat/{mcp,grpc,a2a}-ingress` packages — into one Bridge package tree, and
   frees the word *ingress* to mean only the network edge (`nps-ingress` daemon).

This CR does **not** re-merge Anchor Node and Bridge Node, and does not revisit
NPS-CR-0001's split. It only corrects the direction of the Bridge half.

## 2. Motivation

### 2.1 The spec contradicts itself

Three passages define Bridge Node as bidirectional:

| Source | Text |
|---|---|
| NWP §2.1 node taxonomy | "Bridge Node — **Translates between** NPS frames and non-NPS protocols (HTTP/HTTPS, gRPC, MCP, A2A)" |
| NWP "Removed types" | "Split into Anchor Node (cluster entry / NOP routing) and Bridge Node (**NPS↔non-NPS** protocol translation)" |
| NPS-CR-0001 §1.2 | "Protocol translator — bridges NPS frames **to and from** non-NPS protocols" |

Two passages define it as outbound-only:

| Source | Text |
|---|---|
| NWP §2.1 CR-0001 callout | "Bridge Node is a new type whose role is **NPS → external protocol** translation. (*Direction note*: this is the inverse of the bridges historically published under `compat/*-bridge` … those have been renamed `compat/*-ingress` **to free the "Bridge" word** for this new node type.)" |
| NWP Bridge Node — detailed semantics | MUST (1) accept inbound NWP frames carrying `bridge_target`, (2) produce outbound requests in the target protocol, (3) translate responses back |

The callout states its own cause: the outbound-only narrowing exists **so that
`Bridge` and `compat/*-ingress` would not overlap as names**. It is a
naming-driven artifact, not a design decision. NPS-CR-0001's own header already
concedes that its §1/§2 framing was "corrected during implementation".

### 2.2 The implementation ignored the narrowing — correctly

`NPS.NWP.Bridge` ships an inbound surface that the normative semantics never
authorised:

- `McpServerBridge` — *"Inbound MCP adapter that exposes local NPS actions as MCP tools."*
- `A2aServerBridge`, `BridgeServerMiddleware` — *"middleware exposing inbound MCP/A2A Bridge server adapters."*

So the inbound half exists **twice**: once in the SDK's Bridge package, once in
the published `compat/*-ingress` packages. Two implementations of the same
translation, neither derived from the other, and they have already drifted:

| | SDK Bridge inbound | `compat/*-ingress` |
|---|---|---|
| MCP methods | `initialize`, `tools/list`, `tools/call`, `ping` | `initialize`, **`resources/list`**, **`resources/read`**, `tools/list`, `tools/call`, `ping` |
| Protocols served inbound | MCP, A2A (**no gRPC**) | MCP, gRPC, A2A |
| JSON-RPC envelope + error codes | `BridgeJsonRpc` / `BridgeJsonRpcErrorCodes` | separate `JsonRpc.cs` |

Two MCP servers over the same NPS nodes answer differently: a client reaching the
SDK's inbound bridge cannot read Memory Nodes at all. This is a live correctness
defect, not only an aesthetic one.

### 2.3 The `ingress` collision

`ingress` currently names two unrelated things: the `compat/*-ingress` protocol
adapter libraries, and the `nps-ingress` daemon (public network edge — TLS
termination, rate limiting, DDoS defence, CGN debit, NID reputation checks).
That is the same "one name, two jobs" defect NPS-CR-0001 retired `Gateway` for.
Folding the adapters into Bridge dissolves the collision **without renaming
anything published**: `ingress` keeps its standard networking meaning (the edge),
`bridge` means protocol translation, and each word names exactly one thing.

## 3. Specification Changes

### 3.1 Bridge Node semantics (NWP §2.1, rewritten)

Replace the outbound-only MUST list with two direction-scoped lists. A Bridge
Node MUST implement at least one direction and MUST declare which (§3.2).

**Outbound (NPS → external)** — unchanged from today. A Bridge Node serving
outbound MUST:

1. Accept inbound NWP frames carrying a `bridge_target` parameter identifying the
   external protocol and endpoint (`protocol`, `endpoint`, optional `headers`;
   unknown fields ignored).
2. Produce outbound requests in the target protocol's format.
3. Translate target-protocol responses back into NWP frames (typically `CapsFrame`).

**Inbound (external → NPS)** — new. A Bridge Node serving inbound MUST:

4. Expose a server endpoint for each protocol listed in
   `bridge_inbound_protocols`, speaking that protocol's native wire format.
5. Translate a foreign-protocol request into NWP frames addressed to the NPS
   nodes it fronts — Memory Node `Query` for reads, Action/Complex Node `Invoke`
   for calls — and translate the NWP response back into the foreign protocol's
   response format.
6. NOT require the foreign client to possess any NPS addressing, NID, or frame
   knowledge. The foreign client MUST be able to treat the Bridge as a native
   server of its own protocol.
7. Map NWP/NPS error codes onto the foreign protocol's error space using the
   normative mapping table for that protocol (§3.4).

For **MCP inbound** specifically, a conformant Bridge MUST serve `initialize`,
`ping`, `tools/list`, `tools/call`, `resources/list`, and `resources/read` — the
last two are what the SDK's current inbound bridge is missing.

Bridge Nodes remain stateless per request and do not participate in cluster
topology, in **both** directions. A single Bridge Node MAY serve several
protocols and both directions; deployments MAY operate dedicated Bridge Nodes per
protocol or per direction for isolation.

Delete the "Direction note" parenthetical in the §2.1 CR-0001 callout, and restate
the Bridge bullet as: *"**Bridge Node** is a new type whose role is NPS ↔ non-NPS
protocol translation, in both directions. It is stateless per request and does not
participate in cluster topology."*

### 3.2 Direction declaration (NPS-4 NDP §3.1 AnnounceFrame, additive)

`bridge_protocols` (array of strings; values `http`/`grpc`/`mcp`/`a2a`) **retains
its current meaning unchanged**: the protocols this node translates **outbound**.

Add **`bridge_inbound_protocols`** (array of strings, same value domain,
OPTIONAL): the protocols this node **serves inbound**. Absent or empty means the
node exposes no inbound surface.

Rules:

- A node declaring `node_roles: ["bridge"]` MUST have at least one of
  `bridge_protocols` / `bridge_inbound_protocols` non-empty.
- A protocol MAY appear in both arrays (the node bridges it in both directions).
- Receivers MUST treat an absent `bridge_inbound_protocols` as `[]` — which is
  exactly today's outbound-only Bridge Node.

This is what keeps `bridge` from becoming the next `gateway`: the role name no
longer has to carry the direction, because the Announce does.

### 3.3 Bridge Node is a role, not merely a library

A deployment MAY run the inbound translation as a plain hosting library in front
of NPS nodes, with no NID and no `Announce` — as `compat/*-ingress` does today.
Such a deployment is **not** a Bridge Node; it is the Bridge library. Only a
deployment that issues an `Announce` with `node_roles: ["bridge"]` is a Bridge
Node, and only it is bound by §3.1's MUSTs and §3.2's declaration rules.

This preserves NPS-CR-0001's core lesson while permitting the merge: the *package*
carries both directions, the *role* is precisely declared.

### 3.4 Error-code mapping (error-codes.md)

Add one code:

**One genuinely new code:**

| Code | Maps to | Meaning |
|---|---|---|
| `NWP-BRIDGE-DIRECTION-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | The request targets a protocol/direction pair the Bridge did not declare in `bridge_protocols` / `bridge_inbound_protocols`. Response SHOULD include the declared arrays in `hint`. Mapped to `NPS-SERVER-UNSUPPORTED` (501) by analogy with `NWP-RESERVED-TYPE-UNSUPPORTED` — the protocol/direction is well-formed but unserved, not malformed, so it is *not* a `NPS-CLIENT-*` fault. |

**Seven codes already shipping, none of them ever registered.** Auditing the Bridge
surface for this CR turned up `BridgeErrorCodes` in the .NET SDK emitting seven
`NWP-BRIDGE-*` codes on the wire that appear nowhere in `error-codes.md`:
`TARGET-INVALID`, `PROTOCOL-UNSUPPORTED`, `ENDPOINT-INVALID`, `UPSTREAM-FAILED`,
`SERVER-TOOL-NOT-FOUND`, `SERVER-DISPATCHER-MISSING`, `SERVER-DISPATCH-FAILED`.
The codes themselves are well-designed; they were simply never registered, in
violation of the repo's own rule that an error-code change MUST update
`error-codes.md` + `status-codes.md`. This CR registers all seven with their NPS
status mappings (see `error-codes.md`). No wire value changes — this is
regularisation, not redesign.

**Invented status codes, removed.** Six status strings that do not exist in
`status-codes.md` were being emitted across the Bridge surface. In the inbound
dispatchers: `NPS-SERVER-NOT-IMPLEMENTED` (`BridgeServerActionInvoker`) and
`NPS-SERVER-ERROR` (the MCP/A2A server bridges). In the outbound Bridge Node's
NWP-over-HTTP middleware (`BridgeNodeMiddleware`): `NPS-CLIENT-UNAUTHORIZED`,
`NPS-CLIENT-BAD-REQUEST`, `NPS-SERVER-UPSTREAM-FAILED`, and again `NPS-SERVER-ERROR`.
All are replaced with registered values from `NpsStatusCodes`: unauthenticated →
`NPS-AUTH-UNAUTHENTICATED`, malformed frame → `NPS-CLIENT-BAD-FRAME`, missing body
→ `NPS-CLIENT-UNPROCESSABLE`, upstream failure → `NPS-DOWNSTREAM-UNAVAILABLE`,
generic 500 → `NPS-SERVER-INTERNAL`, missing backend →
`NWP-BRIDGE-SERVER-DISPATCHER-MISSING` → `NPS-SERVER-INTERNAL`. Separately, the
`NpsStatusCodes` constants class was missing `ServerUnsupported` even though
`NPS-SERVER-UNSUPPORTED` is spec'd; this CR adds it. (The full sweep was driven by
review finding F7.)

Promote the NWP↔MCP, NWP↔gRPC, and NWP↔A2A error mappings — today implemented
twice and inconsistently — to normative tables, one per protocol, shared by both
directions. These land in **NWP §16.3** (see OQ-3).

## 4. Conformance

§16 Bridge conformance splits into two profiles. An implementation declares which
it claims; claiming neither is not a conformant Bridge Node.

- `TC-N2-BRIDGE-OUT-*` — the existing outbound vectors, unchanged. An
  outbound-only Bridge Node (declaring only `bridge_protocols`) stays conformant
  with no code change.
- `TC-N2-BRIDGE-IN-*` — new: (1) MCP inbound serves all six required methods,
  including `resources/*`; (2) gRPC inbound; (3) A2A inbound; (4) foreign client
  needs no NID/frame knowledge; (5) error mapping matches the §3.4 table;
  (6) undeclared protocol/direction → `NWP-BRIDGE-DIRECTION-UNSUPPORTED`.
  Realized as `TC-N2-BridgeIn-01..06` in
  [NPS-Node-L2 §3.3](../services/conformance/NPS-Node-L2.md) v0.4.

## 5. Backward compatibility

Fully additive on the wire; no existing behaviour changes meaning.

- `bridge_protocols` keeps its exact current semantics.
- `bridge_inbound_protocols` absent ⇒ outbound-only ⇒ today's Bridge Node.
- Existing outbound-only Bridge Nodes remain conformant without modification.
- Existing MCP/gRPC/A2A clients of `compat/*-ingress` see no wire change — the
  same surface is served, from a different package.

**Package migration** (implementation-level, not wire-level):

**The two inbound implementations are not duplicates — they are two deployment
shapes.** This was discovered while implementing, and it corrects an earlier draft
of this table that said "delete the SDK's inbound, keep `compat`'s":

- **SDK `BridgeServerOptions`** dispatches **in-process** through a delegate
  (`DispatchAsync`) to the local node's actions. It never speaks HTTP upstream. Its
  object model is *action-only* — there is no notion of a Memory Node or a query,
  which is the root cause of the missing `resources/*`: not two forgotten methods,
  but an object model with no resources in it.
- **`compat/*-ingress`** fronts **remote** NWP nodes over HTTP (`NwpUpstream.BaseUrl`,
  reading `/.nwm`, `/query`, `/actions`, `/invoke`). It is a standalone gateway.

Both shapes are legitimate and both must survive. Deleting either loses a capability.
The consolidation is therefore a **backend abstraction**, not a deletion:

```
INwpBackend ──┬── InProcessNwpBackend   (delegate dispatch — the SDK's shape)
              └── HttpNwpBackend        (HTTP to a remote node — compat's shape)
                     ▲
        one McpInboundServer / A2aInboundServer / GrpcInboundServer
        serving the full method set, over either backend
```

| Was | Becomes |
|---|---|
| `NPS.NWP.Bridge` | Absorbs everything: the translation core (wire types, JSON-RPC envelope, §16.3 error map), outbound dispatchers, `INwpBackend`, the unified inbound servers, **and** the ASP.NET Core hosting (`BridgeNodeMiddleware`, `BridgeServerMiddleware`, DI extensions) with the protobuf gRPC contract. Kept as one package rather than splitting hosting into a separate `NPS.NWP.Bridge.AspNetCore` — the package already depended on ASP.NET before this CR, so a split would be new surface, not the removal of an existing coupling. The inbound servers still stay off `HttpContext` via the `BridgeInboundOptions` / `BridgeServerOptions` type split, so they remain host-free to drive (stdio, tests) even though both types ship in one assembly. |
| SDK `McpServerBridge` / `A2aServerBridge` | Replaced by `McpInboundServer` / `A2aInboundServer` over `INwpBackend`. The in-process shape is preserved via `InProcessNwpBackend`; `resources/*` now works whenever the backend fronts a Memory Node, and is served as an empty set (still conformant) when it does not. |
| — | `GrpcInboundService` (new): inbound gRPC did not exist in the SDK at all — only `compat/grpc-ingress` had it. It carries over that package's published `nwp_ingress.proto` contract unchanged. |
| `LabAcacia.McpIngress` / `.GrpcIngress` / `.A2aIngress` | One transition release whose public types are marked `[Obsolete]` and delegate to the Bridge package, then the `NPS-{mcp,grpc,a2a}-ingress` repos are archived. **Not** `TypeForwardedTo` — the namespaces differ (`LabAcacia.NPS.McpIngress` → `NPS.NWP.Bridge`), so type forwarding does not apply; these are thin delegating shims. GitHub redirects preserve existing links (verified). |
| `compat` `tools/call` error handling | **Fixed.** It currently returns upstream failures as a *successful* JSON-RPC result with `isError: true` and the raw body. §16.1.2 MUST-4 and §16.3 forbid this for auth-class errors. The merged implementation maps NPS status → JSON-RPC error per §16.3. |
| Client SDKs (go/rust/python/ts/java) | Add `bridge_inbound_protocols` to `BridgeNodeDescriptor`. These SDKs are client-first and carry Bridge *types* only (31–69 LOC each) — no server implementation is required of them. |
| `nps-ingress` daemon | **Unchanged.** Keeps the name; the collision dissolves on its own. No MSI/service-account/install-path churn. |

### 5.1 One client-visible behaviour change: qualified tool / skill names

`tools/list` (MCP) and the A2A AgentCard now emit **qualified** names of the form
`node__action` — e.g. `orders_lookup` becomes `orders-node__orders_lookup`. MCP tool
names are a flat namespace, so once a Bridge can front several nodes at once (which is
exactly what `INwpBackend` makes possible), an unqualified name is no longer guaranteed
unique.

`tools/call` and A2A `tasks/send` remain **forgiving on input**: a bare action id still
resolves, provided it is unambiguous across the configured backends. So an MCP client
written against the alpha.15 Bridge keeps working; only the name it sees in a fresh
`tools/list` changes.

The alternative — qualify only when more than one backend is configured — was rejected.
It would preserve today's names, but adding a second node later would then *silently
rename every tool*, breaking clients at config-change time rather than at upgrade time.
A single announced rename now is the more predictable failure.

## 6. Open questions

- **OQ-1**: Should a new `nps-bridge` daemon be added to the daemons repo to host
  a Bridge Node as a deployable process, alongside `nps-ingress`? The two are
  orthogonal (network edge vs protocol translation) and may co-locate. (Proposed:
  yes, but as a follow-up CR — this CR stays spec + package consolidation.)
- **OQ-2**: HTTP inbound (a REST facade over NPS nodes) has no `compat/http-ingress`
  today and would be genuinely new surface, not a consolidation. (Proposed: out of
  scope; `bridge_inbound_protocols` already reserves the value.)
- **OQ-3**: Should the normative error-mapping tables (§3.4) live in NWP §16 or in
  a new `spec/mappings/` file, given they are per-protocol and will grow?
  (Proposed: NWP §16 for alpha.16; split out if a fourth protocol lands.)
