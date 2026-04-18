# SDK

> [English](sdks.md) | 中文版

六种官方 SDK。每个 SDK 都实现了完整的五层协议 — **NCP + NWP + NIP + NDP + NOP** — 统一版本 **1.0.0-alpha.1**。

---

## 矩阵

| 语言 | 包名 | 最低版本 | 仓库 | API 参考 |
|------|------|----------|------|----------|
| .NET       | `NPS.SDK`                                    | .NET 10     | [NPS-sdk-dotnet](https://github.com/labacacia/NPS-sdk-dotnet) | [doc/](https://github.com/labacacia/NPS-sdk-dotnet/tree/main/doc) |
| Python     | `nps-sdk`                                    | 3.11        | [NPS-sdk-py](https://github.com/labacacia/NPS-sdk-py)         | [doc/](https://github.com/labacacia/NPS-sdk-py/tree/main/doc)     |
| TypeScript | `@labacacia/nps-sdk`                         | Node 18     | [NPS-sdk-ts](https://github.com/labacacia/NPS-sdk-ts)         | [doc/](https://github.com/labacacia/NPS-sdk-ts/tree/main/doc)     |
| Java       | `com.labacacia:nps-sdk`                      | Java 21     | [NPS-sdk-java](https://github.com/labacacia/NPS-sdk-java)     | [doc/](https://github.com/labacacia/NPS-sdk-java/tree/main/doc)   |
| Rust       | `nps-sdk`                                    | Rust stable | [NPS-sdk-rust](https://github.com/labacacia/NPS-sdk-rust)     | [doc/](https://github.com/labacacia/NPS-sdk-rust/tree/main/doc)   |
| Go         | `github.com/labacacia/nps/impl/go`           | Go 1.23     | [NPS-sdk-go](https://github.com/labacacia/NPS-sdk-go)         | [doc/](https://github.com/labacacia/NPS-sdk-go/tree/main/doc)     |

每个 SDK 仓库下都附带一个 `nip-ca-server/` — NIP Certificate Authority 的参考部署，Docker 打包。

---

## 安装

### Python

```bash
pip install nps-sdk
```

### TypeScript / JavaScript

```bash
npm install @labacacia/nps-sdk
# 或
pnpm add @labacacia/nps-sdk
```

### Rust

```toml
[dependencies]
nps-sdk = "1.0.0-alpha.1"
```

### Go

```bash
go get github.com/labacacia/nps/impl/go@v1.0.0-alpha.1
```

### Java（Gradle Kotlin DSL）

```kotlin
dependencies {
    implementation("com.labacacia:nps-sdk:1.0.0-alpha.1")
}
```

### .NET

```bash
dotnet add package NPS.SDK --version 1.0.0-alpha.1
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
    "github.com/labacacia/nps/impl/go/core"
    "github.com/labacacia/nps/impl/go/ncp"
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
