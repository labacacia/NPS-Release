# Roadmap

> English | [中文版](roadmap.cn.md)

NPS is on a four-phase path from draft specification to formal standard. The current release — **v1.0.0-alpha.4** — completes Phase 1 and makes significant progress on Phase 2: Anchor topology queries (CR-0002), X.509/ACME NID prototype (RFC-0002), SQLite-backed registry and ledger daemons, and gRPC/A2A/MCP adapters across all six languages.

---

## Version convention

| Tag           | Meaning                                  |
|---------------|------------------------------------------|
| `v0.x-draft`  | internal draft; breaking changes allowed |
| `v0.x-alpha`  | public preview; API not stable           |
| `v0.x-beta`   | feature-complete; external testing       |
| `v1.0`        | spec frozen; production-ready            |

---

## Phase 0 — Spec Unification (2026 Q2) — ✅ done

Established the full NPS spec skeleton across all five protocols (NCP / NWP / NIP / NDP / NOP), unified the frame namespace, published the AaaS and Node conformance profiles, and released public repositories.

---

## Phase 1 — Core Implementation (2026 Q3) — ✅ shipped

All five protocols production-ready in six reference SDKs (.NET / Python / TypeScript / Java / Rust / Go), with NIP CA Server reference deployments in all six languages. Key milestones:

- NPS-RFC-0001 (NCP connection preamble), NPS-RFC-0003 (agent identity assurance levels), NPS-CR-0001 (Anchor/Bridge Node split), NPS-CR-0002 (Anchor topology queries) — all shipped
- NPS-RFC-0002 (X.509 NID + ACME) prototype across all six SDKs; promotion to Accepted pending IANA PEN
- NPS-RFC-0004 (CT-style NID reputation ledger) Phase 1+2: SQLite + Merkle tree + STH + inclusion proofs
- `npsd` L1 daemon with sub-NID issuance and per-NID inbox queues
- Token-savings: **45.0 %** NPT reduction vs REST · Wire-size: **63.6 %** reduction vs JSON

---

## Phase 2 — Ecosystem Expansion (2026 Q4) — 🚧 in progress

Adapters to MCP, A2A, and gRPC ecosystems; Tier-2 MsgPack production hardening; reference tooling.

- ✅ MCP, A2A, and gRPC ingress adapters shipped (v1.0.0-alpha.4)
- ✅ Tier-2 MsgPack and token-savings benchmarks published
- ✅ NOP orchestrator executes multi-node DAGs; Claude Desktop integration via `mcp-ingress` verified
- ⬜ `NDP.ResolveFrame` DNS TXT resolution (`nwp://` → physical endpoint)
- ⬜ **NPS Studio** (visual frame debugger) + **NPS Probe** (conformance CLI)

---

## Phase 3 — Ecosystem Validation (2027 Q1–Q2)

Real-world deployments, NPS Cloud CA v1.0, and the beginnings of de-facto standard status.

- ⬜ NPS Cloud CA v1.0 — multi-region HA, real-time OCSP
- ⬜ LangChain / AutoGen / CrewAI integration packages
- ⬜ FinTech and connected-vehicle PoCs
- ⬜ NIP CA Server OSS v1.0 (PostgreSQL + Web Admin UI)
- ⬜ Public token-savings benchmark report
- ⬜ GitHub stars ≥ 500

---

## Phase 4 — Standardization (2027 Q3 onward)

Formal W3C / IETF standardization and NPS 1.0 spec freeze.

- ⬜ W3C WebAI Community Group proposal
- ⬜ IETF Internet-Draft (NCP + NWP core)
- ⬜ NPS 1.0 spec freeze
- ⬜ ISO/IEC JTC 1 evaluation

---

## Next

- [Overview](overview.md) — what NPS is
- [Protocols](protocols.md) — the five layers
- [SDKs](sdks.md) — pick a language
