English | [中文版](./NPS-Roadmap.cn.md)

# NPS Roadmap

**Version**: 0.2  
**Date**: 2026-04-12  
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

## Phase 0 — Spec Unification (2026 Q2)

**Goal**: establish the full NPS spec skeleton, unify the frame namespace and naming, produce a v0.1-draft for community comment.

- [x] NPS-0-Overview.md v0.2 complete
- [x] NPS-1-NCP.md v0.3-draft complete (dual transport, configurable frame size, ErrorFrame)
- [x] NPS-2-NWP.md v0.2-draft complete (Node-published AnchorFrame, NPT)
- [x] NPS-3-NIP.md v0.2-draft complete (metadata field, NPS status codes)
- [x] NPS-4-NDP.md v0.2-draft complete (unified port, NPS status codes)
- [x] NPS-5-NOP.md v0.2-draft complete (unified port, NPS status codes)
- [x] frame-registry.yaml v0.2 complete (with ErrorFrame 0xFE)
- [x] error-codes.md v0.2 complete (with NPS status-code mapping)
- [x] status-codes.md v0.1 complete (NPS native status codes + HTTP mapping)
- [x] token-budget.md v0.1 complete (NPT metering + tokenizer resolution chain)
- [ ] LabAcacia repo made public, Discussions enabled
- [ ] v0.2-draft GitHub Release published

**Completion bar**: no empty sections in any sub-protocol doc, `frame-registry.yaml` covers every frame (including ErrorFrame), `status-codes.md` covers every error-code mapping, GitHub repo public.

---

## Phase 1 — Core Implementation (2026 Q3)

**Goal**: NCP + NWP + NIP production-ready, NIP CA Server OSS v0.1, initial C# + Python SDKs.

- [ ] NPS.Core NuGet (.NET 10, frame codec Tier-1/2, AnchorFrame cache, ErrorFrame, EXT header)
- [ ] NPS.NWP: Memory Node + Action Node (SQL Server / PostgreSQL / Redis)
- [ ] NPS.NWP: NWM manifest auto-generation, Overlay-mode middleware, NPT token metering
- [ ] NPS.NIP: IdentFrame / TrustFrame / RevokeFrame codecs
- [ ] NIP CA Server OSS v0.1 (Docker, SQLite, REST API + CLI)
- [ ] C# SDK NuGet release (v0.1.0-alpha, .NET 10)
- [ ] Python SDK initial PyPI release (v0.1.0a)

**Completion bar**:
- NPS.Core unit coverage ≥ 90%
- Memory Node QueryFrame round-trip integration tests passing
- NIP CA Server one-command Docker Compose boot documented
- Token-savings baseline: single-session vs REST ≥ 30%

---

## Phase 2 — Full Protocol Suite (2026 Q4)

**Goal**: NDP + NOP implementation, Complex Node, MCP/A2A adapters, TypeScript SDK, Tier-2 production readiness.

- [ ] NPS.NDP: AnnounceFrame / ResolveFrame / GraphFrame (DNS TXT + local Multicast)
- [ ] NPS.NOP: TaskFrame / DelegateFrame / SyncFrame / AlignStream
- [ ] Complex Node complete implementation (cross-source aggregation, subnode routing)
- [ ] MCP Bridge (NWP Memory Node → MCP Resource)
- [ ] A2A Bridge (NOP TaskFrame → A2A Task)
- [ ] TypeScript SDK npm package (v0.1.0-beta)
- [ ] Tier-2 MsgPack production validation (size ≤ 50% of JSON)

**Completion bar**:
- NDP ResolveFrame can resolve `nwp://` to a physical endpoint (DNS TXT mode)
- NOP Orchestrator can execute a 3-node DAG task
- Claude Desktop can reach an NWP Memory Node through the MCP Bridge

---

## Phase 3 — Ecosystem Validation (2027 Q1–Q2)

**Goal**: real-world PoCs, NPS Cloud CA v1.0 live, lay the groundwork for a de-facto standard.

- [ ] NPS Cloud CA v1.0 (multi-region HA, real-time OCSP, Professional Plan ¥299/month)
- [ ] LangChain / AutoGen / CrewAI integration adapter packages
- [ ] FinTech PoC (Open Banking scenario, cross-org TrustFrame)
- [ ] Connected-vehicle PoC (device NIDs, StreamFrame real-time telemetry)
- [ ] Token-savings benchmark report (publicly released)
- [ ] NIP CA Server OSS v1.0 (PostgreSQL + Web Admin UI)
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
    │         │                  │              [MCP Bridge] ──→ [framework integrations]
    │    [NIP CA OSS] ──────────────────────→  [A2A Bridge]
    │         │
    └──→ [Python SDK] ──────────────────────→  [TS SDK]
                                               [NDP]
                                               [NOP]          [Cloud CA]──→ [PoC]
```

---

## Risk Register

| ID  | Risk | Probability | Impact | Mitigation |
|-----|------|-------------|--------|-----------|
| R01 | Spec changes force implementation rework | High | High | Phase 0 freezes spec before implementation; changes go through RFC |
| R02 | MCP ecosystem evolves quickly, breaks the bridge | Medium | Medium | Version `mcp-bridge` independently |
| R03 | Token savings fall short (<30%) | Medium | High | Benchmark from Phase 1; AnchorFrame hit rate is the lever |
| R04 | NIP CA private-key incident | Low | Critical | Reserve HSM interface; enforce annual key rotation |
| R05 | Competitor reaches similar positioning first | Medium | Medium | NPS differentiates on Token Economy; accelerate OSS release |
| R06 | Phase 3 PoC partner resources fall through | Medium | Medium | Backup: internal demo datasets in lieu of real partners |
| R07 | W3C/IETF cycle too long | High | Low | Pursue de-facto-standard path (GitHub adoption) before formal RFC |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
