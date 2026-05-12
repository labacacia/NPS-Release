<!--
NPS-Node Profile L1 — 自我声明模板（Self-Attestation Template）
==============================================================

将本文件复制到被测实现（IUT）仓库的根目录，填写每一项字段，并使用 IUT 的根
Ed25519 私钥对 attestation 块进行签名。通过的实现可在此基础上对外声明 L1
合规。

本模板对应的测试套件见 spec/services/conformance/NPS-Node-L1.md。
-->

[English Version](./NPS-NODE-L1-CERTIFIED.md) | 中文版

# NPS-Node Profile L1 — Certified

本实现声明已在以下条件下通过
[NPS-Node-L1 合规测试套件](https://github.com/LabAcacia/nps/blob/main/spec/services/conformance/NPS-Node-L1.md)。

---

## Implementation（实现信息）

| 字段 | 值 |
|------|----|
| **Name** | _(例如：example-daemon)_ |
| **Version** | _(semver，例如：0.1.0)_ |
| **Repository** | _(https URL)_ |
| **License** | _(SPDX id)_ |
| **Root NID** | `urn:nps:node:<authority>:<id>` |
| **Root public key (hex)** | _(64 个十六进制字符，Ed25519 公钥)_ |

## Peer Used for Certification（用于认证的对端实现）

| 字段 | 值 |
|------|----|
| **Name** | _(例如：nps-dotnet-reference)_ |
| **Version** | _(例如：1.0.0-alpha.3)_ |
| **Source** | _(URL 或包引用)_ |

## Run Environment（运行环境）

| 字段 | 值 |
|------|----|
| **Date** | _(ISO 8601 UTC，例如：2026-04-24T00:00:00Z)_ |
| **Platform** | _(例如：linux-x64、macos-arm64、win-x64)_ |
| **Hardware** | _(例如：1 vCPU / 1 GB RAM)_ |
| **NPS-Node Profile version** | 0.1 |
| **Conformance suite version** | 0.1 |

## Case Outcomes（用例结果）

_勾选每一项通过的测试用例；对 IUT 主动放弃的可选用例填写 `N/A`。_

### NCP — Wire format（线格式）
- [ ] `TC-N1-NCP-01` — Tier-1 JSON frame round-trip
- [ ] `TC-N1-NCP-02` — Hello + Anchor handshake
- [ ] `TC-N1-NCP-03` — Loopback listener default
- [ ] `TC-N1-NCP-04` — Tier-2 negotiation hygiene

### NIP — Identity（身份）
- [ ] `TC-N1-NIP-01` — Root keypair generation and permission
- [ ] `TC-N1-NIP-02` — IdentFrame sign and verify
- [ ] `TC-N1-NIP-03` — NID format
- [ ] `TC-N1-NIP-04` — Sub-NID issuance（可选；可填写 `N/A`）

### NDP — Discovery（发现）
- [ ] `TC-N1-NDP-01` — AnnounceFrame carries activation_mode
- [ ] `TC-N1-NDP-02` — AnnounceFrame signature
- [ ] `TC-N1-NDP-03` — ResolveFrame response
- [ ] `TC-N1-NDP-04` — GraphFrame subscription（可选；可填写 `N/A`）

### NWP — Inbox and delivery（信箱与投递）
- [ ] `TC-N1-NWP-01` — Inbox accepts ActionFrame
- [ ] `TC-N1-NWP-02` — Inbox persists across restart
- [ ] `TC-N1-NWP-03` — NWP pull serves inbox
- [ ] `TC-N1-NWP-04` — 100 QPS baseline
- [ ] `TC-N1-NWP-05` — Push path（可选；可填写 `N/A`）

### Observability（可观测性）
- [ ] `TC-N1-OBS-01` — Frame log entry per direction
- [ ] `TC-N1-OBS-02` — Log entry fields
- [ ] `TC-N1-OBS-03` — Log destination flexibility

## Results Manifest（结果清单）

_请将合规测试套件输出的 JSON 清单粘贴到此处：_

```json
{
  "profile": "NPS-Node-L1",
  "profile_version": "0.1",
  "iut": { "name": "", "version": "", "nid": "" },
  "peer": { "name": "", "version": "" },
  "run": { "date": "", "environment": "" },
  "cases": [],
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

_可选。说明环境特定的偏离、跳过的可选用例及其理由，或运行注意事项。_

---

*本声明为自签发。第三方认证（NPS Cloud CA）目标为 2027 Q1+ 的 L3 阶段，本版本暂不提供 L1 第三方认证。*

*版权：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
