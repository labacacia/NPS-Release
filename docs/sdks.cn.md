# SDK

> [English](sdks.md) | 中文版

六种官方 SDK，每种均完整实现五层协议（NCP + NWP + NIP + NDP + NOP），当前套件版本 **1.0.0-alpha.5.2**。

---

## 语言矩阵

| 语言 | 包名 | 最低版本 | 仓库 | Wiki 深度文档 |
|------|------|----------|------|--------------|
| .NET       | `LabAcacia.NPS.Core` (+ `.NWP` / `.NIP` / `.NDP` / `.NOP`) | .NET 10     | [NPS-sdk-dotnet](https://github.com/labacacia/NPS-sdk-dotnet) | [Wiki: SDK-dotnet](https://github.com/labacacia/NPS-Release/wiki/SDK-dotnet) |
| Python     | `nps-lib`                                                    | 3.11        | [NPS-sdk-py](https://github.com/labacacia/NPS-sdk-py)         | [Wiki: SDK-Python](https://github.com/labacacia/NPS-Release/wiki/SDK-Python) |
| TypeScript | `@labacacia/nps-sdk`                                         | Node 22     | [NPS-sdk-ts](https://github.com/labacacia/NPS-sdk-ts)         | [Wiki: SDK-TypeScript](https://github.com/labacacia/NPS-Release/wiki/SDK-TypeScript) |
| Java       | `com.labacacia.nps:nps-java`                                 | Java 21     | [NPS-sdk-java](https://github.com/labacacia/NPS-sdk-java)     | [Wiki: SDK-Java](https://github.com/labacacia/NPS-Release/wiki/SDK-Java) |
| Rust       | `nps-sdk`                                                    | Rust stable | [NPS-sdk-rust](https://github.com/labacacia/NPS-sdk-rust)     | [Wiki: SDK-Rust](https://github.com/labacacia/NPS-Release/wiki/SDK-Rust) |
| Go         | `github.com/labacacia/NPS-sdk-go`                            | Go 1.25     | [NPS-sdk-go](https://github.com/labacacia/NPS-sdk-go)         | [Wiki: SDK-Go](https://github.com/labacacia/NPS-Release/wiki/SDK-Go) |

安装命令、极简示例、功能覆盖对照表等深度内容，请访问上方各语言 Wiki 页面，或查阅 [SDK-Quickstart](https://github.com/labacacia/NPS-Release/wiki/SDK-Quickstart) 获取语言无关的入门教程。

---

## NIP CA Server

可独立部署的 Neural Identity Protocol（NPS-3 §8）证书颁发机构，自 `v1.0.0-alpha.3` 起从 SDK 拆出独立发布。

| 仓库 | 技术栈 | 快速开始 |
|------|--------|----------|
| [labacacia/nip-ca-server](https://github.com/labacacia/nip-ca-server) | C# / ASP.NET Core 10，PostgreSQL 或 SQLite，单 Docker | `docker compose up -d` |

运维指南及嵌入选项（SQLite vs PostgreSQL）请参阅 [Wiki: NIP-CA-Server-Ops](https://github.com/labacacia/NPS-Release/wiki/NIP-CA-Server-Ops)。

---

## NPS Daemons

标准三层 NPS 拓扑的参考部署二进制，当前版本 `v1.0.0-alpha.5.2`。

| 仓库 | Daemon | 快速开始 |
|------|--------|----------|
| [labacacia/nps-daemons](https://github.com/labacacia/nps-daemons) | `npsd`（L1，:17433）· `nps-runner`（L1 FaaS）· `nps-gateway`（L2，:8080）· `nps-registry`（L2 NDP，:17436）| `docker compose up -d` |

Layer-3 信任锚 daemon（`nps-cloud-ca` 和 `nps-ledger`）在 `innolotus` 组织私有仓库，随 NPS Cloud GA 公开发布（2027 Q1+）。运维和架构详情请参阅 [Wiki: Operators-QuickStart](https://github.com/labacacia/NPS-Release/wiki/Operators-QuickStart)。

---

## 如何选择 SDK？

| 你在做… | 建议使用 |
|---------|---------|
| 编写调用 NPS 节点的 Agent | **Python** 或 **TypeScript** — 迭代最快 |
| 为已有服务构建 Memory Node | 跟服务语言匹配（企业级用 .NET / Java / Go；初创用 Python / TS） |
| 编写高吞吐编排器（NOP） | **Rust** 或 **Go** |
| 与 React 前端打包 | **TypeScript**（双 ESM + CJS） |
| JVM 环境 | **Java 21** |

所有 SDK 产出线缆上完全一致的帧，可自由混搭语言（如 Python Agent 调用 Rust Memory Node，由 Go NOP 服务编排）。

---

## 下一步

- [总览](overview.cn.md) — NPS 是什么，为何存在
- [协议族](protocols.cn.md) — 五层协议
- [路线图](roadmap.cn.md) — 已发布与待发布
- [快速开始](get-started.cn.md) — 按受众选择路径

---

📖 教程、参考资料和运维指南，请访问 [NPS Wiki](https://github.com/labacacia/NPS-Release/wiki)。
