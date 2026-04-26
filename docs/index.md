# NPS — A Wire Protocol for AI Agents

> **Neural Protocol Suite** — a complete internet protocol stack purpose-built for AI agents and neural models.
>
> Version 1.0.0-alpha.3 · Apache 2.0 · [中文版](index.cn.md)

---

## The problem

Today's AI agents consume the web through HTTP, REST, and HTML — protocols designed for **human browsers**, not for models reasoning at token scale.

| Problem | Cost |
|---------|------|
| Schema repeats on every response | Token waste, latency |
| No native agent identity | Bolted-on auth, no trust chain |
| Semantic interpretation left to the agent | Prompt bloat, hallucinations |
| Single-request model | No native streaming or task orchestration |

## The answer

NPS solves all four **at the wire level**:

- **One-time schema anchors** — servers publish an `AnchorFrame` once; agents reference it by content-addressed id
- **Ed25519 identity on every hop** — `NipIdentity` is first-class, not an afterthought
- **Semantic annotations in the frame itself** — not in prose
- **Unified DAG task frame** — orchestrate multi-agent workflows without re-inventing MQ or Temporal

---

## Try it

```bash
# Python
pip install nps-lib==1.0.0a2

# TypeScript
npm install @labacacia/nps-sdk@1.0.0-alpha.3

# Rust
cargo add nps-sdk@=1.0.0-alpha.3

# Go
go get github.com/labacacia/NPS-sdk-go@v1.0.0-alpha.3

# Java (Gradle)
implementation("com.labacacia:nps-sdk:1.0.0-alpha.3")

# .NET
dotnet add package LabAcacia.NPS.Core --version 1.0.0-alpha.3
```

---

## Explore

- [**Overview**](overview.md) — what NPS is, why it exists, who it's for
- [**Protocols**](protocols.md) — the five layers (NCP / NWP / NIP / NDP / NOP) at a glance
- [**SDKs**](sdks.md) — install and quick-start for all six languages
- [**Roadmap**](roadmap.md) — Phase 0 → Phase 4

---

## Status

**v1.0.0-alpha.3** — Phase 1 / Phase 2 synchronized release. All 11 spec documents at `Proposed` (NCP v0.6, NWP v0.7, NIP v0.5, NDP v0.5, NOP v0.4). Reference implementations in **.NET**, **Python**, **TypeScript**, **Java**, **Rust**, **Go** — full NCP + NWP + NIP + NDP + NOP coverage. **NIP CA Server** is now released as a standalone repo at [`labacacia/nip-ca-server`](https://github.com/labacacia/nip-ca-server). **NPS Daemons** (`npsd` + `nps-runner` + `nps-gateway` + `nps-registry` — Layer 1 + Layer 2 of the standard topology) ship together as a bundle at [`labacacia/nps-daemons`](https://github.com/labacacia/nps-daemons). Layer-3 trust-anchor daemons (`nps-cloud-ca`, `nps-ledger`) are private under the `innolotus` org and ship publicly with NPS Cloud GA (2027 Q1+).

---

## Links

- [GitHub — NPS-Release](https://github.com/labacacia/NPS-Release) · the protocol specifications (authoritative)
- [CONTRIBUTING](https://github.com/labacacia/NPS-Release/blob/main/CONTRIBUTING.md) · how to propose changes
- [LICENSE](https://github.com/labacacia/NPS-Release/blob/main/LICENSE) · Apache 2.0

Copyright 2026 INNO LOTUS PTY LTD — LabAcacia Open Source Lab.
