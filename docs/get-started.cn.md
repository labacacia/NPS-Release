# 快速开始

> [English](get-started.md) | 中文版

NPS 是一套多层协议族，从哪里开始取决于你在构建什么。

---

## 我在编写 AI Agent 或模型客户端

你需要一个 **SDK**，选择你的语言，跟着 Wiki 上的快速入门走。

| 语言 | 安装 | Wiki 快速入门 |
|------|------|--------------|
| Python     | `pip install nps-lib`                                    | [SDK-Python](https://github.com/labacacia/NPS-Release/wiki/SDK-Python) |
| TypeScript | `npm install @labacacia/nps-sdk`                         | [SDK-TypeScript](https://github.com/labacacia/NPS-Release/wiki/SDK-TypeScript) |
| Rust       | `cargo add nps-sdk`                                      | [SDK-Rust](https://github.com/labacacia/NPS-Release/wiki/SDK-Rust) |
| Go         | `go get github.com/labacacia/NPS-sdk-go`                 | [SDK-Go](https://github.com/labacacia/NPS-Release/wiki/SDK-Go) |
| Java       | `implementation("com.labacacia.nps:nps-java:...")`        | [SDK-Java](https://github.com/labacacia/NPS-Release/wiki/SDK-Java) |
| .NET       | `dotnet add package LabAcacia.NPS.Core`                  | [SDK-dotnet](https://github.com/labacacia/NPS-Release/wiki/SDK-dotnet) |

---

## 我在部署 NPS 节点或基础设施

你需要 **NPS Daemons** bundle（`npsd` + `nps-runner` + `nps-gateway` + `nps-registry`）。

→ [Wiki: Operators-QuickStart](https://github.com/labacacia/NPS-Release/wiki/Operators-QuickStart)

---

## 我在开发协议集成或协议桥接器

阅读 [协议族](protocols.cn.md) 页面中链接的协议规范，然后选择对应的兼容适配器：

- `compat/mcp-ingress` — MCP → NPS 翻译
- `compat/a2a-ingress` — A2A → NPS 翻译
- `compat/grpc-ingress` — gRPC → NPS 翻译

→ [Wiki: Protocol-Designer-Guide](https://github.com/labacacia/NPS-Release/wiki/Protocol-Designer-Guide)

---

## 我想在写代码之前先理解 NPS

从 [总览](overview.cn.md)（5 分钟阅读）开始，然后阅读 [协议族](protocols.cn.md) 了解五层架构。

---

## 我想为 NPS 做贡献

见 [CONTRIBUTING](https://github.com/labacacia/NPS-Release/blob/main/CONTRIBUTING.md) 了解流程，以及 [Wiki: Contributors-Guide](https://github.com/labacacia/NPS-Release/wiki/Contributors-Guide) 了解惯例。

---

📖 教程、参考资料和运维指南，请访问 [NPS Wiki](https://github.com/labacacia/NPS-Release/wiki)。
