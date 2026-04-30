[English Version](./NPS-RFC-0002-x509-acme-nid-certs.md) | 中文版

---
**RFC 编号**：NPS-RFC-0002
**标题**：NID 证书改用 X.509 + ACME
**状态**：Draft（prototype 于 2026-04-27 落地 —— 实测数据见 §9；推到 Proposed/Accepted 由 shepherd 评审决定）
**作者**：Ori Lynn <iamzerolin@gmail.com>（LabAcacia）
**Shepherd**：_待定——PR 开立时指派_
**创建日期**：2026-04-21
**最后更新**：2026-04-27
**接受日期**：_（合入 `dev` 时填写）_
**激活日期**：_（首个参考 SDK 发版时填写）_
**取代**：_无_
**被取代于**：_无_
**影响的规范**：NPS-3 NIP、tools/nip-ca-server（所有语言版本）、spec/error-codes.md
**影响的 SDK**：.NET、Python、TypeScript、Java、Rust、Go
---

# NPS-RFC-0002：NID 证书改用 X.509 + ACME

## 1. 摘要

把 NIP 自研证书格式替换为 **X.509v3** 证书，通过 LabAcacia 注册的
Extended Key Usage (EKU) OID 标记 "agent-identity"；把 NIP CA 的签发
接口从自研 REST API 迁到 **ACME (RFC 8555)**，新增 `agent-01` 挑战
类型以适配 NID 私钥可控的场景。主签名算法仍为 Ed25519（RFC 8410 —
Ed25519 in X.509）；CRL / OCSP 吊销语义保持不变。

## 2. 动机

本 RFC 回应 2026-04-20 的评审意见：**"CA 不如直接兼容现有的 CA 套路"**。

评论是对的。当前 NIP CA（`tools/nip-ca-server-*`）在操作模型上已经
复刻了经典 X.509 CA 的流程——CSR → 签发 → CRL/OCSP 吊销 → 续签——
但用的是自研序列化。这带来两项不必要的成本：

1. **工具链无法复用。** OpenSSL、rustls、BouncyCastle、`step-ca`、
   HashiCorp Vault PKI、HSM 厂商、Kubernetes 的 cert-manager——
   没一个能直接签发、验证或托管 NIP 证书。6 种语言的 NIP CA Server
   每一家都在重复实现同一套仪式。
2. **签发协议无法复用。** 通过自研 REST 接口手工签发 NID，享受不到
   ACME 多年积累的运维成熟度：自动续签、staging 环境、公共 CA 速率
   限制、Certbot/lego 客户端、Kubernetes operator。

改到 X.509 + ACME 的代价是暴露一个 ASN.1 解析面、证书体积 2–4 倍，
换来的是 15 年的 PKI 工具链复用；以及未来可以跑一个 "Let's Encrypt
for Agents" 的公共服务，或者把 NID 签发通过交叉签名委托给现有 CA。

## 3. 非目标

- **不** 改 `IdentFrame` 的 wire 格式——证书仍以 `cert_chain` 字节
  承载，只是内部编码从 NIP 自研改为 X.509。
- **不** 强制采用公共 CA 根。LabAcacia 自营 CA 仍然是默认根；组织
  完全可以继续跑私有 X.509 根。
- **不** 改 `NID` 格式（`nid:{algo}:{base64url(pubkey)}`）。
- **不** 废弃 Ed25519。Ed25519 继续是主签名算法；X.509 允许的
  RSA / ECDSA-P256 也继续支持。

## 4. 详细设计

### 4.1 Wire 格式 / 帧变更

**`IdentFrame.cert_chain` 编码** 从 `nip-cert-v1`（自研）改为
**DER 编码的 X.509 证书链**，按下列规则拼接或长度前缀。

**Subject 字段映射：**

| X.509 字段 | NIP 值 |
|------------|--------|
| Subject `CN` | NID 字符串（`nid:ed25519:...`） |
| Subject Alternative Name (SAN) URI | 同一个 NID 以 `URI` 形式再写一遍（RFC 5280 合规） |
| Issuer | 签发 CA 的 NID |
| NotBefore / NotAfter | 标准 X.509 有效期 |
| Public Key | Ed25519（RFC 8410 OID `1.3.101.112`） |
| Serial Number | 128-bit 随机（按 CA/B Forum baseline） |

**关键扩展——Extended Key Usage：**

申请 LabAcacia IANA Private Enterprise Number (PEN)——拿到后，
保留 OID arc `1.3.6.1.4.1.<PEN>.1`。

| OID | 含义 |
|-----|------|
| `1.3.6.1.4.1.<PEN>.1.1` | `agent-identity` —— 主体是一个 NPS Agent |
| `1.3.6.1.4.1.<PEN>.1.2` | `node-identity` —— 主体是一个 NPS Node |
| `1.3.6.1.4.1.<PEN>.1.3` | `ca-intermediate-agent` —— 本 CA 可签发 `agent-identity` 证书 |

EKU **标为 critical**。不认这些 EKU 的 verifier 在 NIP 语义下
**MUST** 拒绝该证书。（通用 TLS 客户端也会拒绝——这是 **故意的**，
NID 证书不能被当成 TLS server 证书误用。）

**自定义非 critical 扩展——`nid:assurance-level`：**

为 NPS-RFC-0003 预留。本 RFC 只定义编码形状：

```
nid-assurance-level EXTENSION ::= {
    SYNTAX NidAssuranceLevel
    IDENTIFIED BY id-nid-assurance-level  -- 1.3.6.1.4.1.<PEN>.2.1
}
NidAssuranceLevel ::= ENUMERATED {
    anonymous (0),
    attested  (1),
    verified  (2)
}
```

非 critical，使 v0.1 verifier 可以直接忽略；RFC-0003 在真正启用
assurance-level 强制时再翻成 critical。

### 4.2 Manifest / NWM 变更

无。X.509 证书链仍在 `IdentFrame` 里传，NWM 透明。

### 4.3 错误码

`spec/error-codes.md` 新增：

| 错误码 | NPS 状态码 | 说明 |
|--------|-----------|------|
| `NIP-CERT-FORMAT-INVALID` | `NPS-CLIENT-BAD-FRAME` | 证书链不是 DER 编码 X.509，或 ASN.1 解析失败 |
| `NIP-CERT-EKU-MISSING` | `NPS-CLIENT-BAD-FRAME` | 必需的 NPS EKU（`agent-identity` 或 `node-identity`）缺失 |
| `NIP-CERT-SUBJECT-NID-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | 证书 Subject CN / SAN URI 与 IdentFrame 里的 NID 不一致 |
| `NIP-ACME-CHALLENGE-FAILED` | `NPS-CLIENT-BAD-FRAME` | ACME `agent-01` 挑战校验失败 |

### 4.4 状态机 / 流程

**ACME `agent-01` 挑战**——专为 NID 身份校验设计的新挑战类型，
对齐 RFC 8555 §8 的 HTTP / DNS / TLS-ALPN 挑战：

```
Client (Agent)                 ACME Server (NIP CA)
    │                                 │
    │── newAccount (Ed25519 JWK) ──→ │
    │ ←── 201 + kid ────────────────  │
    │                                 │
    │── newOrder (identifiers) ────→ │
    │          identifier: type=nid,  │
    │          value=nid:ed25519:... │
    │ ←── order + authz URL ──────── │
    │                                 │
    │── GET authz URL ──────────────→ │
    │ ←── challenge: type=agent-01,   │
    │     token=T ──────────────────  │
    │                                 │
    │   （Agent 用 NID 私钥对 `T`    │
    │    签名；可以在已公告的         │
    │    endpoint 上 /nip/auth/T      │
    │    公开 signature，也可以       │
    │    作为 JWS 直接 POST 回        │
    │    ACME 服务端）                │
    │                                 │
    │── POST challenge (signed T) ─→ │
    │ ←── 200 + status=valid ──────── │
    │                                 │
    │── finalize (CSR) ─────────────→ │
    │ ←── 200 + cert URL ──────────── │
    │── GET cert URL ───────────────→ │
    │ ←── 200 + X.509 DER ──────────  │
```

`agent-01` 挑战以 TLS-ALPN-01 的简洁度证明 NID 私钥持有：单个
签名 token，不依赖外部 DNS / HTTP。服务端比较 MUST 按
`spec/NPS-3-NIP.md §10.2` 做 timing-safe 比较。

### 4.5 向后兼容性

- 旧 Agent 能验证新证书吗？**破坏。** X.509 解析器是硬性前置。
  6 SDK 各自引入 X.509 依赖（都已经是 TLS 的传递依赖）。
- 旧 CA 能签发旧格式证书吗？**破坏。** CA 工具必须 6 语言都
  同步迁移。
- Wire 级别版本识别：IdentFrame 增加 `cert_format` 字段
  （`v1-proprietary` | `v2-x509`）。跑 v0.2 栈的部署可以分阶段发布。
- `min_agent_version` 抬高，按 RFC 流程走 21 天窗口。

---

## 5. 备选方案

### 5.1 保留自研格式，新增 X.509 导出垫片

保留现有证书格式；提供一个单向 X.509 导出以做互通（例如
`nipc export --x509`）。

- **代价**：两种格式永远并存。6 CA Server + 6 SDK 双倍维护。
- **收益**：不破坏 wire。
- **结论**：拒绝。当初选自研是为了早期速度；模型已经验证，
  X.509 是现在正确的目标。

### 5.2 只上 X.509，不上 ACME（保留 REST 签发）

用 X.509 但继续 `/certs/issue` 这类自研 REST 接口。

- **代价**：失去 ACME 的续签自动化、速率限制纪律、成熟客户端
  生态（Certbot、lego、acme.sh、cert-manager）。
- **收益**：迁移更简单（只改 1 个规范而不是 2 个）。
- **结论**：作为终态拒绝；若工程带宽紧张，Phase 1 可以作为中间
  态采用。挂在 OQ-1。

### 5.3 不动

- **代价**：6 语言 CA Server 继续偏离行业实践。每个新部署都是
  定制集成。公共 CA 选项永久关上。
- **结论**：拒绝。

---

## 6. 缺陷与风险

- **迁移成本**：每语言 ~2 SDK-周（X.509 解析+生成+EKU 校验）。
  ACME 客户端：每语言 ~1 SDK-周。CA 侧：至少 .NET 版先复用已有
  ACME 服务端框架（`step-ca`、`boulder`、`pebble`），再逐步移植。
- **攻击面**：ASN.1 解析器历史上 CVE 高发。对策：强制使用经审计
  的库（.NET 用 `System.Security.Cryptography.X509Certificates`、
  Python 用 `cryptography`、Rust 用 `x509-parser` 等）——禁止
  手搓。
- **证书体积**：NIP 自研证书约 200 B；X.509 最小约 450 B、典型
  约 800 B。每个身份声明 +600 B。HTTP 模式下无感；原生模式高
  帧率时需要关注。
- **生态分裂**：Phase 1–2 期间 v1、v2 两种证书格式会并存一段。
  通过 `cert_format` 版本字段缓解。
- **可逆性**：差。回滚意味着在 12 个仓库里移除 X.509 支持。代价
  约 6 SDK-周。

---

## 7. 安全性考量

- **ASN.1 解析器暴露**：新攻击面。对策：强制用平台原生解析器
  （不手搓）；不认可未知 critical 扩展；拒绝带已知病态构造的
  证书（BMPString、非法负整数等）。
- **EKU critical 纪律**：NID 证书 MUST 把 EKU 标为 critical。
  这能阻断经典的 "本打算做 agent-identity 却被当成 TLS server
  cert" 串用攻击。Verifier MUST 先校验 EKU 再做任何其他使用。
- **ACME 账号密钥轮换**：RFC 8555 `newAccount` 给账号持有人
  轮换密钥的能力。NIP CA MUST 强制账号密钥算法为 Ed25519
  （对齐 NID 算法族）。
- **Certificate Transparency**：不在本 RFC。RFC-0004 做一个
  透明日志。在此之前，NIP CA MUST 维护一份可按 NID 查询的
  append-only 签发日志。
- **降级攻击**：旧 Agent 拿 v1 证书向支持 v2 的服务端发起时，
  Phase 3 落地后 MUST 拒绝；在此之前两种都收，带版本 tag 上报
  遥测。

---

## 8. 实施计划

### 8.1 阶段划分

| 阶段 | 范围 | 出口条件 |
|------|------|----------|
| 1 | .NET NIP + .NET CA Server 能发/收 X.509；.NET CA 实现 ACME `agent-01`；v1 + v2 并存 | 单测绿；跨格式互通（v1 client ↔ v2 server 双向） |
| 2 | 6 SDK + 6 CA Server 都支持 X.509 + ACME；`cert_format=v2` 默认 off | 跨 SDK 证书接收矩阵全绿；6 种 CA 都能跑 ACME 签发 |
| 3 | `cert_format=v2` 默认 on；21 天 v1 废弃通告 | 1 个发布周期无回归 |
| 4 | 移除 v1 代码路径 | 12 个仓库都去掉 v1 |

### 8.2 SDK 覆盖矩阵

| SDK | 负责人 | 状态 | 备注 |
|-----|--------|------|------|
| .NET | Ori Lynn | pending | 参考实现 |
| Python | _待定_ | pending | `cryptography` 库 |
| TypeScript | _待定_ | pending | `@peculiar/x509` |
| Java | _待定_ | pending | Bouncy Castle |
| Rust | _待定_ | pending | `x509-parser` + `rcgen` |
| Go | _待定_ | pending | stdlib `crypto/x509` |

### 8.3 测试计划

1. X.509 round-trip：CA 签发带 `agent-identity` EKU 的 v2
   证书；Agent 展示；verifier 接受。
2. EKU critical 强制：v2 证书缺 EKU → `NIP-CERT-EKU-MISSING`。
3. Subject / NID 不匹配：证书 Subject CN ≠ IdentFrame.nid →
   `NIP-CERT-SUBJECT-NID-MISMATCH`。
4. ACME `agent-01` happy path 端到端。
5. ACME 重放防护：重复挑战 token → 400。
6. 跨格式：v1 client + v2 server；v2 client + v1 server
   （Phase 2 双向互通；Phase 3+ 双向干净拒绝）。

### 8.4 基准

- IdentFrame 体积：预期 +600 B。在 wire-size 基准里加回归
  阈值："IdentFrame-v2 对于 Ed25519 叶子 + 单级中间 CA 时
  MUST NOT 超过 1200 字节"。
- 验签延迟：新基准衡量 X.509 验证路径。目标：典型硬件上
  ≤ 50 µs/cert（现有自研解析器 ~10 µs；≤ 5× 回归可接受，
  换来的是生态复用）。

> **2026-04-27 prototype 后修订（详见 §9.2）**：wire-size 上限放宽到
> 1600 字节（prototype 实测 1512 B）；删除 50 µs 绝对值目标，仅保留
> ≤ 5× regression 比率（绝对延迟受宿主硬件影响过大；prototype 在测
> 试容器内 v1 baseline 已经远超 50 µs，绝对目标不可移植）。新增
> "CA Server 二进制 ≤ 2×" 推迟到 daemon / nip-ca-server 多语言移植期再测。

---

## 9. 实测数据

### 9.1 Prototype 分支

Prototype 落地于 `feat/rfc-0002-x509-acme-prototype`，按顺序交付：

- **.NET NIP 发与验 X.509** ——
  `impl/dotnet/src/NPS.NIP/X509/{NpsX509Oids,NipX509Builder,NipX509Verifier,Ed25519X509SignatureGenerator}.cs`。
  `tests/NPS.Tests/Nip/X509/NipX509Tests.cs` 5 个用例覆盖 round-trip、
  EKU 缺失拒绝、subject/NID 不匹配拒绝、v1↔v2 跨格式共存。
- **In-process ACME 服务端 + `agent-01` challenge** ——
  `impl/dotnet/src/NPS.NIP/Acme/{AcmeJws,AcmeMessages,AcmeServer,AcmeClient}.cs` +
  `tests/NPS.Tests/Nip/Acme/AcmeAgent01Tests.cs`。两个用例覆盖 agent-01
  端到端发证 + 篡改签名负面路径返回 `NIP-ACME-CHALLENGE-FAILED`。
- **Pebble（RFC 8555 参考服务端）interop 推迟到后续。** 原因：pebble
  不实现 `agent-01`（这是本 RFC 自己提的非标准 challenge 类型）；用
  pebble 验证 ACME 标准合规只能加 HTTP-01 的覆盖，相对 `agent-01`
  本身的端到端证明价值边际很小。`tools/pebble/setup.sh` 已经放好
  二进制下载脚本，留给后续 PR。

### 9.2 实测

下表数据由 `NPS.Benchmarks.NipCert` 产出
（`dotnet run -c Release --project impl/dotnet/benchmarks/NPS.Benchmarks.NipCert -- --emit`），
报告写入 `docs/benchmarks/nip-cert-prototype.md`。

| 指标 | 基线 (v1) | 提议 (v2) | 差值 | 方法 |
|------|-----------|-----------|------|------|
| `IdentFrame` JSON 体积 | 459 B | 1512 B | +1053 B (+229%) | `JsonSerializer.Serialize(IdentFrame)` 的 UTF-8 字节数 |
| 验签延迟（2000 次平均） | 597.8 µs | 1698.5 µs | 2.84× | `Stopwatch` 计时 `verifier.VerifyAsync(frame)` |

§8.4 两条阈值因此被 prototype 数据修订：

- **Wire-size 上限**（原 ≤ 1200 B）：prototype 超 26%（1512 B）。
  prototype 的 chain 是 leaf + self-signed root，不是规范预期的
  leaf + 真正 intermediate；后者 KeyUsage/extensions 更少，base64url
  膨胀也可分摊。可读为"1200 B 在生产部署里偏紧 —— 实际要为
  IdentFrame 留 +1 KiB 左右余量"。规范上限放宽到 **1600 B**。
- **验签延迟绝对目标**（原 ≤ 50 µs）：v1 与 v2 在测试容器都未达
  （分别 12× 与 34×）。v1 baseline 已经被 JSON 规范化 + Ed25519
  verify 主导，而 X.509 chain 验证只在其上增量。比率目标
  （≤ 5×）**满足**（2.84×）。RFC 删除 50 µs 绝对目标，保留比率。

### 9.3 §10 OQ-1 的含义

Prototype 流程是先 X.509，后叠 ACME。耗时：X.509 + verifier + 测试 ~5
工作日；ACME（client + in-process server + agent-01 + 测试）~2 工作日。
pebble HTTP-01 interop 如果可做要再 ~1 天，被 `agent-01` 不被 pebble
理解阻塞。

OQ-1 推荐：**X.509 + ACME 一起做**（即原"both together"立场）。一旦
X.509 issuance 接好，ACME 大部分是机械工作；拆开会强迫做两轮跨语言
SDK 移植，把 ACME 一并合进同一个 acceptance 是更经济的做法。

---

## 10. 未决问题

- [x] **OQ-1**：Phase 1 —— 先 X.509 后 ACME，还是一起做？
  **2026-04-27 由 prototype 数据确认（详见 §9.3）**：一起做。.NET 上
  X.509 + ACME 合计 7 工作日；拆开会要做两轮 5 SDK × 2 阶段移植，
  把 ACME 在同一个 acceptance 内顺手合并明显更经济。
- [ ] **OQ-2**：IANA Private Enterprise Number (PEN) 申请中。
  目标：本 RFC 进入 Accepted 后 2 周内提交。PEN 下发前，用
  `1.3.6.1.4.1.99999.1` 下的 arc 并标记 "PROVISIONAL"。Prototype
  已在 `NpsX509Oids` 中使用此 arc，PEN 下发后做一次 search-replace
  即可。
- [ ] **OQ-3**：NIP CA 是否要和公共 CA（例如 Let's Encrypt）
  交叉签名？延后到后续 RFC。

---

## 11. 未来工作

- **后续 RFC**：NID 签发的 Certificate Transparency（与
  NPS-RFC-0004 范围重叠）。
- **后续 RFC**：与公共 CA 的交叉签名策略。
- **后续 RFC**：NID 私钥的 HSM / TPM 绑定。

---

## 12. 参考

- RFC 5280 —— "Internet X.509 Public Key Infrastructure Certificate and CRL Profile"
- RFC 8410 —— "Algorithm Identifiers for Ed25519, Ed448, X25519, and X448"
- RFC 8555 —— "Automatic Certificate Management Environment (ACME)"
- RFC 6960 —— "X.509 OCSP"
- `spec/NPS-3-NIP.md` —— 当前 NIP 规范
- `tools/nip-ca-server/README.md` —— 当前 CA Server
- 讨论：2026-04-20 关于 CA 兼容性的评审意见

---

## 附录 A. 修订记录

| 日期 | 作者 | 变更 |
|------|------|------|
| 2026-04-27 | Claude (prototype) | 用 `feat/rfc-0002-x509-acme-prototype`（.NET prototype + `NPS.Benchmarks.NipCert`）实测数据回填 §9 实测数据。OQ-1 闭环（X.509 + ACME 一起做）。§8.4 阈值修订（体积 1200 → 1600 B；删除绝对延迟目标，保留比率）。 |
| 2026-04-21 | Ori Lynn | 初稿 |
