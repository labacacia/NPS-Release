# SDKs

> English | [中文版](sdks.cn.md)

Six official SDKs. Every SDK implements all five protocols — **NCP + NWP + NIP + NDP + NOP** — at version **1.0.0-alpha.2**.

---

## Matrix

| Language | Package | Min version | Repo | API Reference |
|----------|---------|-------------|------|---------------|
| .NET      | `LabAcacia.NPS.Core` (+ `.NWP` / `.NIP` / `.NDP` / `.NOP`)      | .NET 10     | [NPS-sdk-dotnet](https://github.com/labacacia/NPS-sdk-dotnet) | [doc/](https://github.com/labacacia/NPS-sdk-dotnet/tree/main/doc) |
| Python    | `nps-lib`                                    | 3.11        | [NPS-sdk-py](https://github.com/labacacia/NPS-sdk-py)         | [doc/](https://github.com/labacacia/NPS-sdk-py/tree/main/doc)     |
| TypeScript| `@labacacia/nps-sdk`                         | Node 22     | [NPS-sdk-ts](https://github.com/labacacia/NPS-sdk-ts)         | [doc/](https://github.com/labacacia/NPS-sdk-ts/tree/main/doc)     |
| Java      | `com.labacacia:nps-sdk`                      | Java 21     | [NPS-sdk-java](https://github.com/labacacia/NPS-sdk-java)     | [doc/](https://github.com/labacacia/NPS-sdk-java/tree/main/doc)   |
| Rust      | `nps-sdk`                                    | Rust stable | [NPS-sdk-rust](https://github.com/labacacia/NPS-sdk-rust)     | [doc/](https://github.com/labacacia/NPS-sdk-rust/tree/main/doc)   |
| Go        | `github.com/labacacia/NPS-sdk-go`            | Go 1.25     | [NPS-sdk-go](https://github.com/labacacia/NPS-sdk-go)         | [doc/](https://github.com/labacacia/NPS-sdk-go/tree/main/doc)     |

## NIP CA Server

Standalone deployable Certificate Authority for the Neural Identity Protocol (NPS-3 §8). Released independently from the SDKs at `v1.0.0-alpha.3`.

| Repo | Stack | Quickstart |
|------|-------|------------|
| [labacacia/nip-ca-server](https://github.com/labacacia/nip-ca-server) | C# / ASP.NET Core 10, PostgreSQL, single-Docker | `git clone … && docker compose up -d` |

Built on the published [`LabAcacia.NPS.NIP`](https://www.nuget.org/packages/LabAcacia.NPS.NIP/) NuGet package; self-contained (no monorepo dependency). The repo's `example/` folder carries five frozen reference ports (Python / TypeScript / Java / Rust / Go) for educational reading only.

## NPS Daemons

Reference deployment binaries for the standard three-layer NPS topology. The four open-source daemons (Layer 1 + Layer 2) ship in one bundle repo at `v1.0.0-alpha.3`.

| Repo | Daemons | Quickstart |
|------|---------|------------|
| [labacacia/nps-daemons](https://github.com/labacacia/nps-daemons) | `npsd` (L1 host-local, port 17433) · `nps-runner` (L1 FaaS) · `nps-gateway` (L2 ingress, :8080) · `nps-registry` (L2 NDP, :17436) | `git clone … && docker compose up -d` |

All four depend on published `LabAcacia.NPS.*` NuGet packages (Core / NIP / NDP / NWP / NWP.Anchor / NOP) and ship as multi-stage Docker images. Each daemon has its own subdirectory with `README` / `CHANGELOG` / `Dockerfile` / `Program.cs`, and the bundle's top-level `docker-compose.yml` spins them all up at once.

The Layer-3 **trust-anchor** daemons (`nps-cloud-ca` and `nps-ledger`) live in private repos under the `innolotus` GitHub organisation and ship publicly with **NPS Cloud** GA (planned 2027 Q1+). For self-host CA needs today, use [`labacacia/nip-ca-server`](https://github.com/labacacia/nip-ca-server).

---

## Install

### Python

```bash
pip install nps-lib==1.0.0a2
```

### TypeScript / JavaScript

```bash
npm install @labacacia/nps-sdk@1.0.0-alpha.2
# or
pnpm add @labacacia/nps-sdk@1.0.0-alpha.2
```

### Rust

```toml
[dependencies]
nps-sdk = "1.0.0-alpha.2"
```

### Go

```bash
go get github.com/labacacia/NPS-sdk-go@v1.0.0-alpha.2
```

### Java (Gradle Kotlin DSL)

```kotlin
dependencies {
    implementation("com.labacacia:nps-sdk:1.0.0-alpha.2")
}
```

### .NET

```bash
dotnet add package LabAcacia.NPS.Core --version 1.0.0-alpha.2
```

---

## Minimal example — encode / decode an `AnchorFrame`

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

## Which SDK should you pick?

| You are… | Suggested SDK |
|----------|---------------|
| Writing an agent that calls NPS nodes | **Python** or **TypeScript** — fastest iteration |
| Building a Memory Node for an existing service | match the language of the service (.NET / Java / Go for enterprise; Python / TS for startups) |
| Writing a high-throughput orchestrator (NOP) | **Rust** or **Go** |
| Bundling with a React frontend | **TypeScript** (dual ESM + CJS) |
| Shipping to a JVM-heavy environment | **Java 21** |

All SDKs encode wire-identical frames. You can mix languages freely — e.g. a Python agent calling a Rust Memory Node, orchestrated by a Go NOP server.

---

## Next

- [Overview](overview.md) — what NPS is, why it exists
- [Protocols](protocols.md) — the five layers
- [Roadmap](roadmap.md) — what's shipped and what's next
