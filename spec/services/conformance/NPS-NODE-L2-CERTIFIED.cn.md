<!--
NPS-AaaS Profile L2 — 自我声明模板（Self-Attestation Template）
================================================================

将本文件复制到被测实现（IUT）仓库的根目录，填写每一项字段，并使用 IUT 的根
Ed25519 私钥对 attestation 块进行签名。通过的实现可在以下声明范围内对外宣告
L2 合规。

本模板对应的测试套件见 spec/services/conformance/NPS-Node-L2.md。L1 前置条件
通过 NPS-NODE-L1-CERTIFIED.md 单独声明。
-->

[English Version](./NPS-NODE-L2-CERTIFIED.md) | 中文版

# NPS-AaaS Profile L2 — Certified

本实现声明已在以下条件、按所列范围通过
[NPS-Node-L2 合规测试套件](https://github.com/LabAcacia/nps/blob/main/spec/services/conformance/NPS-Node-L2.md)。

---

## Scope of This Attestation（本次声明范围）

本模板覆盖 v0.7 套件当前定义的所有 case family。任何 L2 声明都必须通过 topology
family；其余 family 仅在 IUT 声明相应角色或 transport capability 时适用。
L2-01..L2-05 是强制 AaaS case；L2-06/07 为 SHOULD，必须 `pass`，或以带非空
例外理由的 `na` 处置。

| Requirement（要求） | CR | 本次声明覆盖 |
|---------------------|----|------------|
| L2-01..L2-05 — NOP / OTel / CGN / preflight / retry | 当前 AaaS Profile | 是；强制 |
| L2-06..L2-07 — async / AlignStream | 当前 AaaS Profile | 是；可带理由采用 SHOULD 例外 |
| L2-08 — Anchor Node 上的 `topology.snapshot` / `topology.stream` | [NPS-CR-0002](https://github.com/LabAcacia/nps/blob/main/spec/cr/NPS-CR-0002-anchor-topology-queries.md) | 是 |
| NCP-over-TLS ingress | NPS-RFC-0006 | 条件适用 family |
| Bridge inbound | NPS-CR-0010 | 条件适用 family |
| Multi-Anchor / Registry HA | NPS-CR-0009 | 条件适用 family |

---

## Implementation（实现信息）

| 字段 | 值 |
|------|----|
| **Name** | _(例如：example-anchor)_ |
| **Version** | _(semver，例如：0.1.0)_ |
| **Repository** | _(https URL)_ |
| **License** | _(SPDX id)_ |
| **Root NID** | `urn:nps:node:<authority>:<id>` |
| **Root public key (hex)** | _(64 个十六进制字符，Ed25519 公钥)_ |
| **L1 prerequisite** | _(本实现 NPS-NODE-L1-CERTIFIED.md 的 URL，或本实现通过 L1 的日期)_ |

## Peer Used for Certification（用于认证的对端实现）

| 字段 | 值 |
|------|----|
| **Name** | _(例如：nps-dotnet-reference)_ |
| **Version** | _(例如：1.0.0-alpha.11)_ |
| **Source** | _(URL 或包引用)_ |

## Run Environment（运行环境）

| 字段 | 值 |
|------|----|
| **Date** | _(ISO 8601 UTC，例如：2026-04-27T00:00:00Z)_ |
| **Platform** | _(例如：linux-x64、macos-arm64、win-x64)_ |
| **Hardware** | _(例如：1 vCPU / 1 GB RAM)_ |
| **NPS-AaaS Profile version** | 0.7 |
| **Conformance suite version** | 0.7 |

## Case Outcomes（用例结果）

_emitted manifest 必须记录每个 case。适用且通过的 case 勾选；只有完整的条件适用
family 才能填写 `N/A`。Topology family 始终必选，partial-family `N/A` 非法，
single-/multi-Anchor HA disposition 必须互斥。L2-06/07 若记 `N/A`，emitted
manifest 的 `message` 字段必须写明理由。_

### AaaS 服务基线
- [ ] `TC-N2-AaaS-01` — 内部工作使用 NOP TaskFrame（MUST）
- [ ] `TC-N2-AaaS-02` — 向 TaskFrame.context 注入 OpenTelemetry trace（MUST）
- [ ] `TC-N2-AaaS-03` — CGN-Estimate budget 与 token_est 应答（MUST）
- [ ] `TC-N2-AaaS-04` — NOP preflight 阻断 worker dispatch（MUST）
- [ ] `TC-N2-AaaS-05` — NOP 重试与超时语义（MUST）
- [ ] `TC-N2-AaaS-06` — 异步 Action 生命周期（SHOULD；`N/A` 必须给出理由）
- [ ] `TC-N2-AaaS-07` — AlignStream CGN 背压（SHOULD；`N/A` 必须给出理由）

### Anchor Topology — happy paths（正常路径）
- [ ] `TC-N2-AnchorTopo-01` — Snapshot of a 3-member cluster
- [ ] `TC-N2-AnchorTopo-02` — Version monotonicity across joins
- [ ] `TC-N2-AnchorTopo-03` — Sub-Anchor member surfaces with `child_anchor` and `member_count`

### Anchor Topology — required negative paths (MUST-reject)（必须拒绝的负向路径）
- [ ] `TC-N2-AnchorTopo-04` — No `topology:read` capability → `NWP-TOPOLOGY-UNAUTHORIZED`
- [ ] `TC-N2-AnchorTopo-05` — `depth` exceeds Anchor max → `NWP-TOPOLOGY-DEPTH-UNSUPPORTED`
- [ ] `TC-N2-AnchorTopo-06` — Unknown `scope` → `NWP-TOPOLOGY-UNSUPPORTED-SCOPE`
- [ ] `TC-N2-AnchorTopo-07` — Unknown `filter` key → `NWP-TOPOLOGY-FILTER-UNSUPPORTED`
- [ ] `TC-N2-AnchorTopo-08` — Unknown reserved `type` → `NWP-RESERVED-TYPE-UNSUPPORTED`（MUST NOT 为 `NWP-ACTION-NOT-FOUND`）

### Anchor Stream（拓扑流）
- [ ] `TC-N2-AnchorStream-01` — `member_joined` on NDP Announce
- [ ] `TC-N2-AnchorStream-02` — `member_left` on NDP TTL expiry
- [ ] `TC-N2-AnchorStream-03` — Resume from `topology.since_version`
- [ ] `TC-N2-AnchorStream-04` — `resync_required` when version is too old

### NCP-over-TLS ingress（4 个全部 `pass`，或全部 `N/A`）
- [ ] `TC-N2-Tls-01` — TLS 1.3 上协商 ALPN `nps/1.0`
- [ ] `TC-N2-Tls-02` — 强制双向 TLS
- [ ] `TC-N2-Tls-03` — 客户端证书信任与 NID 绑定
- [ ] `TC-N2-Tls-04` — 拒绝 IdentFrame/证书 NID 不一致

### Bridge inbound（6 个全部 `pass`，或全部 `N/A`）
- [ ] `TC-N2-BridgeIn-01` — MCP inbound 必需方法集
- [ ] `TC-N2-BridgeIn-02` — gRPC inbound round-trip
- [ ] `TC-N2-BridgeIn-03` — A2A inbound round-trip
- [ ] `TC-N2-BridgeIn-04` — 裸 action 解析与歧义拒绝
- [ ] `TC-N2-BridgeIn-05` — 外部协议错误映射
- [ ] `TC-N2-BridgeIn-06` — 拒绝未声明协议或方向

### Multi-Anchor HA — Anchor 侧（6 个全部 `pass`，或全部 `N/A`）
- [ ] `TC-N2-HA-01` — topology read surfaces 携带 `cluster_epoch`
- [ ] `TC-N2-HA-02` — 计划内 `anchor_failover` wire shape
- [ ] `TC-N2-HA-03` — active-loss failover 为终结事件
- [ ] `TC-N2-HA-04` — quorum-loss wire shape 与只读模式
- [ ] `TC-N2-HA-05` — standby 拒绝 topology write
- [ ] `TC-N2-HA-06` — 被取代 leader 受 epoch fence

### Multi-Anchor HA — Registry 侧（2 个全部 `pass`，或全部 `N/A`）
- [ ] `TC-N2-HA-07` — Registry 解析最高 `cluster_epoch`
- [ ] `TC-N2-HA-08` — 拒绝同 epoch split-brain

### Single-Anchor 兼容
- [ ] `TC-N2-HA-09` — Single-Anchor epoch-one compatibility

## Results Manifest（结果清单）

_请将合规测试套件输出的 JSON 清单粘贴到此处：_

```json
{
  "profile": "NPS-Node-L2",
  "profile_version": "0.7",
  "scope": ["L2-01", "L2-02", "L2-03", "L2-04", "L2-05", "L2-08"],
  "iut": { "name": "", "version": "", "nid": "" },
  "peer": { "name": "", "version": "" },
  "run": { "date": "", "environment": "" },
  "cases": [
    { "id": "TC-N2-AaaS-01", "result": "pass" }
    /* ... 全部 38 个 case；L2-06/07 的 na 必须带 message；只有完整的条件适用 family 可标记为 na ... */
  ],
  "summary": { "pass": 0, "fail": 0, "skip": 0, "na": 0 }
}
```

## Attestation（声明签名）

_使用 IUT 的根私钥对结果清单 JSON（compact form，按 RFC 8785 canonicalize 后）的 SHA-256 hex 进行签名。验证方 MUST 使用上文声明的根公钥校验该签名。_

> **RFC 8785 JCS 要求**：此处的 "RFC 8785 canonicalized" 指 JSON Canonicalization
> Scheme（JCS）。实现 MUST 使用合规的 JCS 库，确保：
> (1) UTF-8 编码且无 BOM；(2) 键按 Unicode code point 排序；(3) 数字使用 JCS
> 规定的 IEEE 754 / ECMAScript 序列化（不带尾随零、无 `+` 号）；(4) Unicode 转义按
> RFC 8785 §3.2.2.2 处理。任一规则的偏离都会产生不同的 SHA-256，导致跨语言
> 校验失败。

| 字段 | 值 |
|------|----|
| **Manifest SHA-256** | _(64 个十六进制字符)_ |
| **Signature algorithm** | `Ed25519` |
| **Signature** | _(128 个十六进制字符，对清单 SHA-256 进行签名)_ |
| **Signed at** | _(ISO 8601 UTC)_ |

## Notes（备注）

_可选。说明环境特定的偏离或运行注意事项。_

---

*本声明为自签发。第三方认证（NPS Cloud CA）目标为 2027 Q1+ 的 L3 阶段，本版本暂不提供 L2 第三方认证。*

*版权：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
