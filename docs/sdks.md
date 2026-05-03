# SDKs

> English | [中文版](sdks.cn.md)

Six official SDKs — each implementing all five protocols (NCP + NWP + NIP + NDP + NOP) — at suite version **1.0.0-alpha.5.2**.

---

## Language matrix

| Language | Package | Min version | Repo | Wiki deep-dive |
|----------|---------|-------------|------|----------------|
| .NET       | `LabAcacia.NPS.Core` (+ `.NWP` / `.NIP` / `.NDP` / `.NOP`) | .NET 10     | [NPS-sdk-dotnet](https://github.com/labacacia/NPS-sdk-dotnet) | [Wiki: SDK-dotnet](https://github.com/labacacia/NPS-Release/wiki/SDK-dotnet) |
| Python     | `nps-lib`                                                    | 3.11        | [NPS-sdk-py](https://github.com/labacacia/NPS-sdk-py)         | [Wiki: SDK-Python](https://github.com/labacacia/NPS-Release/wiki/SDK-Python) |
| TypeScript | `@labacacia/nps-sdk`                                         | Node 22     | [NPS-sdk-ts](https://github.com/labacacia/NPS-sdk-ts)         | [Wiki: SDK-TypeScript](https://github.com/labacacia/NPS-Release/wiki/SDK-TypeScript) |
| Java       | `com.labacacia.nps:nps-java`                                 | Java 21     | [NPS-sdk-java](https://github.com/labacacia/NPS-sdk-java)     | [Wiki: SDK-Java](https://github.com/labacacia/NPS-Release/wiki/SDK-Java) |
| Rust       | `nps-sdk`                                                    | Rust stable | [NPS-sdk-rust](https://github.com/labacacia/NPS-sdk-rust)     | [Wiki: SDK-Rust](https://github.com/labacacia/NPS-Release/wiki/SDK-Rust) |
| Go         | `github.com/labacacia/NPS-sdk-go`                            | Go 1.25     | [NPS-sdk-go](https://github.com/labacacia/NPS-sdk-go)         | [Wiki: SDK-Go](https://github.com/labacacia/NPS-Release/wiki/SDK-Go) |

For install commands, minimal examples, and per-feature coverage tables, see the per-language Wiki pages above or [SDK-Quickstart](https://github.com/labacacia/NPS-Release/wiki/SDK-Quickstart) for a language-agnostic walkthrough.

---

## NIP CA Server

Standalone deployable Certificate Authority for the Neural Identity Protocol (NPS-3 §8). Independently versioned from the SDKs since `v1.0.0-alpha.3`.

| Repo | Stack | Quickstart |
|------|-------|------------|
| [labacacia/nip-ca-server](https://github.com/labacacia/nip-ca-server) | C# / ASP.NET Core 10, PostgreSQL or SQLite, single-Docker | `docker compose up -d` |

For operator guides and embedding options (SQLite vs PostgreSQL) see [Wiki: NIP-CA-Server-Ops](https://github.com/labacacia/NPS-Release/wiki/NIP-CA-Server-Ops).

---

## NPS Daemons

Reference deployment binaries for the standard three-layer NPS topology, currently at `v1.0.0-alpha.5.2`.

| Repo | Daemons | Quickstart |
|------|---------|------------|
| [labacacia/nps-daemons](https://github.com/labacacia/nps-daemons) | `npsd` (L1, :17433) · `nps-runner` (L1 FaaS) · `nps-gateway` (L2, :8080) · `nps-registry` (L2 NDP, :17436) | `docker compose up -d` |

The Layer-3 trust-anchor daemons (`nps-cloud-ca` and `nps-ledger`) are private under the `innolotus` org and ship publicly with NPS Cloud GA (2027 Q1+). For operator and architecture detail see [Wiki: Operators-QuickStart](https://github.com/labacacia/NPS-Release/wiki/Operators-QuickStart).

---

## Which SDK should you pick?

| You are… | Suggested SDK |
|----------|---------------|
| Writing an agent that calls NPS nodes | **Python** or **TypeScript** — fastest iteration |
| Building a Memory Node for an existing service | match the service language (.NET / Java / Go for enterprise; Python / TS for startups) |
| Writing a high-throughput orchestrator (NOP) | **Rust** or **Go** |
| Bundling with a React frontend | **TypeScript** (dual ESM + CJS) |
| Shipping to a JVM-heavy environment | **Java 21** |

All SDKs produce wire-identical frames — mix languages freely (e.g. a Python agent calling a Rust Memory Node, orchestrated by a Go NOP server).

---

## Next

- [Overview](overview.md) — what NPS is, why it exists
- [Protocols](protocols.md) — the five layers
- [Roadmap](roadmap.md) — what's shipped and what's next
- [Get Started](get-started.md) — choose your path by audience

---

📖 For tutorials, references, and operator guides, see the [NPS Wiki](https://github.com/labacacia/NPS-Release/wiki).
