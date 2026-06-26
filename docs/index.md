# NPS — A Wire Protocol for AI Agents

> **Neural Protocol Suite** — a complete internet protocol stack purpose-built for AI agents and neural models.
>
> Version 1.0.0-alpha.14 · latest suite release 1.0.0-alpha.14 · Apache 2.0 · [中文版](index.cn.md)

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
cargo add nps-sdk@=1.0.0-alpha.14

# Go
go get github.com/labacacia/NPS-sdk-go@v1.0.0-alpha.14

# Java (Gradle)
implementation("com.labacacia.nps:nps-java:1.0.0-alpha.14")

# .NET
dotnet add package LabAcacia.NPS.Core --version 1.0.0-alpha.14
```

> npm note: `@labacacia/nps-sdk@1.0.0-alpha.14` fixed the earlier alpha.11 tarball issue; the `alpha` dist-tag resolves to the latest published alpha.

---

## Explore

- [**Overview**](overview.md) — what NPS is, why it exists, who it's for
- [**Protocols**](protocols.md) — the five layers (NCP / NWP / NIP / NDP / NOP) at a glance
- [**SDKs**](sdks.md) — install and quick-start for all six languages
- [**Roadmap**](roadmap.md) — Phase 0 → Phase 4

---

## Status

**v1.0.0-alpha.14** — docs and specs now track NCP v0.8, NWP v0.14, NIP v0.10, NDP v0.9, and NOP v0.7. This release adds typed remote NIP CA clients, native-mode NWP serving helpers, TC-N1/TC-N2 conformance helpers, live revocation, native NCP TLS/mTLS hardening, signed CRL output, and transport-neutral observability. Package-manager availability is tracked per ecosystem; the .NET package bundle is attached to the GitHub Release while Nexus registry publish waits on a credential/permission update for the new Nexus URL. **NIP CA Server** lives at [`labacacia/nip-ca-server`](https://github.com/labacacia/nip-ca-server). **NPS Daemons** bundle (`npsd` + `nps-runner` + `nps-ingress` + `nps-registry`) lives at [`labacacia/nps-daemons`](https://github.com/labacacia/nps-daemons). Layer-3 trust-anchor daemons remain private under the `innolotus` org until NPS Cloud GA.

---

## Links

- [GitHub — NPS-Release](https://github.com/labacacia/NPS-Release) · the protocol specifications (authoritative)
- [CONTRIBUTING](https://github.com/labacacia/NPS-Release/blob/main/CONTRIBUTING.md) · how to propose changes
- [LICENSE](https://github.com/labacacia/NPS-Release/blob/main/LICENSE) · Apache 2.0

Copyright 2026 INNO LOTUS PTY LTD — LabAcacia Open Source Lab.

---

📖 For tutorials, references, and operator guides, see the [NPS Wiki](https://github.com/labacacia/NPS-Release/wiki).
