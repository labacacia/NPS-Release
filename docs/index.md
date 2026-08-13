# NPS — A Wire Protocol for AI Agents

> **Neural Protocol Suite** — a complete internet protocol stack purpose-built for AI agents and neural models.
>
> Candidate 1.0.0-alpha.18 · latest suite release 1.0.0-alpha.17 · Apache 2.0 · [中文版](index.cn.md)

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
pip install nps-lib==1.0.0a13

# TypeScript
npm install @labacacia/nps-sdk@alpha

# Rust
cargo add nps-sdk@=1.0.0-alpha.17

# Go
go get github.com/labacacia/NPS-sdk-go@v1.0.0-alpha.17

# Java (Gradle)
implementation("com.labacacia.nps:nps-java:1.0.0-alpha.17")

# .NET
dotnet add package LabAcacia.NPS.Core --version 1.0.0-alpha.17
```

> npm note: `@labacacia/nps-sdk@1.0.0-alpha.17` includes the package build output; the `alpha` dist-tag resolves to the latest published alpha.

---

## Explore

- [**Overview**](overview.md) — what NPS is, why it exists, who it's for
- [**Protocols**](protocols.md) — the five layers (NCP / NWP / NIP / NDP / NOP) at a glance
- [**SDKs**](sdks.md) — install and quick-start for all six languages
- [**Roadmap**](roadmap.md) — Phase 0 → Phase 4

---

## Status

**v1.0.0-alpha.17** — the published SDK/spec line tracks NCP v0.11, NWP v0.20, NIP v0.13, NDP v0.12, and NOP v0.9. It delivers portable server/conformance profiles across all six SDKs, multi-Anchor HA, bidirectional Bridge serving, NIP Phase-3 enforcement, corrected NOP wire keys, and hardened daemon runtime behavior. The unreleased alpha.18 candidate adds NWP 0.21 stateful LLM context semantics and related SDK/runtime work. **NIP CA Server** and the public **NPS Daemons** bundle remain independently published at alpha.16 while their alpha.18 candidates are prepared.

---

## Links

- [GitHub — NPS-Release](https://github.com/labacacia/NPS-Release) · the protocol specifications (authoritative)
- [CONTRIBUTING](https://github.com/labacacia/NPS-Release/blob/main/CONTRIBUTING.md) · how to propose changes
- [LICENSE](https://github.com/labacacia/NPS-Release/blob/main/LICENSE) · Apache 2.0

Copyright 2026 INNO LOTUS PTY LTD — LabAcacia Open Source Lab.

---

📖 For tutorials, references, and operator guides, see the [NPS Wiki](https://github.com/labacacia/NPS-Release/wiki).
