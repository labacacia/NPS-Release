# Roadmap

> English | [中文版](roadmap.cn.md)

NPS is on a four-phase path from draft specification to formal standard. The current release — **v1.0.0-alpha.2** — closes out Phase 1 for six SDKs and delivers the Phase 2 MCP + A2A compatibility bridges.

---

## Cadence

Each phase breaks down into three segments:

```
① Spec Sprint   (2 weeks)    — freeze all specs for the phase
② Impl Sprint   (6–8 weeks)  — implement, test, document
③ Review Gate   (1 week)     — community / internal review before advancing
```

### Version convention

| Tag           | Meaning                                  |
|---------------|------------------------------------------|
| `v0.x-draft`  | internal draft; breaking changes allowed |
| `v0.x-alpha`  | public preview; API not stable           |
| `v0.x-beta`   | feature-complete; external testing       |
| `v1.0`        | spec frozen; production-ready            |

---

## Phase 0 — Spec Unification (2026 Q2) — ✅ done

Goal: establish the full NPS skeleton, unify the frame namespace, publish a v0.2-draft for community comment.

- ✅ `NPS-0-Overview.md` v0.3
- ✅ `NPS-1-NCP.md` v0.5 (dual transport, configurable frame size, ErrorFrame)
- ✅ `NPS-2-NWP.md` v0.5 (Node-published AnchorFrame, NPT)
- ✅ `NPS-3-NIP.md` v0.3 (metadata field, NPS status codes)
- ✅ `NPS-4-NDP.md` v0.3
- ✅ `NPS-5-NOP.md` v0.4
- ✅ `frame-registry.yaml` v0.5 (with ErrorFrame 0xFE)
- ✅ `error-codes.md` v0.5 + `status-codes.md` v0.2 + `token-budget.md` v0.2
- ✅ `services/NPS-AaaS-Profile.md` v0.2 (Gateway Node, Vector Proxy Layer, L1/L2/L3 compliance)

---

## Phase 1 — Core Implementation (2026 Q3) — ✅ shipped

Goal: NCP + NWP + NIP + NDP + NOP production-ready in the six reference languages. NIP CA Server OSS in all six.

### SDKs

| Language   | Package                            | Status                 |
|------------|------------------------------------|------------------------|
| .NET       | `LabAcacia.NPS.Core` (+ `.NWP` / `.NIP` / `.NDP` / `.NOP`) | ✅ v1.0.0-alpha.2 shipped |
| Python     | `nps-lib`                          | ✅ v1.0.0-alpha.2 shipped (162 tests, 97% coverage) |
| TypeScript | `@labacacia/nps-sdk`               | ✅ v1.0.0-alpha.2 shipped (264 tests) |
| Java       | `com.labacacia:nps-sdk`            | ✅ v1.0.0-alpha.2 shipped (87 tests) |
| Rust       | `nps-sdk`                          | ✅ v1.0.0-alpha.2 shipped (88 tests) |
| Go         | `github.com/labacacia/NPS-sdk-go`  | ✅ v1.0.0-alpha.2 shipped (75 tests) |

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

- ✅ `NPS.Core` — frame codec, AnchorCache, EXT header
- ✅ `NPS.NWP` — Memory Node middleware (SQL Server / PostgreSQL), 284 integration tests
- ✅ `NPS.NOP` — DAG validator + orchestration engine, delegation-chain depth limit (NPS-5 §8.2), callback_url SSRF protection (§8.4), exponential backoff retry, 429 tests
- ✅ `NPS.NIP` — CA library: keygen, issuance / revocation, OCSP, CRL

Completion bar:
- ✅ Unit coverage ≥ 90 % across SDKs
- ✅ Memory Node `QueryFrame` round-trip integration tests passing
- ✅ NIP CA Server one-command Docker Compose boot documented

---

## Phase 2 — Ecosystem Expansion (2026 Q4) — 🚧 in progress

Goal: adapters to existing ecosystems (MCP, A2A), richer SDK examples, Tier-2 MsgPack production hardening.

- ✅ TypeScript SDK shipped (Phase 2 scope advanced to Phase 1)
- ✅ Go SDK shipped (Phase 2 scope advanced to Phase 1)
- ✅ `compat/mcp-bridge/` — NWP Memory/Action/Complex Node ↔ MCP 2024-11-05 adapter (`LabAcacia.McpBridge` v1.0.0-alpha.2)
- ✅ `compat/a2a-bridge/` — NOP `TaskFrame` ↔ A2A Task adapter (`LabAcacia.A2aBridge` v1.0.0-alpha.2)
- ✅ Tier-2 MsgPack wire-size benchmark (aggregate **63.6 %** reduction vs JSON — steady-state frames 61.9 %–88.4 %)
- ✅ Token-savings benchmark (aggregate **45.0 %** NPT reduction vs REST — above Phase 1 ≥ 30 % bar)
- ⬜ First reference product — **NPS Studio** (human visual debugger) + **NPS Probe** (Agent Coder conformance CLI)

Completion bar:
- ⬜ `NDP.ResolveFrame` resolves `nwp://` to a physical endpoint via DNS TXT
- ✅ NOP orchestrator executes a 3-node DAG end-to-end (`impl/dotnet/samples/NPS.Samples.NopDag/`)
- ✅ Claude Desktop talks to an NWP Memory Node through `mcp-bridge`

---

## Phase 3 — Ecosystem Validation (2027 Q1–Q2)

Goal: real-world PoCs, NPS Cloud CA v1.0, the beginnings of de-facto standard status.

- ⬜ **NPS Cloud CA v1.0** — multi-region HA, real-time OCSP, Professional Plan
- ⬜ LangChain / AutoGen / CrewAI integration packages
- ⬜ FinTech PoC (Open Banking scenario, cross-org `TrustFrame`)
- ⬜ Connected-vehicle PoC (device NIDs, `StreamFrame` real-time telemetry)
- ⬜ Public token-savings benchmark report
- ⬜ NIP CA Server OSS v1.0 (PostgreSQL + Web Admin UI)
- ⬜ GitHub stars ≥ 500

---

## Phase 4 — Standardization (2027 Q3 onward)

Goal: move NPS toward W3C / IETF formal standards; freeze NPS 1.0.

- ⬜ Joint vendor support statement (≥ 3 vendors)
- ⬜ W3C WebAI Community Group proposal
- ⬜ IETF Internet-Draft (NCP + NWP core)
- ⬜ NPS 1.0 spec freeze
- ⬜ ISO/IEC JTC 1 evaluation
- ⬜ Tier-3 MatrixTensor specification

---

## Risk register (excerpt)

| ID  | Risk | P | Impact | Mitigation |
|-----|------|---|--------|-----------|
| R01 | Spec changes force implementation rework | High | High | Freeze spec before code; breaking changes via RFC |
| R03 | Token savings < 30 % in practice | Med | High | Measure from Phase 1; AnchorFrame hit rate is the lever |
| R04 | NIP CA key compromise | Low | Critical | HSM interface reserved; annual rotation enforced |
| R07 | W3C / IETF standardization slow | High | Low | Pursue de-facto adoption via OSS; formal RFC is secondary |

Full register: [NPS-Roadmap.md](https://github.com/labacacia/NPS-Release/blob/main/spec/NPS-Roadmap.md).

---

## Next

- [Overview](overview.md) — what NPS is
- [Protocols](protocols.md) — the five layers
- [SDKs](sdks.md) — pick a language
