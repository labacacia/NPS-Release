[English Version](./NPS-3-NIP.md) | 中文版

# NPS-3: Neural Identity Protocol (NIP)

**Spec Number**: NPS-3  
**Status**: Draft  
**Version**: 0.2  
**Date**: 2026-04-12  
**Port**: 17433（默认，共用）/ 17435（可选独立）  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  
**Depends-On**: NPS-1 (NCP v0.3)  

---

## 1. Terminology

本文档中的关键字 "MUST"、"MUST NOT"、"SHOULD"、"MAY" 按照 RFC 2119 解释。

---

## 2. 协议概述

NIP 为每个 AI Agent、NWP 节点和人类操作员颁发可验证的神经身份（NID），携带能力声明和权限 scope，支持信任链传递和实时吊销。NIP 是 NPS 安全模型的核心。

### 2.1 参与方

| 角色 | 描述 |
|------|------|
| Root CA | 根证书颁发机构，离线保存 |
| Org CA | 组织中间 CA，每组织一个 |
| Agent | 持有 NID 证书的 AI Agent |
| Node | 持有 NID 证书的 NWP 节点 |
| Operator | 人类管理员，持有 Operator 证书 |
| Verifier | 验证 NID 的一方（通常是 Node）|

### 2.2 CA 层级结构

```
Root CA（离线保存，极少使用）
    │
    └── Org Intermediate CA（每组织一个，有效期 1 年）
            ├── Agent Certificate    （有效期 30 天，支持自动续期）
            ├── Node Certificate     （有效期 90 天）
            └── Operator Certificate （有效期 1 年，绑定 MFA）
```

---

## 3. NID 格式

```abnf
nid         = "urn:nps:" entity-type ":" issuer-domain ":" identifier
entity-type = "agent" / "node" / "org"
issuer-domain = <RFC 1034 domain>
identifier  = 1*(ALPHA / DIGIT / "-" / "_" / ".")
```

**示例**

```
urn:nps:agent:ca.innolotus.com:550e8400-e29b-41d4    ← AI Agent
urn:nps:node:api.myapp.com:products                   ← NWP 节点
urn:nps:org:mycompany.com                              ← 组织 CA
```

---

## 4. 签名算法

| 算法 | 用途 | 密钥长度 |
|------|------|---------|
| **Ed25519**（主算法）| Agent 高频验签 | 32 字节私钥 / 32 字节公钥 |
| **ECDSA P-256**（备用）| 兼容性场景 | 32 字节私钥 / 64 字节公钥 |

公钥编码格式：`{algorithm}:{base64url(DER)}`，例如：`ed25519:MCowBQYDK2VwAyEA...`

---

## 5. 帧类型

### 5.1 IdentFrame (0x20)

Agent 身份声明与证书携带。每次建立连接时作为握手帧发送。

**字段定义**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x20` |
| `nid` | string | 必填 | Agent NID |
| `pub_key` | string | 必填 | 公钥，格式：`{alg}:{base64url}` |
| `capabilities` | array | 必填 | Agent 持有的能力列表 |
| `scope` | object | 必填 | 访问范围声明 |
| `issued_by` | string | 必填 | 颁发者 NID（Org CA）|
| `issued_at` | string | 必填 | 颁发时间（ISO 8601 UTC）|
| `expires_at` | string | 必填 | 过期时间（ISO 8601 UTC）|
| `serial` | string | 必填 | 证书序列号（Org CA 全局唯一，十六进制）|
| `signature` | string | 必填 | CA 对本帧签名，格式：`{alg}:{base64url}` |
| `metadata` | object | 可选 | Agent 元数据，见下 |

**metadata 字段（可选）**

| 字段 | 类型 | 描述 |
|------|------|------|
| `model_family` | string | Agent 使用的模型族标识，如 `"openai/gpt-4o"`、`"anthropic/claude-4"` |
| `tokenizer` | string | Agent 使用的 tokenizer 标识，如 `"cl100k_base"` |
| `runtime` | string | Agent 运行时标识，如 `"langchain/0.2"`、`"autogen/0.4"` |

metadata 字段不参与签名计算，Agent 可在运行时动态设置。Node 使用 metadata 中的 tokenizer 信息实现 NPT 自动匹配（见 [token-budget.cn.md](token-budget.cn.md)）。

**capabilities 标准值**

| 能力 | 描述 |
|------|------|
| `nwp:query` | 可查询 Memory Node |
| `nwp:action` | 可调用 Action Node |
| `nwp:stream` | 可接收 StreamFrame 响应 |
| `ncp:stream` | 可发起 NCP 流式传输 |
| `nop:delegate` | 可委托子任务给其他 Agent |
| `nop:orchestrate` | 可作为 Orchestrator 发起 TaskFrame |

**scope 字段**

```json
{
  "nodes":            ["nwp://api.myapp.com/*"],
  "actions":          ["orders:read", "orders:create"],
  "max_token_budget": 50000
}
```

**签名计算**

签名对象为 IdentFrame 去掉 `signature` 字段后的规范化 JSON（字段字母序，无空白）。

**完整示例**

```json
{
  "frame": "0x20",
  "nid": "urn:nps:agent:ca.innolotus.com:550e8400-e29b-41d4",
  "pub_key": "ed25519:MCowBQYDK2VwAyEA...",
  "capabilities": ["nwp:query", "nwp:action", "ncp:stream"],
  "scope": {
    "nodes":            ["nwp://api.myapp.com/*"],
    "actions":          ["orders:read", "orders:create"],
    "max_token_budget": 50000
  },
  "issued_by":  "urn:nps:org:mycompany.com",
  "issued_at":  "2026-04-10T00:00:00Z",
  "expires_at": "2026-05-10T00:00:00Z",
  "serial":     "0x0A3F9C",
  "signature":  "ed25519:3045022100..."
}
```

---

### 5.2 TrustFrame (0x21)

跨 CA 信任链传递与能力授权（商业功能，NPS Cloud）。

```json
{
  "frame":       "0x21",
  "grantor_nid": "urn:nps:org:org-a.com",
  "grantee_ca":  "urn:nps:org:org-b.com",
  "trust_scope": ["nwp:query"],
  "nodes":       ["nwp://api.org-a.com/public/*"],
  "expires_at":  "2026-12-31T00:00:00Z",
  "signature":   "ed25519:..."
}
```

---

### 5.3 RevokeFrame (0x22)

吊销 NID 或特定能力。

```json
{
  "frame":      "0x22",
  "target_nid": "urn:nps:agent:ca.innolotus.com:550e8400-e29b-41d4",
  "serial":     "0x0A3F9C",
  "reason":     "key_compromise",
  "revoked_at": "2026-04-10T12:00:00Z",
  "signature":  "ed25519:..."
}
```

`reason` 取值：`key_compromise` / `ca_compromise` / `affiliation_changed` / `superseded` / `cessation_of_operation`

---

## 6. 证书生命周期

```
注册          自动续期          主动吊销        CA 轮换
  │               │                 │              │
  ↓               ↓                 ↓              ↓
CA 颁发      到期前 7 天         RevokeFrame    新旧 CA 并行
IdentFrame   Agent 自动触发      立即生效         30 天
有效期 30 天  CA 颁发新证书      节点拒绝所有    平滑过渡
             新旧并行 1 小时      该 NID 请求
```

---

## 7. 验证流程

```
Node 收到 IdentFrame
  │
  ├─ 1. 检查 expires_at > now（过期→ NIP-CERT-EXPIRED）
  ├─ 2. 检查 issued_by 在 NWM trusted_issuers 中（不信任→ NIP-CERT-UNTRUSTED-ISSUER）
  ├─ 3. 用 issued_by CA 公钥验证 signature（失败→ NIP-CERT-SIGNATURE-INVALID）
  ├─ 4. OCSP 查询（若 NWM 配置了 ocsp_url）或检查本地 CRL（已吊销→ NIP-CERT-REVOKED）
  ├─ 5. 检查 capabilities 包含节点要求的能力（不足→ NIP-CERT-CAPABILITY-MISSING）
  └─ 6. 检查 scope.nodes 覆盖目标节点路径（不覆盖→ NWP-AUTH-NID-SCOPE-VIOLATION）
        全部通过 → 请求授权
```

---

## 8. NIP CA Server OSS API

| 方法 | 路径 | 认证 | 描述 |
|------|------|------|------|
| POST | `/v1/agents/register` | Operator Cert | 注册 Agent，返回 NID + IdentFrame |
| POST | `/v1/agents/{nid}/renew` | Agent Cert | 续期证书（到期前 7 天可续）|
| POST | `/v1/agents/{nid}/revoke` | Operator Cert | 吊销 NID |
| GET | `/v1/agents/{nid}/verify` | 无 | 验证 NID 有效性（OCSP 查询）|
| POST | `/v1/nodes/register` | Operator Cert | 注册 NWP 节点，颁发 Node Certificate |
| GET | `/v1/ca/cert` | 无 | CA 公钥证书 |
| GET | `/v1/crl` | 无 | 证书吊销列表 |
| GET | `/.well-known/nps-ca` | 无 | CA 发现端点 |

**`/.well-known/nps-ca` 响应**

```json
{
  "nps_ca": "0.1",
  "issuer": "urn:nps:org:ca.mycompany.com",
  "display_name": "MyCompany NPS CA",
  "public_key": "ed25519:MCowBQYDK2VwAyEA...",
  "algorithms": ["ed25519", "ecdsa-p256"],
  "endpoints": {
    "register": "https://ca.mycompany.com/v1/agents/register",
    "verify":   "https://ca.mycompany.com/v1/agents/{nid}/verify",
    "ocsp":     "https://ca.mycompany.com/ocsp",
    "crl":      "https://ca.mycompany.com/v1/crl"
  },
  "capabilities": ["agent", "node", "operator"],
  "max_cert_validity_days": 30
}
```

---

## 9. 错误码

| 错误码 | NPS 状态码 | 描述 |
|--------|-----------|------|
| `NIP-CERT-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` | 证书已过期 |
| `NIP-CERT-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` | 证书已吊销 |
| `NIP-CERT-SIGNATURE-INVALID` | `NPS-AUTH-UNAUTHENTICATED` | 证书签名验证失败 |
| `NIP-CERT-UNTRUSTED-ISSUER` | `NPS-AUTH-UNAUTHENTICATED` | 颁发者不在信任列表 |
| `NIP-CERT-CAPABILITY-MISSING` | `NPS-AUTH-FORBIDDEN` | 证书缺少所需能力 |
| `NIP-CERT-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` | 证书 scope 不覆盖目标路径 |
| `NIP-CA-NID-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | NID 不存在 |
| `NIP-CA-NID-ALREADY-EXISTS` | `NPS-CLIENT-CONFLICT` | NID 已存在（重复注册）|
| `NIP-CA-SERIAL-DUPLICATE` | `NPS-CLIENT-CONFLICT` | 证书序列号已存在 |
| `NIP-CA-RENEWAL-TOO-EARLY` | `NPS-CLIENT-BAD-PARAM` | 尚未到续期窗口 |
| `NIP-CA-SCOPE-EXPANSION-DENIED` | `NPS-AUTH-FORBIDDEN` | 请求 scope 超出父级 scope |
| `NIP-OCSP-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | OCSP 服务暂不可用 |
| `NIP-TRUST-FRAME-INVALID` | `NPS-CLIENT-BAD-FRAME` | TrustFrame 签名或格式不合法 |

HTTP 模式下的状态码映射见 [status-codes.cn.md](status-codes.cn.md)。

---

## 10. 安全考量

### 10.1 密钥存储
CA 私钥 MUST 存储于 HSM 或加密密钥文件（AES-256-GCM）。OSS 实现使用加密文件并预留 HSM 接口。

### 10.2 时序攻击防御
OCSP 响应时间 SHOULD 归一化（固定延迟至 200ms），防止通过响应时间推断证书状态。

### 10.3 Scope 不可扩大原则
委托链中任何环节的 scope MUST NOT 超过其父级 scope。CA MUST 在颁发证书时强制校验。

---

## 11. 变更历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.2 | 2026-04-12 | 统一端口 17433；IdentFrame 增加 metadata 字段（tokenizer 自动匹配）；错误码改用 NPS 状态码映射；完善错误码列表 |
| 0.1 | 2026-04-10 | 初始规范：NID 格式、IdentFrame/TrustFrame/RevokeFrame、CA Server API、验证流程 |

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
