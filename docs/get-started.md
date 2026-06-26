# Get Started

> English | [中文版](get-started.cn.md)

NPS is a multi-layer protocol suite. Where you start depends on what you're building.

The latest published package set is `1.0.0-alpha.14`. The documentation and
SDK notes are aligned with the alpha.14 release boundary.

---

## I'm writing an AI agent or model client

You need an **SDK**. Pick your language and follow the quickstart on the Wiki.

| Language | Install | Wiki quickstart |
|----------|---------|-----------------|
| Python     | `pip install nps-lib==1.0.0a13`                          | [SDK-Python](https://github.com/labacacia/NPS-Release/wiki/SDK-Python) |
| TypeScript | `npm install @labacacia/nps-sdk@alpha`                   | [SDK-TypeScript](https://github.com/labacacia/NPS-Release/wiki/SDK-TypeScript) |
| Rust       | `cargo add nps-sdk@=1.0.0-alpha.14`                       | [SDK-Rust](https://github.com/labacacia/NPS-Release/wiki/SDK-Rust) |
| Go         | `go get github.com/labacacia/NPS-sdk-go@v1.0.0-alpha.14`  | [SDK-Go](https://github.com/labacacia/NPS-Release/wiki/SDK-Go) |
| Java       | `implementation("com.labacacia.nps:nps-java:1.0.0-alpha.14")` | [SDK-Java](https://github.com/labacacia/NPS-Release/wiki/SDK-Java) |
| .NET       | `dotnet add package LabAcacia.NPS.Core --version 1.0.0-alpha.14` | [SDK-dotnet](https://github.com/labacacia/NPS-Release/wiki/SDK-dotnet) |

> npm note: `@labacacia/nps-sdk@1.0.0-alpha.14` fixed the earlier alpha.11 tarball issue; the `alpha` dist-tag resolves to the latest published alpha.

---

## I'm deploying an NPS node or infrastructure

You need the **NPS Daemons** bundle (`npsd` + `nps-runner` + `nps-ingress` + `nps-registry`).

`nps-ingress` is a process-level Internet ingress daemon name, not the retired
NWP **Gateway Node** logical role. CR-0001 replaced that logical role with
**Anchor Node** and **Bridge Node**.

→ [Wiki: Operators-QuickStart](https://github.com/labacacia/NPS-Release/wiki/Operators-QuickStart)

---

## I'm building a protocol integration or bridge

Read the protocol specs (linked from [Protocols](protocols.md)), then pick a compat adapter:

- `compat/mcp-ingress` — MCP → NPS translation
- `compat/a2a-ingress` — A2A → NPS translation
- `compat/grpc-ingress` — gRPC → NPS translation

→ [Wiki: Protocol-Designer-Guide](https://github.com/labacacia/NPS-Release/wiki/Protocol-Designer-Guide)

---

## I want to understand NPS before writing any code

Start with [Overview](overview.md) (5 min read), then [Protocols](protocols.md) for the five-layer architecture.

---

## I want to contribute to NPS

See [CONTRIBUTING](https://github.com/labacacia/NPS-Release/blob/main/CONTRIBUTING.md) for the process, and [Wiki: Contributors-Guide](https://github.com/labacacia/NPS-Release/wiki/Contributors-Guide) for conventions.

---

📖 For tutorials, references, and operator guides, see the [NPS Wiki](https://github.com/labacacia/NPS-Release/wiki).
