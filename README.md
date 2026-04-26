English | [中文版](./README.cn.md)

# Neural Protocol Suite (NPS) — Protocol Specification

> **Version:** 1.0.0-alpha.3 | **Status:** Proposed | **License:** Apache 2.0
>
> Copyright 2026 INNO LOTUS PTY LTD — LabAcacia Open Source

**NPS (Neural Protocol Suite)** is a purpose-built protocol family for AI agents and neural models — designed to replace the HTTP/REST stack with a semantics-first, agent-native wire protocol.

---

## NPS Repositories

| Repo | Role | Language |
|------|------|----------|
| **[NPS-Release](https://github.com/labacacia/NPS-Release)** (this repo) | Protocol specifications — authoritative source of truth for all SDKs | Markdown / YAML |
| [NPS-sdk-dotnet](https://github.com/labacacia/NPS-sdk-dotnet) | Reference implementation — frame codec, Memory Node middleware, NIP CA | C# / .NET 10 |
| [NPS-sdk-py](https://github.com/labacacia/NPS-sdk-py) | Async Python SDK | Python 3.11+ |
| [NPS-sdk-ts](https://github.com/labacacia/NPS-sdk-ts) | Dual-format (ESM + CJS) Node/browser SDK | TypeScript |
| [NPS-sdk-java](https://github.com/labacacia/NPS-sdk-java) | JVM SDK | Java 21+ |
| [NPS-sdk-rust](https://github.com/labacacia/NPS-sdk-rust) | Async Rust SDK | Rust stable |
| [NPS-sdk-go](https://github.com/labacacia/NPS-sdk-go) | Go SDK | Go 1.23+ |
| [nip-ca-server](https://github.com/labacacia/nip-ca-server) | NIP Certificate Authority — single-Docker self-hostable CA for issuing NID certificates | C# / .NET 10 + PostgreSQL |

The .NET SDK ships the underlying CA library (`LabAcacia.NPS.NIP`); the
standalone [`nip-ca-server`](https://github.com/labacacia/nip-ca-server)
repo wraps it as a deployable service. Five frozen reference ports
(Python / TypeScript / Java / Rust / Go) live under that repo's
`example/` directory.

---

## Why NPS?

Existing web protocols (HTTP, REST, GraphQL) were built for human browsers. AI agents consume them with significant overhead:

| Problem | Impact |
|---------|--------|
| Schema repeated on every response | Token waste, increased latency |
| No native agent identity concept | Bolted-on auth, no trust chain |
| Semantic interpretation left to agents | Prompt complexity, hallucination risk |
| Single-request model | No native streaming or task orchestration |

NPS solves all four at the wire level: one-time schema anchors, Ed25519 identity baked into every hop, semantic annotations in the frame itself, and a unified DAG-based task frame.

---

## Protocol Family

| Protocol | Analogue | Version | Summary |
|----------|----------|---------|---------|
| **NCP** — Neural Communication Protocol | Wire / Framing | v0.6 | Binary frame format, dual-tier codec (JSON / MsgPack), streaming |
| **NWP** — Neural Web Protocol | HTTP | v0.7 | Semantic request/response, AnchorFrame schema cache, Memory / Action / Anchor / Bridge nodes |
| **NIP** — Neural Identity Protocol | TLS / PKI | v0.5 | Ed25519 identity, certificate lifecycle, CA, OCSP, CRL |
| **NDP** — Neural Discovery Protocol | DNS | v0.5 | Node announcement, signed records, graph traversal |
| **NOP** — Neural Orchestration Protocol | SMTP / MQ | v0.4 | DAG task orchestration, delegation, streaming results |

**Dependency chain:** `NCP ← NWP ← NIP ← NDP` / `NCP + NWP + NIP ← NOP`

---

## Documentation

### Protocol Specifications

| Document | Description |
|----------|-------------|
| [NPS-0 Overview](./spec/NPS-0-Overview.md) | Suite overview — start here |
| [NPS-1 NCP](./spec/NPS-1-NCP.md) | Wire format, frame header, encoding tiers |
| [NPS-2 NWP](./spec/NPS-2-NWP.md) | Neural Web Protocol |
| [NPS-3 NIP](./spec/NPS-3-NIP.md) | Neural Identity Protocol |
| [NPS-4 NDP](./spec/NPS-4-NDP.md) | Neural Discovery Protocol |
| [NPS-5 NOP](./spec/NPS-5-NOP.md) | Neural Orchestration Protocol |

### Reference Documents

| Document | Description |
|----------|-------------|
| [Frame Registry](./spec/frame-registry.yaml) | Machine-readable frame type registry |
| [Error Codes](./spec/error-codes.md) | Unified protocol error code namespace |
| [Status Codes](./spec/status-codes.md) | NPS native status codes + HTTP mapping |
| [Token Budget](./spec/token-budget.md) | NPT token budget specification |
| [Roadmap](./spec/NPS-Roadmap.md) | Phase 0–4 development roadmap |

### Service Specifications

| Document | Description |
|----------|-------------|
| [AaaS Profile](./spec/services/NPS-AaaS-Profile.md) | Agent-as-a-Service compliance profile (Anchor Node, Bridge Node, Vector Proxy Layer, L1/L2/L3 compliance) |

---

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Default port | **17433** | Shared across all protocols; frame type routes internally |
| Transport | HTTP overlay + native mode | Overlay: firewall-friendly; Native: low-latency |
| Schema ownership | Node publishes AnchorFrame | Node owns its data model; agents reference by id |
| Token metering | **NPT** (NPS Token) | Unified cross-model metering unit |
| Primary signature | **Ed25519** | Performance-first for high-frequency agent verification |
| Default encoding | **MsgPack** (Tier-2) | ~60% size reduction vs JSON in production |
| Default frame size | 64 KB (EXT=0) | Extended: 4 GB (EXT=1) for large payloads |
| Max DAG nodes | 32 | DoS prevention |
| Max delegation depth | 3 | Prevents infinite delegation chains |
| Max graph traversal depth | 5 | X-NWP-Depth upper bound |
| AnchorFrame TTL | 3600 s | Balances cache hit rate with schema freshness |

---

## Frame Type Namespace (Quick Reference)

| Range | Protocol | Frames |
|-------|----------|--------|
| `0x01–0x0F` | **NCP** | Anchor(0x01), Diff(0x02), Stream(0x03), Caps(0x04), Align(0x05, deprecated), Hello(0x06) |
| `0x10–0x1F` | **NWP** | Query(0x10), Action(0x11) |
| `0x20–0x2F` | **NIP** | Ident(0x20), Trust(0x21), Revoke(0x22) |
| `0x30–0x3F` | **NDP** | Announce(0x30), Resolve(0x31), Graph(0x32) |
| `0x40–0x4F` | **NOP** | Task(0x40), Delegate(0x41), Sync(0x42), AlignStream(0x43) |
| `0xFE`      | System   | ErrorFrame — unified error across all layers |

See [`spec/frame-registry.yaml`](./spec/frame-registry.yaml) for the complete machine-readable registry.

---

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for contribution guidelines and the RFC process for breaking changes.

Issue prefixes:
- `spec:` — specification questions and design discussion
- `impl:` — reference implementation bugs
- `sdk:` — SDK-specific (Python / TypeScript / …)
- `docs:` — documentation improvements

Breaking changes to `spec/` **must** open an RFC issue first.

---

## License

Copyright 2026 INNO LOTUS PTY LTD

Licensed under the Apache License, Version 2.0. See [LICENSE](./LICENSE) for details.

- **Specifications & reference implementations** — LabAcacia open source
- **Commercial services (NPS Cloud)** — operated by INNO LOTUS PTY LTD
- **Primary author** — Ori Lynn
