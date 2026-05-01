[English Version](./NPS-3-NIP.md) | 中文版

# NPS-3: Neural Identity Protocol (NIP)

**Spec Number**: NPS-3  
**Status**: Proposed  
**Version**: 0.5  
**Date**: 2026-04-26  
**Port**: 17433（默认，共用）/ 17435（可选独立）  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  
**Depends-On**: NPS-1 (NCP v0.6)  

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
| `assurance_level` | string | 可选 | 取值 `"anonymous"` / `"attested"` / `"verified"` 之一（见 §5.1.1）。当 NID 证书携带 `id-nid-assurance-level` 扩展时本字段为 REQUIRED。接收方 MUST 把缺失字段视作 `"anonymous"`（向后兼容 v1.0-alpha.2 发布者）。两端都存在时 MUST 与证书扩展一致，否则 `NIP-ASSURANCE-MISMATCH`。（NPS-RFC-0003）|

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

### 5.1.1 保证等级（NPS-RFC-0003）

NPS 为 Agent 身份定义三个**保证等级**，参考 NIST SP 800-63 IAL 与 CA/B Forum DV/OV/EV 设计。等级通过 `IdentFrame.assurance_level` 传输，并作为 Node 决策（`NWM.min_assurance_level`）的权威来源；当使用 X.509 NID 证书（NPS-RFC-0002）时，证书 MUST 同样携带 `id-nid-assurance-level` 扩展，且两者 MUST 一致。

| 等级 | 枚举值 | CA 最低要求 | 典型用途 |
|------|--------|-------------|---------|
| L0 | `"anonymous"` | 自签，或 CA 签发但无身份绑定 | 业余 Agent、开发/测试、免费只读端点 |
| L1 | `"attested"` | NID 由 RFC-0002 兼容 CA 签发；CA 验证 NID 私钥持有（如 ACME `agent-01` 挑战）；联系邮箱或域名经过验证。**注**："RFC-0002 兼容"要求 RFC-0002 处于 Accepted 状态且使用已注册的 IANA OID。使用临时 OID `1.3.6.1.4.1.99999.1` 的原型实现不满足此合规或生产标准，须待 PEN 分配后方可（见 NPS-RFC-0002 §10 OQ-2）。| 大多数生产 Agent；默认限速级别 |
| L2 | `"verified"` | L1 要求 **加** CA 绑定运营者法律身份（org-NID 用工商注册，托管 Agent 用 AaaS 运营者签字声明）| 受监管集成、付费高级、可签合约的编排 |

默认为 `"anonymous"` —— RFC-0003 之前的 NID 以及任何缺该字段的 NID 都按 L0 处理。Node 通过 NWM 中 `min_assurance_level` 声明严格要求（NPS-2 §4.1、§4.3）。

枚举**有序**：`anonymous < attested < verified`。请求等级低于 Node 要求时 MUST 返回 `NWP-AUTH-ASSURANCE-TOO-LOW`（`NPS-AUTH-FORBIDDEN`）。

完整设计动机（含与 NPS-RFC-0002 协调的 X.509 critical extension 翻转计划）参见 [NPS-RFC-0003](rfcs/NPS-RFC-0003-agent-identity-assurance-levels.cn.md)。

---

### 5.1.2 声誉日志条目（NPS-RFC-0004）

> 与保证等级（§5.1.1）配套。`assurance_level` 回答*"这是谁？"*，声誉日志条目回答*"这个 NID 行为如何？"* —— 以 Certificate-Transparency 风格的 append-only 日志发布，让任何方（Node、审计、下游运营方）都能在没有预先关系的情况下拿到关于同一 NID 的同一份证据。

声誉条目是签名 JSON 对象，由 issuer（典型为 AaaS Anchor Node、审计、CA）发布，针对一个 `subject_nid` 记录一次观察到的行为事件。条目由**日志运营方**汇总成 append-only 历史；下面的 wire 格式是条目本身，与具体日志运营方的 HTTP 接口无关（运营接口见 [NPS-RFC-0004 §4.3](rfcs/NPS-RFC-0004-nid-reputation-log.cn.md)）。

**字段定义**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `v` | uint8 | 必填 | Schema 版本。本文档定义 `1`。|
| `log_id` | string (NID) | 必填 | append（或将要 append）此条目的日志运营方 NID。自发布时 MAY 与 `issuer_nid` 相等。|
| `seq` | uint64 | 必填 | 每个 `log_id` 内单调递增的序号。由日志运营方在 append 时设置。|
| `timestamp` | string (RFC 3339 UTC) | 必填 | 日志运营方的提交时间。|
| `subject_nid` | string (NID) | 必填 | 本条目*关于*哪个 NID。|
| `incident` | string (枚举) | 必填 | §5.1.2.1 中的取值之一，或前向兼容的未知值（接收方 MUST 保留未知值）。|
| `severity` | string (枚举) | 必填 | `info` / `minor` / `moderate` / `major` / `critical` 之一。|
| `window` | object | 可选 | `{ start: ISO 8601, end: ISO 8601 }`，描述观察窗口。|
| `observation` | object | 可选 | 自由格式的、机器可读的、按事件类型而定的细节（如限速违规可写 `{requests: 45000, threshold: 300}`）。|
| `evidence_ref` | string (URL) | 可选 | 更丰富证据（日志、转录）的 URL。|
| `evidence_sha256` | string (hex) | 可选 | 证据 blob 的 SHA-256；让查询方可以检测对日志外证据的篡改。|
| `issuer_nid` | string (NID) | 必填 | 做出断言的一方的 NID。MAY 等于 `log_id`。|
| `signature` | string | 必填 | `{alg}:{base64url}` Ed25519 签名，由 `issuer_nid` 私钥对**移除 `signature` 字段后**经 RFC 8785（JCS）规范化的条目计算。日志运营方校验该签名后，再用自己的 `log_id` 私钥重签整个条目以承诺序号（双签名模型）。|

#### 5.1.2.1 事件类型词汇表（初始集）

| 取值 | 含义 |
|------|------|
| `cert-revoked` | CA 吊销了 `subject_nid` 的证书。|
| `rate-limit-violation` | 持续违反公布的限速。|
| `tos-violation` | 违反 AaaS Anchor Node 公布的服务条款。|
| `scraping-pattern` | 行为匹配爬虫启发式规则。|
| `payment-default` | NPT 或法币的已承诺交易违约。|
| `contract-dispute` | 异步 NOP 任务上未解决的合同违约。|
| `impersonation-claim` | 第三方主张 `subject_nid` 在冒充自己。|
| `positive-attestation` | 显式正面信号（如独立审计通过）。|

接收方 MUST 把未知 `incident` 取值视作不透明透传（前向兼容）；运营侧过滤器 MAY 忽略它们。详见 [NPS-RFC-0004 §4.2](rfcs/NPS-RFC-0004-nid-reputation-log.cn.md)。

**错误**

| 错误码 | 触发条件 |
|--------|---------|
| `NIP-REPUTATION-ENTRY-INVALID`（`NPS-CLIENT-BAD-FRAME`）| 条目签名校验失败或规范化形式不合法。|
| `NIP-REPUTATION-LOG-UNREACHABLE`（`NPS-DOWNSTREAM-UNAVAILABLE`）| 准入评估时无法到达 Node `reputation_policy` 引用的某个日志运营方。|
| `NWP-AUTH-REPUTATION-BLOCKED`（`NPS-AUTH-FORBIDDEN`）| 声誉策略命中了对发起方 `subject_nid` 的 `reject_on` 规则。（定义在 NPS-2 §6 —— Anchor / Memory / Action / Complex / Bridge Node 选择启用。）|

> NPS-RFC-0004 Phase 1 规范化条目 wire 格式与上方签名规则；Merkle 树 / Signed Tree Head / inclusion-proof 接口（RFC 9162 风格）以及 NDP 发现接口在 Phase 2 落地。完整 phasing 见 RFC。

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
| `NIP-ASSURANCE-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | `IdentFrame.assurance_level` 与证书扩展 `id-nid-assurance-level` 不一致（防 downgrade 攻击）—— 见 §5.1.1 |
| `NIP-ASSURANCE-UNKNOWN` | `NPS-CLIENT-BAD-FRAME` | `assurance_level` 取值不在定义枚举（`anonymous` / `attested` / `verified`）之中 —— 见 §5.1.1 |
| `NIP-REPUTATION-ENTRY-INVALID` | `NPS-CLIENT-BAD-FRAME` | 声誉日志条目签名校验失败或规范化形式不合法 —— 见 §5.1.2（NPS-RFC-0004）|
| `NIP-REPUTATION-LOG-UNREACHABLE` | `NPS-DOWNSTREAM-UNAVAILABLE` | 准入评估时无法到达 Node `reputation_policy` 引用的某个日志运营方 —— 见 §5.1.2（NPS-RFC-0004）|

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
| 0.5 | 2026-04-26 | 新增 §5.1.2 声誉日志条目 —— Certificate-Transparency 风格的 append-only NID 行为记录 wire 格式。Phase 1 落地条目结构（12 字段，含基于 JCS 规范化的双签名要求）、初始 8 项 `incident` 枚举（`cert-revoked` / `rate-limit-violation` / `tos-violation` / `scraping-pattern` / `payment-default` / `contract-dispute` / `impersonation-claim` / `positive-attestation`）、5 级 `severity` 枚举。新增错误码 `NIP-REPUTATION-ENTRY-INVALID` 与 `NIP-REPUTATION-LOG-UNREACHABLE`。Merkle / STH / inclusion-proof 接口推迟到 Phase 2，详见 [NPS-RFC-0004](rfcs/NPS-RFC-0004-nid-reputation-log.cn.md) §8.1。|
| 0.4 | 2026-04-25 | 新增 §5.1.1 保证等级（`anonymous` / `attested` / `verified`）；IdentFrame 增加可选 `assurance_level` 字段，缺失向后兼容默认 `anonymous`；新增错误码 `NIP-ASSURANCE-MISMATCH`、`NIP-ASSURANCE-UNKNOWN`。详见 [NPS-RFC-0003](rfcs/NPS-RFC-0003-agent-identity-assurance-levels.cn.md)。`Depends-On` NCP 版本订正为 `v0.6`（NPS-RFC-0001）。 |
| 0.2 | 2026-04-12 | 统一端口 17433；IdentFrame 增加 metadata 字段（tokenizer 自动匹配）；错误码改用 NPS 状态码映射；完善错误码列表 |
| 0.1 | 2026-04-10 | 初始规范：NID 格式、IdentFrame/TrustFrame/RevokeFrame、CA Server API、验证流程 |

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
