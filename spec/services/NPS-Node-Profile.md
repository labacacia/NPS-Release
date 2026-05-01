English | [中文版](./NPS-Node-Profile.cn.md)

# NPS-Node Profile: Node-Side Compliance Specification

**Status**: Draft
**Version**: 0.1
**Date**: 2026-04-24
**Authors**: Ori Lynn / INNO LOTUS PTY LTD
**Depends-On**: NPS-1 (NCP v0.5), NPS-3 (NIP v0.3), NPS-4 (NDP v0.4)

> Companion to [NPS-AaaS Profile](./NPS-AaaS-Profile.md). Where AaaS Profile defines what a
> *service* must do to expose Agent capabilities, this profile defines what a *node host*
> (daemon, embedded SDK, runtime) must do to participate in the NPS network. The two
> profiles are orthogonal: a deployment may carry both badges, either, or neither.

---

## 1. Overview

### 1.1 Scope and Audience

Two reader groups:

- **Implementers** building an NPS node — daemons, embedded libraries, agent runtimes.
- **Operators** wiring nodes into a graph — NeuronHub, MCP hosts, headless agent fleets.

The document answers one question per level: *what does my node have to do to claim L{n}?*

### 1.2 Compliance Levels at a Glance

Three positions on the activation spectrum:

- **L1 Basic** — *passive*: the node accepts and serves frames; agents are spawned per-message.
- **L2 Interactive** — *active*: the node maintains long-lived agent processes that subscribe and react.
- **L3 Autonomous** — *adaptive*: the node hosts hibernating agents that wake on demand and may spawn ephemeral peers.

### 1.3 Activation Model (load-bearing)

Every agent registered to an L1+ node carries one `activation_mode`:

| Mode | Meaning | Daemon role |
|------|---------|-------------|
| `ephemeral` | Agent is not resident; messages instantiate a fresh process via Agent SDK or equivalent runtime; the process exits when done. | Spawn on demand, garbage-collect after completion. |
| `resident` | Agent runs in a long-lived process subscribed to its inbox; the daemon pushes inbound frames over a long-lived connection. | Relay inbound frames; never spawn. |
| `hybrid` | Agent is resident but hibernates when idle; inbound frames wake the process. | Wake the process, then push; track the idle timer. |

`activation_mode` MUST be present in every AnnounceFrame published by an L1+ node and is
the basis on which senders choose between push and pull for outbound traffic. Wire encoding
is normative in [NPS-4 §3.1](../NPS-4-NDP.md#31-announceframe-0x30).

---

## 2. Compliance Matrix

| Dimension | L1 Basic | L2 Interactive | L3 Autonomous |
|-----------|----------|----------------|---------------|
| **Position** | Passive frame I/O | Subscribe and react | Spawn on demand |
| **NCP encoding** | Tier-1 JSON MUST; Tier-2 OPTIONAL | + Tier-2 MsgPack MUST | + Multiplexed long-lived connections MUST |
| **NIP** | Ed25519 sign / verify MUST | + Trust-chain validation MUST | + Dynamic sub-NID issuance MUST |
| **NDP** | Announce / Resolve (static) MUST | + GraphFrame subscription MUST | + On-demand register / deregister MUST |
| **NWP** | ActionFrame pull MUST | + ActionFrame push + Subscribe MUST | + Query fan-out MUST |
| **NOP** | Not required | + TaskFrame / DelegateFrame MUST | + DAG execution + K-of-N sync barrier MUST |
| **Activation modes** | `ephemeral` only (pull-style pseudo-resident) | `resident` MUST; others MAY | All three; default `ephemeral` |
| **Observability** | Per-frame local log MUST | + Prometheus-style metrics MUST | + OpenTelemetry traces MUST |
| **Conformance** | `nps-node-l1` self-test passes | + `nps-node-l2` self-test | + `nps-node-l3` self-test + third-party audit (NPS Cloud CA, 2027 Q1+) |

The "+" prefix is cumulative: an L2 implementation MUST satisfy every L1 row; an L3
implementation MUST satisfy every L2 row.

---

## 3. Level 1 — Basic Compliance

### 3.1 NCP — Wire format

| Req ID | Description |
|--------|-------------|
| N1-NCP-01 | MUST decode and encode every Tier-1 JSON frame referenced by L1 (HelloFrame, AnchorFrame, IdentFrame, AnnounceFrame, ResolveFrame, ActionFrame, CapsFrame, ErrorFrame). |
| N1-NCP-02 | MUST complete the Hello + Anchor handshake against any conformant peer. |
| N1-NCP-03 | MUST listen on a configurable address; the default is `127.0.0.1:17433` (loopback). Cross-host listening is OPTIONAL at L1. |
| N1-NCP-04 | MAY implement Tier-2 MsgPack; if implemented, MUST negotiate via HelloFrame. |

### 3.2 NIP — Identity

| Req ID | Description |
|--------|-------------|
| N1-NIP-01 | MUST generate (on first run) and persist an Ed25519 root keypair with file permission `0600` or platform equivalent. |
| N1-NIP-02 | MUST sign IdentFrames with the root key and verify IdentFrames on every inbound connection. |
| N1-NIP-03 | MUST expose a stable NID of the form `urn:nps:node:<authority>:<id>`, where `<authority>` is the operator's domain or organization identifier and `<id>` is a stable per-host identifier. |
| N1-NIP-04 | MAY issue session sub-NIDs to hosted agents (REQUIRED at L3). |

### 3.3 NDP — Discovery

| Req ID | Description |
|--------|-------------|
| N1-NDP-01 | MUST emit AnnounceFrames carrying `activation_mode = "ephemeral"` for every hosted agent (since L1 supports only `ephemeral`). |
| N1-NDP-02 | MUST sign every AnnounceFrame with the publisher's IdentFrame private key. |
| N1-NDP-03 | MUST respond to ResolveFrames for any agent listed in the local registry. |
| N1-NDP-04 | MAY subscribe to GraphFrame updates (REQUIRED at L2). |

### 3.4 NWP — Inbox and delivery

| Req ID | Description |
|--------|-------------|
| N1-NWP-01 | MUST maintain a per-NID inbox accepting ActionFrame deliveries. |
| N1-NWP-02 | MUST persist undelivered inbox contents across daemon restart. |
| N1-NWP-03 | MUST serve inbox contents via NWP pull (the receiving agent retrieves on demand). |
| N1-NWP-04 | MUST handle at least 100 QPS of pull requests on commodity hardware (baseline; not a stress benchmark). |
| N1-NWP-05 | MAY push frames over a long-lived connection (REQUIRED at L2). |

### 3.5 NOP — Orchestration

Not required at L1. An L1 node receiving a TaskFrame MAY return `NPS-CLIENT-NOT-IMPLEMENTED`.

### 3.6 Observability

| Req ID | Description |
|--------|-------------|
| N1-OBS-01 | MUST write a structured log entry for every frame received and sent. |
| N1-OBS-02 | Each entry MUST include: ISO 8601 UTC timestamp, direction (`in`/`out`), frame type, source NID, destination NID, frame size in bytes. |
| N1-OBS-03 | Log destination is implementation-defined (file / stdout / journal); local-only storage is acceptable. |

### 3.7 Conformance

An L1 implementation is certified by passing the [`nps-node-l1`](./conformance/NPS-Node-L1.md)
test suite. A passing implementation MAY include the `NPS-NODE-L1-CERTIFIED.md` template
(provided in the conformance directory) at its repository root to declare compliance.

---

## 4. Level 2 — Interactive Compliance

L2 = L1 + every "+" row in the L2 column of §2. Headline additions:

- **NCP**: Tier-2 MsgPack MUST.
- **NIP**: Trust-chain validation against a configured trust anchor MUST.
- **NDP**: GraphFrame subscription MUST; node SHOULD react to incremental changes within the GraphFrame `seq` window.
- **NWP**: ActionFrame push and Subscribe MUST; pull remains available.
- **NOP**: TaskFrame and DelegateFrame MUST be accepted; full DAG execution is L3.
- **Activation**: `resident` mode MUST be supported; `hybrid` MAY be supported.
- **Observability**: Prometheus-style metrics endpoint MUST be exposed (frame counters, inbox depth, connection count, handshake latency histogram).

Detailed requirement IDs (`N2-NCP-*` etc.) are TODO; tracked under [Roadmap Phase 2](../../docs/roadmap.md).

---

## 5. Level 3 — Autonomous Compliance

L3 = L2 + every "+" row in the L3 column. Headline additions:

- **NCP**: A single TCP/TLS connection MUST carry multiplexed streams (multiple concurrent agents per socket).
- **NIP**: The node MUST be able to issue session sub-NIDs to hosted agents on demand, sign them with the root key, and revoke them.
- **NDP**: The node MUST support on-demand register and deregister operations triggered by spawn / shutdown of hosted agents.
- **NWP**: Query fan-out (one inbound query → multiple Memory/Action targets) MUST be supported.
- **NOP**: Full DAG execution and K-of-N sync barriers MUST be supported.
- **Activation**: All three modes MUST be supported; default for new agents is `ephemeral`.
- **Observability**: OpenTelemetry traces (W3C `traceparent` propagation) MUST be emitted for every inter-node hop.

L3 introduces the **spawn-on-demand** requirement. A node MUST be able to instantiate an
`ephemeral` agent process from a `spawn_spec_ref` (see NPS-4 §3.1) without a pre-existing
process; the spawned process MUST receive its target frame within an implementation-defined
startup budget (recommended: ≤ 2 s for headless agents). The schema of `spawn_spec_ref`
content is standardized at L3 in a future companion document (NPS-Daemon-Spec).

Detailed requirement IDs (`N3-*`) are TODO; tracked under [Roadmap Phase 3](../../docs/roadmap.md).

---

## 6. NDP Wire Δ (informative)

This profile depends on three additive fields on NPS-4 NDP v0.4 AnnounceFrame (0x30):

| Field | Required | Used by |
|-------|----------|---------|
| `activation_mode` | REQUIRED for L1+ publishers; OPTIONAL for receivers (default `ephemeral` if absent, for backward compatibility with NPS v1.0-alpha.2 nodes) | All levels |
| `activation_endpoint` | REQUIRED for `resident` / `hybrid` publishers | L2, L3 |
| `spawn_spec_ref` | OPTIONAL; meaningful for `ephemeral` and `hybrid` cold start | L3 |

The normative field definitions, JSON example, and backward-compatibility rules live in
[NPS-4 §3.1](../NPS-4-NDP.md#31-announceframe-0x30) and the corresponding entry in
[`spec/frame-registry.yaml`](../frame-registry.yaml).

---

## 7. Conformance

### 7.1 Test suites

| Level | Document | Reference suite |
|-------|----------|-----------------|
| L1 | [`conformance/NPS-Node-L1.md`](./conformance/NPS-Node-L1.md) | .NET 10 / xUnit (this release) |
| L2 | `conformance/NPS-Node-L2.md` | TODO (Roadmap Phase 2) |
| L3 | `conformance/NPS-Node-L3.md` | TODO (Roadmap Phase 3) |

### 7.2 Self-attestation

A passing implementation MAY copy the `NPS-NODE-L{n}-CERTIFIED.md` template from the
conformance directory to its repository root as a public claim of compliance.

### 7.3 Third-party certification

Third-party certification is targeted for L3 in 2027 Q1+ via NPS Cloud CA and is not
available at this release.

---

## 8. Relationship to NPS-AaaS Profile

| Profile | Question answered |
|---------|-------------------|
| NPS-AaaS | "Is my service a compliant Agent-as-a-Service provider?" |
| NPS-Node | "Is my host a compliant participant in the NPS network?" |

A deployment may carry both badges, either, or neither. AaaS L1 does not require Node L1
of its underlying host, but in practice an AaaS Anchor Node operator SHOULD also satisfy
Node L1 for the daemon hosting the Anchor (the role formerly known as "Gateway Node" prior
to NPS-CR-0001).

---

## 9. Change Log

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-04-24 | Initial draft: three-level node compliance (L1/L2/L3), `activation_mode` model, L1 detailed requirement IDs, L1 conformance suite reference, AaaS Profile relationship |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
