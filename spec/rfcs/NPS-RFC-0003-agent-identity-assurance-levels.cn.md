[English Version](./NPS-RFC-0003-agent-identity-assurance-levels.md) | 中文版

---
**RFC 编号**：NPS-RFC-0003
**标题**：Agent 身份三级保证等级（反爬 / 信任分流）
**状态**：Accepted（Phase 1 —— spec + .NET 参考类型已落地）
**作者**：Ori Lynn <iamzerolin@gmail.com>（LabAcacia）
**Shepherd**：Ori Lynn（1.0 之前快速通道，见 `spec/cr/README.cn.md`）
**创建日期**：2026-04-21
**最后更新**：2026-04-25
**接受日期**：2026-04-25（1.0 之前快速通道；见 `spec/cr/README.cn.md`）
**激活日期**：_（首个参考 SDK 发版时填写，目标 v1.0-alpha.3）_
**取代**：_无_
**被取代于**：_无_
**影响的规范**：NPS-3 NIP、NPS-2 NWP、spec/services/NPS-AaaS-Profile.md、spec/error-codes.md、spec/status-codes.md
**影响的 SDK**：.NET、Python、TypeScript、Java、Rust、Go
---

# NPS-RFC-0003：Agent 身份三级保证等级（反爬 / 信任分流）

## 1. 摘要

为 Agent 身份引入三个 **保证等级**——`anonymous`（L0）、`attested`
（L1）、`verified`（L2），Node 可以在 NWM 里声明 `min_assurance_level`。
等级随 `IdentFrame` 传输，并作为 X.509 critical 扩展承载（见
NPS-RFC-0002 §4.1）。本 RFC 是 NPS 对 **"我怎么知道来的请求是
Agent 不是爬虫"** 的回答——协议不直接拦爬虫，但给 Node 一个原语：
要求可验证的 Agent 来源，然后可以按等级 计费 / 限速 / 拒绝。

## 2. 动机

本 RFC 回应 2026-04-20 的评审意见：

> 最核心的，如果我要接这套协议，最关心的反而是怎么做反爬？我怎么
> 知道来个请求不是爬虫而是 Agent 访问？

评论是对的：NPS 今天回答了 **"这个 Agent 是谁"**（NIP），但没回答
**"这个 Agent 的身份有多可信"**。NID 只是一个公钥——任何人，包括
爬虫农场，都可以生成一个 NID 并用它签请求。没有信任等级这个原语，
Node 要么收所有匿名流量（对爬虫友好），要么手工维护白名单（运维
负担）。

X.509 世界用 EV / OV / DV + CA/B Forum baseline 解决了同类问题；
身份验真世界用 NIST SP 800-63 的 IAL1/2/3 解决了同类问题。我们
**借用等级形状**，不照搬人类身份的具体流程。

这带来的商业模型空间：AaaS 运营商可以把 `L2` 认证流量定价高于
`L0`；审计方可以要求受监管集成强制 `L2`；被爬虫困扰的站点可以在
热点 endpoint 上设 `min_assurance_level: attested`，同时不误伤
把 NID 升级过的合法 Agent。

## 3. 非目标

- **不** 强制任何身份验真方法。CA 在各自信任域内决定 "attested"
  或 "verified" 具体怎么做，只需满足 §4 的最低标准。
- **不** 替代速率限制、WAF、CAPTCHA 等反滥用层。保证等级是
  **补充**，不是替代。
- **不** 拦截爬虫。Node 仍可以接受 `L0` 流量——等级是**信号**，
  不是单独的强制决策。
- **不** 定义声誉。声誉（这个 NID 以前有没有干过坏事？）是
  NPS-RFC-0004 的范围。保证等级关心身份声明的**来源**，不关心
  身份的行为记录。
- **不** 处理人类。本 RFC 是 Agent-to-Agent / Agent-to-Node；
  刻意绕开 FIDO2 / WebAuthn，因为那些需要人坐在键盘前。

## 4. 详细设计

### 4.1 保证等级

| 等级 | 枚举 | 最低标准 | 典型用途 |
|------|------|----------|----------|
| `anonymous` | 0 | 自签名 NID；或 CA 签名但不绑身份 | 业余 Agent、开发/测试、"给我一个免费报价" 只读 endpoint |
| `attested` | 1 | 被合规（RFC-0002）CA 签发的 NID；CA 已通过 ACME `agent-01` 证明私钥持有；CA 验证过联系邮箱 / 域名 | 大多数生产 Agent；限速分层默认 |
| `verified` | 2 | L1 标准 **+** CA 核实了运营者法律身份：组织 NID 要公司注册，托管 Agent 要已知 AaaS 运营商的签字声明 | 受监管集成（金融、医疗）、付费高级档、高信任编排 |

**"attested" 对 Node 真正的意义**：CA 证明了*有人*既控制 NID 私钥
*又*可以通过带外通道（ACME 账号邮箱、域名等）联系到——Agent 跑飞
时 CA 至少能邮件提醒运营者，最多直接吊销证书。L0 两个都给不了。

**"verified" 再往上加的**：CA 把 NID 和法律实体绑定。Node 可以
开发票、可以起诉、可以做合同级别的主张。

### 4.2 Wire 格式 / 帧变更

**`IdentFrame` 新增一个字段：**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `assurance_level` | `enum { anonymous, attested, verified }` | 是 | 当 NID 证书携带 `id-nid-assurance-level` 扩展（NPS-RFC-0002 §4.1）时，本字段 MUST 与之保持一致。**Phase gate**：Phase 1–2（当前）强制可选（SHOULD 检查并记录不匹配，MAY 强制执行）；Phase 3 flag day（见 §8.1）起强制变为 MUST，违规返回 `NIP-ASSURANCE-MISMATCH`。 |

证书是事实源；`IdentFrame.assurance_level` 是给不想解析证书就路由请求的服务端的冗余便利。

**Phase 1–2（当前）**：Verifier SHOULD 校验证书扩展并记录不匹配，但 MAY 跳过强制执行。选择执行的实现 MUST 以 `NIP-ASSURANCE-MISMATCH` 关闭连接。

**Phase 3（flag day，见 §8.1）**：Verifier MUST 校验证书扩展，若两者不一致 MUST 以 `NIP-ASSURANCE-MISMATCH` 关闭连接。这是完成升级的硬性截止点。

**X.509 扩展升级（Phase 3 — 尚未生效）**：NPS-RFC-0002 定义 `id-nid-assurance-level`（`1.3.6.1.4.1.<PEN>.2.1`）为**非 critical**。从 Phase 3 flag day 起，本 RFC 将其升级为 **critical**——旧 verifier MUST 拒绝带 critical 扩展的证书直到升级。这是**故意的**：一个启用了 `min_assurance_level` 的 Node MUST NOT 静默接收一个解析不了该扩展的 verifier。该升级在 Phase 1–2 **尚未生效**。

### 4.3 Manifest / NWM 变更

NWM 新增一个顶层可选字段：

```yaml
# /.nwm 片段
min_assurance_level: attested   # 默认：anonymous
auth:
  required_scopes: [...]
  min_assurance_level: verified  # per-action 覆盖（可选）
```

- 顶层 `min_assurance_level` 作用于所有读路径（`/.schema`、
  `/.actions`、anchor 拉取、同步 `/invoke`）。
- `auth:` 下的 per-action `min_assurance_level` 覆盖顶层。
- 默认 `anonymous`——本 RFC 不改变默认信任姿态；Node 主动 opt-in。
- 收到低于要求等级的请求时 MUST 以 `NWP-AUTH-ASSURANCE-TOO-LOW`
  拒绝，状态码 `NPS-AUTH-FORBIDDEN`，SHOULD 在 hint 里给一个
  ACME 注册 URL。

### 4.4 错误码

`spec/error-codes.md` 新增：

| 错误码 | NPS 状态 | 说明 |
|--------|---------|------|
| `NIP-ASSURANCE-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | `IdentFrame.assurance_level` 与证书扩展不一致 |
| `NWP-AUTH-ASSURANCE-TOO-LOW` | `NPS-AUTH-FORBIDDEN` | 请求的保证等级 < Node 的 `min_assurance_level` |
| `NIP-ASSURANCE-UNKNOWN` | `NPS-CLIENT-BAD-FRAME` | 证书扩展里的值超出已定义枚举 |

### 4.5 状态机 / 流程

拒绝路径（Node 启用 `min_assurance_level: attested`）：

```
Agent (L0)                         Node
   │                                 │
   │── HelloFrame ─────────────────→ │
   │── IdentFrame (assurance=0) ───→ │
   │                                 │  check: 0 < 1 (attested)
   │ ←── ErrorFrame ──────────────── │  code: NWP-AUTH-ASSURANCE-TOO-LOW
   │     hint: "https://ca.../acme"  │  status: NPS-AUTH-FORBIDDEN
   │ ←── close                       │
```

升级路径是带外的：Agent 去合规（RFC-0002）CA 重新注册，拿到带
`id-nid-assurance-level=attested` 扩展的新证书，重试连接。

### 4.6 向后兼容性

- 旧 Agent（没有 assurance-level 字段）？隐式 L0。默认
  `min_assurance_level: anonymous` 的 Node 继续工作。
- 旧 Node（NWM 里没有 `min_assurance_level`）？接收一切
  （默认 L0）。新 Agent 带 L2 证书也能连。
- X.509 扩展从非 critical（RFC-0002）翻成 critical（本 RFC）
  **是**兼容性风险点。由 `min_agent_version` 抬高 + 21 天窗口
  门禁。

---

## 5. 备选方案

### 5.1 二分 "认证 / 不认证"（两级）

砍掉 `attested`；只保留 `unverified` / `verified`。

- **代价**：中间地带坍塌。大多数生产 Agent 会落在 `unverified`
  里，因为法律实体绑定对业余/开发太重。Node 失去有用的把手。
- **结论**：拒绝。L1 是反爬价值真正所在的层级；L2 根据定义
  本就稀少。

### 5.2 仅用声誉（跳过等级）

不要等级，直接让 Node 依赖 NPS-RFC-0004 声誉日志。

- **代价**：声誉只在事故**之后**才有用。保证等级让 Node 从
  第一天就能说 "我只跟 CA 见过的 Agent 做生意"。两者解决不同
  问题。
- **结论**：作为替代拒绝；作为互补采纳（RFC-0003 + RFC-0004
  一起）。

### 5.3 自由形式 CA 策略 OID

让 CA 按 X.509 certificatePolicies 自定义 OID，Node 在 NWM 里
枚举接受哪些 OID。

- **代价**：每个 Node 都要知道每个 CA 的 OID 目录。运维噩梦；
  正是 CA/B Forum baseline 存在的原因。
- **结论**：拒绝。`certificatePolicies` 仍可在三级之上做**更细
  粒度**的 CA 特定策略，但三级是通用词汇。

### 5.4 不动

- **代价**：2026-04-20 评论未被回应。对爬虫敏感的早期采纳方
  （新闻站、价格 API、SaaS 目录）没法安心切 NWP。商业案例有洞。
- **结论**：拒绝。

---

## 6. 缺陷与风险

- **CA 策略负担**：CA 现在需要书面策略说明如何区分 L1 / L2。
  LabAcacia MUST 在 .NET NIP CA Server 旁边附一份参考策略文档。
- **信任集中风险**：`verified` 级别把权力集中在能做法律身份
  声明的 CA。对策：(a) 支持多家独立 CA（无单一根）；(b) Node
  NWM 可声明 `trusted_issuers` 列表（RFC-0002 §3 非目标确认
  此能力保留）。
- **套利风险**：爬虫从一家宽松 CA 买 L2 证书。对策：(a)
  NPS-RFC-0004 声誉日志——一个跑飞的 L2 NID 对签发 CA 的
  打击比 L0 匿名更大；(b) Node 维护 `trusted_issuers`。
- **迁移成本**：每语言 ~1 SDK-周（枚举 + NWM 字段 + 错误
  处理）。比 RFC-0002 轻。
- **可逆性**：中等。枚举本身可废弃；证书扩展翻成 critical
  是硬核部分。

---

## 7. 安全性考量

- **降级攻击**：攻击者 `IdentFrame.assurance_level=2` 但证书
  扩展为 0。对策：`NIP-ASSURANCE-MISMATCH` 关闭连接；证书为
  事实源。
- **发行方注水**：L2 只和最弱的能发 L2 的 CA 一样强。对策：
  `trusted_issuers`（per-Node）+ RFC-0004 按发行方追溯声誉。
- **无新密码学假设**：复用 RFC-0002 的 X.509 + Ed25519 栈。
- **DoS 向量**：除了基础证书解析成本外无新增。
- **隐私**：L2 证书绑法律实体。这对受监管用例是功能，对匿名
  Agent 是 bug。默认（anonymous）保护匿名 Agent；要 L2 的
  Node 是主动选择。

---

## 8. 实施计划

### 8.1 阶段划分

| 阶段 | 范围 | 出口条件 |
|------|------|----------|
| 1 | .NET NIP 解析证书扩展；IdentFrame 带字段；NWM 解析 `min_assurance_level`；强制可选 | 单测绿；默认行为不变 |
| 2 | 6 SDK + 6 CA Server 全部支持；CA Server 通过 ACME 签 L1 证书；每家 CA 的 L2 签发流程出文档 | 跨 SDK interop：L0/L1/L2 矩阵绿 |
| 3 | NWM 里设了 `min_assurance_level` 时默认启用强制；`id-nid-assurance-level` 扩展升级为 critical。**Flag day**：提前 ≥ 21 日历天在 NPS-Dev GitHub Discussions 公告；激活后，`NIP-ASSURANCE-MISMATCH` 强制为 MUST，不合规实现 MUST NOT 声称任何 NPS 合规级别 | 无回归；全 6 SDK 通过不匹配强制测试 |
| 4 | 启用了更严默认的 Node 移除 L0 默认 fast path | N/A——运营方决定 |

### 8.2 SDK 覆盖矩阵

| SDK | 负责人 | 状态 | 备注 |
|-----|--------|------|------|
| .NET | Ori Lynn | pending | 参考实现 |
| Python | _待定_ | pending | — |
| TypeScript | _待定_ | pending | — |
| Java | _待定_ | pending | — |
| Rust | _待定_ | pending | — |
| Go | _待定_ | pending | — |

### 8.3 测试计划

1. L0 Agent 对 `min_assurance_level: attested` 的 Node →
   `NWP-AUTH-ASSURANCE-TOO-LOW` 拒绝。
2. L2 Agent 对 `min_assurance_level: anonymous` 的 Node →
   接受。
3. 等级不一致（IdentFrame 说 L2，证书扩展说 L0）→
   `NIP-ASSURANCE-MISMATCH`。
4. Per-action 覆盖：全局 L1、`orders.create` 要 L2 → L1
   Agent 的 `/invoke:orders.create` 被拒，其他 action 通过。
5. 证书无扩展对启用强制的 Node（Phase 3 后）→ 视为 L0，
   等级 > 0 时拒绝。
6. 证书里枚举值未知 → `NIP-ASSURANCE-UNKNOWN`。

### 8.4 基准

- NWM 体积：+1 个可选字段，微不足道。
- IdentFrame 体积：+1 字节枚举 + X.509 扩展（已记入 RFC-0002
  的预算）。
- 每请求校验成本：证书解析后单次整数比较。预期开销 < 1 µs。

---

## 9. 实测数据

暂无。在 `Accepted` 前提交：
- 一个场景测试：L0 和 L1 Agent 对 `min_assurance_level: attested`
  的 Node 跑 10 次循环，确认 L0 循环早期 403 且不触达业务逻辑。
- 一家外部 CA（多半是 `pebble` 测试桩）通过 ACME `agent-01`
  签 L1 证书。

| 指标 | 基线 | 提议 | 差值 | 方法 |
|------|------|------|------|------|
| NWM 体积 | _无此字段_ | +~40 B | +~40 B | JSON 字节数 |
| 每请求开销 | _未启用_ | ~1 µs | +1 µs | `BenchmarkDotNet` |

---

## 10. 未决问题

- [ ] **OQ-1**：参考 CA 策略文档放在哪？建议
  `tools/nip-ca-server/docs/policy.md`。负责人：Ori Lynn。
- [ ] **OQ-2**：L2 证书是否必须携带法律实体名字段（X.509
  `O`、`jurisdictionOfIncorporation`）？默认立场：是；细则
  延后到 CA 策略文档。
- [ ] **OQ-3**：Gateway Node（NPS-AaaS-Profile）是否可提升
  它所代理 Agent 的保证等级，还是要求后端 Action Node 独立
  校验？默认：Gateway 校验，后端通过 `X-NPS-Authed-Nid` header
  信任 Gateway 决策。待 AaaS working-group 签字。

---

## 11. 未来工作

- **NPS-RFC-0004**：NID 声誉日志（CT 风格）。与保证等级互补：
  等级关心**来源**，声誉关心**行为**。
- 后续：CA 被发现 L2 签发宽松时的批量吊销流。
- 后续：为 AaaS Profile 发布 `trusted_issuers` well-known 列表。

---

## 12. 参考

- NIST SP 800-63-3 —— "Digital Identity Guidelines"（IAL 定义）
- CA/B Forum Baseline Requirements —— EV / OV / DV 等级形状
- RFC 5280 §4.2.1.4 —— certificatePolicies
- NPS-RFC-0002 —— NID 证书 X.509 + ACME（前置）
- NPS-RFC-0004 —— NID 声誉日志（互补，进行中）
- 讨论：2026-04-20 关于反爬的评审意见

---

## 附录 A. 修订记录

| 日期 | 作者 | 变更 |
|------|------|------|
| 2026-04-21 | Ori Lynn | 初稿 |
| 2026-04-25 | Ori Lynn | 走 1.0 之前快速通道 Accept。已落地 spec：NPS-3 §5.1.1 保证等级 + IdentFrame `assurance_level` 字段、NPS-2 NWM `min_assurance_level` 字段、错误码 `NIP-ASSURANCE-MISMATCH` / `NIP-ASSURANCE-UNKNOWN` / `NWP-AUTH-ASSURANCE-TOO-LOW`。Phase 1 .NET 参考类型（`NPS.NIP.AssuranceLevel` 枚举、`IdentFrame.AssuranceLevel`、`NipVerifyContext.MinAssuranceLevel`、`NeuralWebManifest.MinAssuranceLevel`、相关错误/状态码常量）同时落地；verifier 主动强制保留为 opt-in（按 RFC §8.1：Phase 1 仅 parse，默认行为不变）。Phase 2（其余 5 SDK + 6 个 CA Server 通过 ACME 颁发 L1 证书 —— 依赖 RFC-0002）推迟到 v1.0-alpha.4。X.509 critical extension 翻转（§4.2）需与 RFC-0002 协调，**尚未生效**。AaaS-Profile §10 OQ-3（Gateway 强制 vs 后端节点强制）随 CR-0001 后续处理。|
