English | [中文版](./NPS-Roadmap.cn.md)

# NPS Roadmap

**Version**: 0.5  
**Date**: 2026-05-17  
**Owner**: LabAcacia / INNO LOTUS PTY LTD  

---

## Cadence

Each phase breaks into three segments:

```
① Spec Sprint   (2 weeks)   — freeze every spec for this phase
② Impl Sprint   (6–8 weeks) — implementation, testing, documentation
③ Review Gate   (1 week)    — community / internal review before advancing
```

## Version convention

| Version tag | Meaning |
|-------------|---------|
| `v0.x-draft` | internal draft; breaking changes allowed |
| `v0.x-alpha` | public preview; API unstable |
| `v0.x-beta`  | feature-complete; external testing welcome |
| `v1.0`       | spec frozen; production-ready |

---

## Phase 0 — Spec Unification (2026 Q2) — ✅ done

**Goal**: establish the full NPS spec skeleton, unify the frame namespace and naming, produce a v0.2-draft for community comment.

- [x] `NPS-0-Overview.md` v0.3
- [x] `NPS-1-NCP.md` v0.6 (dual transport, configurable frame size, ErrorFrame)
- [x] `NPS-2-NWP.md` v0.8 (Node-published AnchorFrame, CGN, topology queries)
- [x] `NPS-3-NIP.md` v0.5 (metadata field, NPS status codes, X.509 NID prototype)
- [x] `NPS-4-NDP.md` v0.5
- [x] `NPS-5-NOP.md` v0.4
- [x] `frame-registry.yaml` v0.9 (with ErrorFrame 0xFE)
- [x] `error-codes.md` v1.0 (with NPS status-code mapping, NIP cert error codes, NWP topology error codes)
- [x] `status-codes.md` v0.2 (NPS native status codes + HTTP mapping)
- [x] `token-budget.md` v0.2 (CGN metering + tokenizer resolution chain)
- [x] `services/NPS-AaaS-Profile.md` v0.4 (Anchor/Bridge Node, VPL, L1/L2/L3, NPS-CR-0002)
- [x] `services/NPS-Node-Profile.md` v0.1 (L1/L2/L3 + activation modes)
- [x] `services/conformance/NPS-Node-L1.md` v0.1 (21 TC-N1-* test cases)
- [x] `services/conformance/NPS-Node-L2.md` v0.1 (10 TC-N2-* test cases, topology queries)
- [x] LabAcacia repos public, Discussions enabled; `NPS-Dev` monorepo intentionally private
- [x] Published: alpha.1 (2026-04-10), alpha.2 (2026-04-19), alpha.3 (2026-04-26), alpha.4 (2026-04-30), alpha.5 (2026-05-01)

---

## Phase 1 — Core Implementation (2026 Q3) — ✅ shipped

**Goal**: NCP + NWP + NIP + NDP + NOP production-ready across six reference SDKs; NIP CA Server OSS in all six.

### SDKs

| Language   | Package                            | Status |
|------------|------------------------------------|--------|
| .NET       | `LabAcacia.NPS.Core` + `.NWP` + `.NWP.Anchor` + `.NWP.Bridge` + `.NIP` + `.NDP` + `.NOP` | ✅ v1.0.0-alpha.5 (655 tests) |
| Python     | `nps-lib` (PyPI)                   | ✅ v1.0.0-alpha.5 (211+ tests, ≥97% coverage) |
| TypeScript | `@labacacia/nps-sdk` (npm)         | ✅ v1.0.0-alpha.5 (284+ tests) |
| Java       | `com.labacacia.nps:nps-java` (Maven Central) | ✅ v1.0.0-alpha.5 (112+ tests) |
| Rust       | `nps-sdk` + 6 sibling crates (crates.io) | ✅ v1.0.0-alpha.4 (109 tests) |
| Go         | `github.com/labacacia/NPS-sdk-go`  | ✅ v1.0.0-alpha.4 (96 tests) |

### NIP CA Server (six-language reference deployment)

| Language   | Stack                           | Status |
|------------|---------------------------------|--------|
| C# / .NET  | ASP.NET Core + SQLite + Docker  | ✅ v0.1 |
| Python     | FastAPI + SQLite + Docker       | ✅ v0.1 |
| TypeScript | Fastify + SQLite + Docker       | ✅ v0.1 |
| Java       | Spring Boot 3.4 + SQLite        | ✅ v0.1 |
| Rust       | Axum + SQLite + Docker          | ✅ v0.1 |
| Go         | net/http stdlib + SQLite        | ✅ v0.1 |

### .NET server runtime (reference)

- [x] `NPS.Core` — frame codec, AnchorCache, EXT header
- [x] `NPS.NWP` — Memory Node middleware (SQL Server / PostgreSQL), 284 integration tests
- [x] `NPS.NWP.Anchor` — `IAnchorTopologyService` + `topology.snapshot` / `topology.stream` (NPS-CR-0002)
- [x] `NPS.NOP` — DAG validator + orchestration engine, delegation-chain depth limit, SSRF protection, exponential backoff retry, 429 tests
- [x] `NPS.NIP` — CA library: keygen, issuance / revocation, OCSP, CRL

### RFCs and CRs shipped in Phase 1

- [x] **NPS-RFC-0001** — NCP connection preamble `b"NPS/1.0\n"` (Accepted, all 6 SDKs)
- [x] **NPS-RFC-0002 Phase A/B** — X.509 NID + ACME `agent-01` prototype (Draft, all 6 SDKs; IANA PEN pending)
- [x] **NPS-RFC-0003** — Agent identity assurance levels `anonymous`/`attested`/`verified` (Accepted, all 6 SDKs)
- [x] **NPS-RFC-0004 Phase 1+2** — NID reputation log (CT-style); SQLite + RFC 9162 Merkle tree + operator-signed STH + inclusion proofs (`nps-ledger`)
- [x] **NPS-CR-0001** — Anchor/Bridge Node split; `NWP.Gateway` retired; `compat/*-ingress` renamed
- [x] **NPS-CR-0002** — Anchor Node topology queries; L2 conformance suite (10 tests)

### Daemons

| Daemon         | Status at alpha.5 |
|----------------|-------------------|
| `npsd`         | ✅ L1 + sub-NID issuance + per-NID inbox queue (17 integration tests) |
| `nps-registry` | ✅ SQLite-backed real registry (SqliteNdpRegistry, 10 tests) |
| `nps-ledger`   | ✅ Phase 3: SQLite + Merkle + STH + inclusion proofs + STH gossip (33 tests) |
| `nps-runner`   | Phase 1 skeleton (L3 runtime deferred) |
| `nps-ingress`  | Phase 1 skeleton (Internet ingress deferred) |
| `nps-cloud-ca` | Stubbed (2027 Q1+) |

### Completion bar

- [x] SDK unit coverage ≥ 90 % across all six languages
- [x] Memory Node `QueryFrame` round-trip integration tests passing
- [x] NIP CA Server one-command Docker Compose boot documented
- [x] Token-savings baseline ≥ 30 % vs REST (actual: 45.0 %)
- [x] Wire-size baseline vs JSON (actual: 63.6 % aggregate reduction)

---

## Phase 2 — Ecosystem Expansion (2026 Q4) — 🚧 in progress

**Goal**: adapters to existing ecosystems (MCP, A2A, gRPC), richer SDK examples, Tier-2 MsgPack production hardening.

- [x] `compat/mcp-ingress/` — NWP Memory/Action/Complex Node ↔ MCP 2024-11-05 adapter (`LabAcacia.McpIngress` v1.0.0-alpha.4)
- [x] `compat/a2a-ingress/` — NOP `TaskFrame` ↔ A2A Task adapter (`LabAcacia.A2aIngress` v1.0.0-alpha.4)
- [x] `compat/grpc-ingress/` — NWP Memory/Action/Complex Node ↔ gRPC adapter (`LabAcacia.GrpcIngress` v1.0.0-alpha.4)
- [x] Tier-2 MsgPack wire-size benchmark (aggregate 63.6 % reduction vs JSON)
- [x] Token-savings benchmark (aggregate 45.0 % CGN reduction vs REST)
- [x] NOP orchestrator executes a 3-node DAG end-to-end
- [x] Claude Desktop talks to an NWP Memory Node through `mcp-ingress`
- [x] `NDP.ResolveFrame` resolves `nwp://` to a physical endpoint via DNS TXT — `resolve_via_dns` / `resolveWithDns` / `ResolveViaDns` across all six SDKs; injectable `DnsTxtLookup` interface; system resolver per language
- [ ] First reference product: **NPS Studio** (human visual debugger) + **NPS Probe** (Agent Coder conformance CLI)

---

## alpha.5 Release — 2026-05-01 ✅

### Completed in alpha.5

| Item | Notes |
|------|-------|
| **NPS-RFC-0004 Phase 3** — STH gossip for `nps-ledger` | `GossipState` + `GossipService` + `GET /v1/log/gossip/sth`; 13 new tests |
| **AaaS-Profile L2-09** — default `reputation_policy` | SHOULD requirement; minimum recommended policy defined |
| **`NWP-RESERVED-TYPE-UNSUPPORTED`** in AnchorNodeMiddleware | HTTP 501; `NPS-SERVER-UNSUPPORTED` status code added |
| **`topology:read` capability gate** on AnchorNodeMiddleware | `AnchorNodeOptions.RequireTopologyCapability`; `X-NWP-Capabilities` header |
| **`cgn_est` per-event** on `TopologyEventEnvelope` | UTF-8/4 estimate per §7.2 (SHOULD) |
| **AssuranceLevel `from_wire("")`** fix | Python, TS, Java SDKs; `""` → Anonymous |
| **Spec / doc CN sync** | `error-codes.cn.md`, `RFC-0004.cn.md`, `status-codes.cn.md` all up-to-date |

### Deferred to alpha.6

| Item | Notes |
|------|-------|
| **NPS-CR-0002 Phase 2** — server-side Anchor middleware push of topology updates | .NET reference push/notify now lands through `AnchorNodeMiddleware` + `IAnchorTopologyService`; non-.NET ports remain below |
| Non-.NET port of NPS-CR-0002 `AnchorNodeClient` topology client | .NET reference done; Python/TS/Go/Java/Rust need port |
| Non-.NET port of NPS-RFC-0004 reputation helpers (`ReputationLogClient`) | .NET reference done; port all six SDKs |
| Non-.NET port of NPS-RFC-0003 assurance-level enforcement helpers | Wired in .NET; other SDKs have enum only, no enforcement helpers |
| **NPS-RFC-0002** promotion Draft → Proposed/Accepted | Closed by NPS-CR-0004 (2026-05-08): IANA PEN **65715** assigned; OID arc `1.3.6.1.4.1.65715` replaces provisional `1.3.6.1.4.1.99999`; RFC-0002 promoted Draft → Proposed (wire-in lands in alpha.6) |

## alpha.6 Task Queue

Tasks queued for v1.0.0-alpha.6:

### In-flight RFCs / CRs

| Item | Notes |
|------|-------|
| **NPS-CR-0002 Phase 2** — server-side Anchor middleware push of topology updates | .NET reference complete; alpha.6 closes the `node_kind` compatibility window and requires `topology.filter.node_roles` |
| **NPS-RFC-0002** promotion Draft → Proposed/Accepted | Closed by NPS-CR-0004 (2026-05-08): IANA PEN **65715** assigned; OID arc `1.3.6.1.4.1.65715` replaces provisional `1.3.6.1.4.1.99999`; RFC-0002 promoted Draft → Proposed (wire-in lands in alpha.6) |

### SDK parity gaps

| Item | Notes |
|------|-------|
| Non-.NET port of NPS-CR-0002 `AnchorNodeClient` topology client | .NET reference done; Python/TS/Go/Java/Rust need port |
| Non-.NET port of NPS-RFC-0004 reputation helpers (`ReputationLogClient`) | .NET has Phase 1 data types only (no client); all six SDKs need full client |
| ~~Non-.NET port of NPS-RFC-0003 assurance-level enforcement helpers~~ | ✅ Done — all six SDKs have full `AssuranceLevel` enum + enforcement logic |

### Protocol / spec items

| Item | Notes |
|------|-------|
| `NDP.ResolveFrame` DNS TXT resolution (`nwp://` → physical endpoint) | ✅ Implemented in all six SDKs — `resolve_via_dns` / `resolveWithDns` / `ResolveViaDns`; injectable `DnsTxtLookup` |
| `nps-ingress` L2 Internet ingress (`:8080`→`:443` termination, NCP over TLS) | Skeleton only at alpha.4; L2 conformance deferred |
| `nps-runner` L3 FaaS task runtime | Skeleton only; full impl Phase 3 scope |

### Tooling

| Item | Notes |
|------|-------|
| **NPS Studio** — visual debugger for NPS frame streams | Phase 2 target; not yet started |
| **NPS Probe** — Agent Coder conformance CLI | Phase 2 target; not yet started |

---

## alpha.7 Task Queue

Tasks queued for v1.0.0-alpha.7:

### SDK parity (carry-over from alpha.6 — hard release gate)

Every SDK release MUST ship all six languages at the same feature level. The following
items were deferred from alpha.6 and MUST be completed before alpha.7 can be tagged.

| Item | Scope | Notes |
|------|-------|-------|
| NPS-CR-0002 `AnchorNodeClient` | Python / TypeScript / Go / Java / Rust | .NET reference: `NPS-sdk-dotnet/src/NPS.NWP.Anchor/Client/`; ports need `GetSnapshotAsync` + `SubscribeAsync` + topology data types |
| ~~NPS-RFC-0004 `ReputationLogClient`~~ | ~~All six SDKs (incl. .NET)~~ | ✅ Done (2026-05-17) — all six SDKs ship `ReputationLogEntry`, `SignedTreeHead`, `InclusionProof`, full `ReputationLogClient` (submit / query / STH / proof / gossip-STH), `VerifyInclusion` (RFC 9162 Merkle fold), and comprehensive regression tests |

### New spec / implementation

| Item | Notes |
|------|-------|
| **NPS-CR-0005** — NIP CA RA model | Spec completion (Draft → Proposed) + .NET reference impl (three enrollment tiers: allowlist / bootstrap-token / approval-queue) + PostgreSQL migration |
| **#51 CGN Profile换算规范** | Complete `cgn-profiles.yaml` with per-model token-per-CGN conversion tables; add protocol binding in `token-budget.md` |

### In-flight CRs / RFCs

| Item | Notes |
|------|-------|
| **NPS-RFC-0002** promotion Proposed → Accepted | Shepherd review; gated on no open OQs |

---

## alpha.8 Task Queue

Tasks queued for v1.0.0-alpha.8:

### SDK parity (carry-over from alpha.7 — hard release gate)

| Item | Scope | Notes |
|------|-------|-------|
| ~~NPS-CR-0002 `AnchorNodeClient`~~ | ~~Python / TypeScript / Go / Java / Rust~~ | ✅ Done (2026-05-17) — all five ports complete with full test suites (Python 25, TS 24, Go 21, Java 25, Rust 25); Java iterator bug fixed |

### NIP identity hardening

| Item | Notes |
|------|-------|
| ~~**NPS-RFC-0002 OID wire-in**~~ | ✅ Done — all six SDKs already carried `1.3.6.1.4.1.65715` (IANA PEN 65715) since alpha.6; no provisional OID found in any SDK or `nip-ca-server` |
| **NPS-CR-0005 non-.NET CA server ports** | Port the RA model (allowlist / bootstrap-token / approval-queue tiers) to Python / TypeScript / Go / Java / Rust CA servers; parity with the .NET reference impl shipped in alpha.7 |
| **NPS-CR-0005 spec promotion** | `NPS-CR-0005.md` Draft → Proposed; shepherd review; mark Accepted when no open OQs |

### New spec / protocol

| Item | Notes |
|------|-------|
| **NPS-RFC-0005** — Reputation Policy Enforcement | Define wire format for `reputation_policy` in `AnchorNodeOptions` and `IdentFrame.metadata`; enforcement decision lifecycle (accept / throttle / reject); builds on RFC-0003 assurance levels + RFC-0004 reputation log |
| **#51 CGN Profile换算规范** | Finalize `cgn-profiles.yaml` with per-model token-per-CGN conversion tables; add protocol binding in `token-budget.md`; wire `cgn_limit` enforcement into `AnchorNodeMiddleware` |

### Tooling

| Item | Notes |
|------|-------|
| **NPS Probe v0.1** — protocol conformance CLI | CLI that smoke-tests an NPS endpoint: NCP preamble handshake, NWP topology query/stream, NIP identity verification; first Phase-2 tooling milestone |

### In-flight CRs / RFCs

| Item | Notes |
|------|-------|
| **NPS-RFC-0002** promotion Proposed → Accepted | Shepherd review; gated on OID wire-in landing and no open OQs |

---

## Phase 3 — Ecosystem Validation (2027 Q1–Q2)

**Goal**: real-world PoCs, NPS Cloud CA v1.0 live, lay the groundwork for de-facto standard status.

- [ ] NPS Cloud CA v1.0 (multi-region HA, real-time OCSP, Professional Plan)
- [ ] LangChain / AutoGen / CrewAI integration adapter packages
- [ ] FinTech PoC (Open Banking scenario, cross-org `TrustFrame`)
- [ ] Connected-vehicle PoC (device NIDs, `StreamFrame` real-time telemetry)
- [ ] Token-savings benchmark report (publicly released)
- [ ] NIP CA Server OSS v1.0 (PostgreSQL + Web Admin UI)
- [ ] NIP CA Server self-hosted on an NWP Memory Node backend (dogfooding)
- [ ] GitHub stars ≥ 500

---

## Phase 4 — Standardization (2027 Q3 onward)

**Goal**: push NPS toward formal W3C / IETF standardization; freeze NPS 1.0.

- [ ] Joint vendor support statement (≥ 3 vendors)
- [ ] W3C WebAI Community Group proposal
- [ ] IETF Internet-Draft (NCP + NWP core specs)
- [ ] NPS 1.0 spec freeze
- [ ] ISO/IEC JTC 1 evaluation
- [ ] Tier-3 MatrixTensor specification

---

## Milestone Dependency Graph

```
Phase 0                Phase 1                  Phase 2             Phase 3
──────                 ────────                 ────────            ────────
[spec skeleton]
    │
    ├──→ [NPS.Core] ──→ [NWP Memory/Action] ──→ [Complex Node]
    │         │                  │              [mcp-ingress] ──→ [framework integrations]
    │    [NIP CA OSS] ──────────────────────→  [a2a-ingress]
    │         │                               [grpc-ingress]
    └──→ [six SDKs] ─────────────────────────→ [SDK parity]
                                               [NDP DNS TXT]
                                               [NOP orchestr] [Cloud CA]──→ [PoC]
```

---

## Risk Register

| ID  | Risk | Probability | Impact | Mitigation |
|-----|------|-------------|--------|-----------|
| R01 | Spec changes force implementation rework | High | High | Phase 0 freezes spec before implementation; changes go through RFC |
| R02 | MCP ecosystem evolves quickly, breaks the ingress adapter | Medium | Medium | Version `mcp-ingress` independently |
| R03 | Token savings fall short (<30%) | Medium | High | Benchmark from Phase 1 (actual: 45 %); AnchorFrame hit rate is the lever |
| R04 | NIP CA private-key incident | Low | Critical | Reserve HSM interface; enforce annual key rotation |
| R05 | Competitor reaches similar positioning first | Medium | Medium | NPS differentiates on Token Economy; accelerate OSS release |
| R06 | Phase 3 PoC partner resources fall through | Medium | Medium | Backup: internal demo datasets in lieu of real partners |
| R07 | W3C/IETF cycle too long | High | Low | Pursue de-facto-standard path (GitHub adoption) before formal RFC |
| R08 | IANA PEN assignment delayed | Medium | Low | RFC-0002 ships with provisional OID; IANA PEN is non-blocking for alpha releases |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
