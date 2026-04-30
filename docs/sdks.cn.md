# SDK

> [English](sdks.md) | 中文版

六种官方 SDK。每个 SDK 都实现了完整的五层协议 — **NCP + NWP + NIP + NDP + NOP** — 统一版本 **1.0.0-alpha.4**。

---

## 矩阵

| 语言 | 包名 | 最低版本 | 仓库 | API 参考 |
|------|------|----------|------|----------|
| .NET       | `LabAcacia.NPS.Core` (+ `.NWP` / `.NIP` / `.NDP` / `.NOP`)       | .NET 10     | [NPS-sdk-dotnet](https://github.com/labacacia/NPS-sdk-dotnet) | [doc/](https://github.com/labacacia/NPS-sdk-dotnet/tree/main/doc) |
| Python     | `nps-lib`                                    | 3.11        | [NPS-sdk-py](https://github.com/labacacia/NPS-sdk-py)         | [doc/](https://github.com/labacacia/NPS-sdk-py/tree/main/doc)     |
| TypeScript | `@labacacia/nps-sdk`                         | Node 22     | [NPS-sdk-ts](https://github.com/labacacia/NPS-sdk-ts)         | [doc/](https://github.com/labacacia/NPS-sdk-ts/tree/main/doc)     |
| Java       | `com.labacacia.nps:nps-java`                 | Java 21     | [NPS-sdk-java](https://github.com/labacacia/NPS-sdk-java)     | [doc/](https://github.com/labacacia/NPS-sdk-java/tree/main/doc)   |
| Rust       | `nps-sdk`                                    | Rust stable | [NPS-sdk-rust](https://github.com/labacacia/NPS-sdk-rust)     | [doc/](https://github.com/labacacia/NPS-sdk-rust/tree/main/doc)   |
| Go         | `github.com/labacacia/NPS-sdk-go`            | Go 1.25     | [NPS-sdk-go](https://github.com/labacacia/NPS-sdk-go)         | [doc/](https://github.com/labacacia/NPS-sdk-go/tree/main/doc)     |

## NIP CA Server

可独立部署的 Neural Identity Protocol（NPS-3 §8）证书颁发机构。当前版本 `v1.0.0-alpha.4`（自 `v1.0.0-alpha.3` 起从 SDK 中拆出独立发布）。

| 仓库 | 技术栈 | 快速开始 |
|------|--------|----------|
| [labacacia/nip-ca-server](https://github.com/labacacia/nip-ca-server) | C# / ASP.NET Core 10 + PostgreSQL，单 Docker | `git clone … && docker compose up -d` |

构建在已发布的 [`LabAcacia.NPS.NIP`](https://www.nuget.org/packages/LabAcacia.NPS.NIP/) NuGet 包之上，自包含（不依赖 monorepo）。仓库 `example/` 收录 5 个其他语言（Python / TypeScript / Java / Rust / Go）的冻结参考移植，仅供阅读。

## NPS Daemons

NPS 标准三层部署拓扑的参考部署二进制。4 个 OSS daemon（Layer 1 + Layer 2）打成一个 bundle 仓，当前版本 `v1.0.0-alpha.4`。

| 仓库 | Daemons | 快速开始 |
|------|---------|----------|
| [labacacia/nps-daemons](https://github.com/labacacia/nps-daemons) | `npsd`（L1 主机本地，端口 17433）· `nps-runner`（L1 FaaS）· `nps-gateway`（L2 ingress，:8080）· `nps-registry`（L2 NDP，:17436） | `git clone … && docker compose up -d` |

4 个 daemon 都依赖 nuget.org 上发布的 `LabAcacia.NPS.*` NuGet 包（Core / NIP / NDP / NWP / NWP.Anchor / NOP），都是多阶段 Docker 镜像。每个 daemon 有自己的子目录（`README` / `CHANGELOG` / `Dockerfile` / `Program.cs`），仓库根的 `docker-compose.yml` 一键拉起 4 个。

Layer-3 **信任锚** daemon（`nps-cloud-ca` 和 `nps-ledger`）在 GitHub `innolotus` 组织下作为私有仓存放，跟 **NPS Cloud** GA 一起公开（计划 2027 Q1+）。今天就要自托管 CA，用 [`labacacia/nip-ca-server`](https://github.com/labacacia/nip-ca-server)。

---

## 安装

### Python

```bash
pip install nps-lib==1.0.0a4
```

### TypeScript / JavaScript

```bash
npm install @labacacia/nps-sdk@1.0.0-alpha.4
# 或
pnpm add @labacacia/nps-sdk@1.0.0-alpha.4
```

### Rust

```toml
[dependencies]
nps-sdk = "1.0.0-alpha.4"
```

### Go

```bash
go get github.com/labacacia/NPS-sdk-go@v1.0.0-alpha.4
```

### Java（Gradle Kotlin DSL）

```kotlin
dependencies {
    implementation("com.labacacia.nps:nps-java:1.0.0-alpha.4")
}
```

### .NET

```bash
dotnet add package LabAcacia.NPS.Core --version 1.0.0-alpha.4
```

---

## 最小示例 — 编码 / 解码一个 `AnchorFrame`

### Python

```python
from nps_sdk.core import NpsFrameCodec, create_full_registry, compute_anchor_id
from nps_sdk.ncp import AnchorFrame

codec = NpsFrameCodec(create_full_registry())

schema = {"fields": [{"name": "id", "type": "uint64"}]}
frame  = AnchorFrame(anchor_id=compute_anchor_id(schema), schema=schema, ttl=3600)

wire = codec.encode(frame.frame_type(), frame.to_dict(), tier=1, is_final=True)
ft, dict_, _ = codec.decode(wire)
```

### TypeScript

```ts
import { NpsFrameCodec, createFullRegistry, computeAnchorId } from "@labacacia/nps-sdk/core";
import { AnchorFrame } from "@labacacia/nps-sdk/ncp";

const codec = new NpsFrameCodec(createFullRegistry());
const schema = { fields: [{ name: "id", type: "uint64" }] };
const frame = new AnchorFrame(computeAnchorId(schema), schema, 3600);

const wire = codec.encode(frame.frameType(), frame.toDict(), "msgpack", true);
const [ft, dict] = codec.decode(wire);
```

### Rust

```rust
use nps_core::{FrameCodec, Registry};
use nps_ncp::{AnchorFrame, FrameSchema, SchemaField};

let codec = FrameCodec::new(&Registry::default());
let schema = FrameSchema {
    fields: vec![SchemaField { name: "id".into(), r#type: "uint64".into(), ..Default::default() }],
};
let frame = AnchorFrame::new(&schema, 3600);
let wire  = codec.encode(&frame)?;
```

### Go

```go
import (
    "github.com/labacacia/NPS-sdk-go/core"
    "github.com/labacacia/NPS-sdk-go/ncp"
)

codec := core.NewNpsFrameCodec(core.CreateFullRegistry())
schema := core.FrameDict{"fields": []any{map[string]any{"name": "id", "type": "uint64"}}}
frame  := &ncp.AnchorFrame{
    AnchorID: core.ComputeAnchorID(schema),
    Schema:   schema,
    TTL:      3600,
}

wire, _ := codec.Encode(frame.FrameType(), frame.ToDict(), core.EncodingTierMsgPack, true)
```

### Java

```java
var registry = FrameRegistry.createFull();
var codec    = new NpsFrameCodec(registry);

var schema = Map.of("fields", List.of(Map.of("name", "id", "type", "uint64")));
var frame  = new AnchorFrame(AnchorIdUtil.compute(schema), schema, 3600L);

byte[] wire = codec.encode(frame.frameType(), frame.toDict(), EncodingTier.MSGPACK, true);
```

### .NET

```csharp
var codec = new NpsFrameCodec(FrameRegistry.CreateFull());

var schema = new Dictionary<string, object>
{
    ["fields"] = new[] { new Dictionary<string, object> { ["name"] = "id", ["type"] = "uint64" } }
};
var frame = new AnchorFrame(AnchorId.Compute(schema), schema, ttl: 3600);

var wire = codec.Encode(frame.FrameType, frame.ToDict(), EncodingTier.MsgPack, isFinal: true);
```

---

## 选哪个 SDK？

| 你是… | 建议的 SDK |
|-------|-----------|
| 写 Agent 调用 NPS 节点 | **Python** 或 **TypeScript** — 迭代最快 |
| 给现有服务加 Memory Node | 匹配服务的语言（企业侧 .NET / Java / Go；创业侧 Python / TS）|
| 写高吞吐编排器（NOP）| **Rust** 或 **Go** |
| 与 React 前端打包 | **TypeScript**（ESM + CJS 双输出）|
| JVM 为主的环境 | **Java 21** |

所有 SDK 编码出的帧在线缆上完全一致。你可以自由混搭 — 例如 Python Agent 调用 Rust Memory Node，由 Go NOP 服务编排。

---

## 下一步

- [总览](overview.cn.md) — NPS 是什么，为何存在
- [协议族](protocols.cn.md) — 五层协议
- [路线图](roadmap.cn.md) — 已发布 / 待发布
