English | [中文版](./README.cn.md)

# Neural Protocol Suite (NPS) — Protocol Specification

> **Version:** 1.0.0-alpha.1 | **Status:** Draft | **License:** Apache 2.0
>
> Copyright 2026 INNO LOTUS PTY LTD — LabAcacia Open Source

NPS (Neural Protocol Suite) is a purpose-built protocol family for AI agents and neural models — designed to replace the HTTP/REST stack with a semantics-first, agent-native wire protocol.

---

## Why NPS?

Existing web protocols (HTTP, REST, GraphQL) were built for human browsers. AI agents consume them with significant overhead:

| Problem | Impact |
|---------|--------|
| Schema repeated on every response | Token waste, increased latency |
| No native agent identity concept | Bolted-on auth, no trust chain |
| Semantic interpretation left to agents | Prompt complexity, hallucination risk |
| Single-request model | No native streaming or task orchestration |

NPS solves all four at the wire level.

---

## Protocol Family

| Protocol | Analogue | Version | Description |
|----------|----------|---------|-------------|
| **NCP** — Neural Communication Protocol | Wire / Framing | v0.4 | Binary frame format, encoding tiers, streaming |
| **NWP** — Neural Web Protocol | HTTP | v0.4 | Semantic request/response, AnchorFrame schema cache |
| **NIP** — Neural Identity Protocol | TLS / PKI | v0.2 | Ed25519 identity, certificate lifecycle, CA |
| **NDP** — Neural Discovery Protocol | DNS | v0.2 | Node announcement, graph traversal, capability lookup |
| **NOP** — Neural Orchestration Protocol | SMTP / MQ | v0.3 | DAG task orchestration, delegation, streaming results |

**Dependency chain:** `NCP ← NWP ← NIP ← NDP` / `NCP + NWP + NIP ← NOP`

---

## Documentation

### Protocol Specifications

| Document | Description |
|----------|-------------|
| [NPS-0 Overview](./doc/protocols/NPS-0-Overview.md) | Suite overview — start here |
| [NPS-1 NCP](./doc/protocols/NPS-1-NCP.md) | Wire format, frame headers, encoding tiers |
| [NPS-2 NWP](./doc/protocols/NPS-2-NWP.md) | Neural Web Protocol (中文) |
| [NPS-2 NWP (EN)](./doc/protocols/NPS-2-NWP.en.md) | Neural Web Protocol (English) |
| [NPS-3 NIP](./doc/protocols/NPS-3-NIP.md) | Neural Identity Protocol |
| [NPS-4 NDP](./doc/protocols/NPS-4-NDP.md) | Neural Discovery Protocol |
| [NPS-5 NOP](./doc/protocols/NPS-5-NOP.md) | Neural Orchestration Protocol (中文) |
| [NPS-5 NOP (EN)](./doc/protocols/NPS-5-NOP.en.md) | Neural Orchestration Protocol (English) |

### Reference Documents

| Document | Description |
|----------|-------------|
| [Frame Registry](./doc/frame-registry.yaml) | Machine-readable frame type registry |
| [Error Codes](./doc/error-codes.md) | Unified error code namespace |
| [Status Codes](./doc/status-codes.md) | NPS native status codes + HTTP mapping |
| [Token Budget](./doc/token-budget.md) | NPT token budget specification |
| [Roadmap](./doc/NPS-Roadmap.md) | Phase 0–4 development roadmap |

### Service Specifications

| Document | Description |
|----------|-------------|
| [AaaS Profile](./doc/services/NPS-AaaS-Profile.md) | Agent-as-a-Service compliance profile (中文) |
| [AaaS Profile (EN)](./doc/services/NPS-AaaS-Profile.en.md) | Agent-as-a-Service compliance profile (English) |

---

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Default port | 17433 | Shared across all protocols; frame type routes internally |
| Transport | HTTP overlay + native mode | Overlay: firewall-friendly; Native: low-latency |
| Schema ownership | Node publishes AnchorFrame | Node owns its data model; agents reference |
| Token metering | NPT (NPS Token) | Unified cross-model metering unit |
| Primary signature | Ed25519 | Performance-first for high-frequency agent verification |
| Default encoding | MsgPack (Tier-2) | ~60% size reduction vs JSON in production |
| Max DAG nodes | 32 | DoS prevention |
| Max delegation depth | 3 | Prevents infinite delegation chains |

---

## SDK Implementations

| Language | Package | Repository |
|----------|---------|------------|
| .NET / C# | `NPS.Core` (NuGet) | https://github.com/labacacia/NPS-sdk-dotnet |
| Python | `nps-sdk` (PyPI) | https://github.com/labacacia/NPS-sdk-py |
| Go | `github.com/labacacia/nps-sdk-go` | https://github.com/labacacia/NPS-sdk-go |
| TypeScript | `@labacacia/nps-sdk` (npm) | https://github.com/labacacia/NPS-sdk-ts |
| Java | `com.labacacia:nps-java` (Maven Central) | https://github.com/labacacia/NPS-sdk-java |
| Rust | `nps-sdk` (crates.io) | https://github.com/labacacia/NPS-sdk-rust |

---

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for contribution guidelines and the RFC process for breaking changes.

## License

Copyright 2026 INNO LOTUS PTY LTD

Licensed under the Apache License, Version 2.0. See [LICENSE](./LICENSE) for details.
