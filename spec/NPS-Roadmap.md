English | [中文版](./NPS-Roadmap.cn.md)

# NPS Roadmap

**Version**: 0.4  
**Date**: 2026-04-30  
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
- [x] `NPS-2-NWP.md` v0.8 (Node-published AnchorFrame, NPT, topology queries)
- [x] `NPS-3-NIP.md` v0.5 (metadata field, NPS status codes, X.509 NID prototype)
- [x] `NPS-4-NDP.md` v0.5
- [x] `NPS-5-NOP.md` v0.4
- [x] `frame-registry.yaml` v0.9 (with ErrorFrame 0xFE)
- [x] `error-codes.md` v1.0 (with NPS status-code mapping, NIP cert error codes, NWP topology error codes)
- [x] `status-codes.md` v0.2 (NPS native status codes + HTTP mapping)
- [x] `token-budget.md` v0.2 (NPT metering + tokenizer resolution chain)
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
| `nps-gateway`  | Phase 1 skeleton (Internet ingress deferred) |
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
- [x] Token-savings benchmark (aggregate 45.0 % NPT reduction vs REST)
- [x] NOP orchestrator executes a 3-node DAG end-to-end
- [x] Claude Desktop talks to an NWP Memory Node through `mcp-ingress`
- [ ] `NDP.ResolveFrame` resolves `nwp://` to a physical endpoint via DNS TXT
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
| **`npt_est` per-event** on `TopologyEventEnvelope` | UTF-8/4 estimate per §7.2 (SHOULD) |
| **AssuranceLevel `from_wire("")`** fix | Python, TS, Java SDKs; `""` → Anonymous |
| **Spec / doc CN sync** | `error-codes.cn.md`, `RFC-0004.cn.md`, `status-codes.cn.md` all up-to-date |

### Deferred to alpha.6

| Item | Notes |
|------|-------|
| **NPS-CR-0002 Phase 2** — server-side Anchor middleware push of topology updates | alpha.4 shipped query API; push/notify leg deferred; needs new CR for authorization model |
| Non-.NET port of NPS-CR-0002 `AnchorNodeClient` topology client | .NET reference done; Python/TS/Go/Java/Rust need port |
| Non-.NET port of NPS-RFC-0004 reputation helpers (`ReputationLogClient`) | .NET reference done; port all six SDKs |
| Non-.NET port of NPS-RFC-0003 assurance-level enforcement helpers | Wired in .NET; other SDKs have enum only, no enforcement helpers |
| **NPS-RFC-0002** promotion Draft → Proposed/Accepted | Blocked on IANA PEN assignment for `nid-assurance-level`; currently uses provisional OID `1.3.6.1.4.1.99999` |

## alpha.6 Task Queue

Tasks queued for v1.0.0-alpha.6 (not yet started):

### In-flight RFCs / CRs

| Item | Notes |
|------|-------|
| **NPS-CR-0002 Phase 2** — server-side Anchor middleware push of topology updates | alpha.5 shipped query API; push/notify leg deferred |
| **NPS-RFC-0002** promotion Draft → Proposed/Accepted | Blocked on IANA PEN assignment for `nid-assurance-level`; currently uses provisional OID `1.3.6.1.4.1.99999` |

### SDK parity gaps

| Item | Notes |
|------|-------|
| Non-.NET port of NPS-CR-0002 `AnchorNodeClient` topology client | .NET reference done; Python/TS/Go/Java/Rust need port |
| Non-.NET port of NPS-RFC-0004 reputation helpers (`ReputationLogClient`) | .NET reference done; port all six SDKs |
| Non-.NET port of NPS-RFC-0003 assurance-level enforcement helpers | Wired in .NET; other SDKs have enum only, no enforcement helpers |

### Protocol / spec items

| Item | Notes |
|------|-------|
| `NDP.ResolveFrame` DNS TXT resolution (`nwp://` → physical endpoint) | Specified; not yet implemented in any SDK |
| `nps-gateway` L2 Internet ingress (`:8080`→`:443` termination, NCP over TLS) | Skeleton only at alpha.4; L2 conformance deferred |
| `nps-runner` L3 FaaS task runtime | Skeleton only; full impl Phase 3 scope |

### Tooling

| Item | Notes |
|------|-------|
| **NPS Studio** — visual debugger for NPS frame streams | Phase 2 target; not yet started |
| **NPS Probe** — Agent Coder conformance CLI | Phase 2 target; not yet started |

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
