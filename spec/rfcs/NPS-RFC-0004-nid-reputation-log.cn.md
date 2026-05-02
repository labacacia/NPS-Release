[English Version](./NPS-RFC-0004-nid-reputation-log.md) | 中文版

---
**RFC 编号**：NPS-RFC-0004
**标题**：NID 声誉日志（Agent 版 Certificate Transparency）
**状态**：Accepted（Phase 1 —— 条目 wire 格式 + .NET 参考类型已落地）
**作者**：Ori Lynn <iamzerolin@gmail.com>（LabAcacia）
**Shepherd**：Ori Lynn（1.0 之前快速通道，见 `spec/cr/README.cn.md`）
**创建日期**：2026-04-21
**最后更新**：2026-04-26
**接受日期**：2026-04-26（1.0 之前快速通道；见 `spec/cr/README.cn.md`）
**激活日期**：_（首个参考日志运营方发版时填写，目标 v1.0-alpha.4）_
**取代**：_无_
**被取代于**：_无_
**影响的规范**：NPS-3 NIP、NPS-4 NDP、spec/services/NPS-AaaS-Profile.md、spec/error-codes.md
**影响的 SDK**：.NET、Python、TypeScript、Java、Rust、Go
---

# NPS-RFC-0004：NID 声誉日志（Agent 版 Certificate Transparency）

## 1. 摘要

为 NID 行为事件定义一个 Certificate-Transparency 风格的 **append-only
日志**。条目是针对特定 NID 的已签名观察（滥用报告、限速违规、合约
纠纷、吊销），由 AaaS gateway、审计方、CA 发布。新增一个可选 NDP
sub-path（`/.nid/reputation?nid=...`）允许任何方查询聚合记录。
配合 NPS-RFC-0003 的保证等级，Node 同时拿到 *来源*（这个 Agent
是谁？）和 *履历*（它有没有跑飞过？）。

## 2. 动机

接续驱动 RFC-0003 的同一条 2026-04-20 评审意见。保证等级回答了
"这个 Agent 是谁"，但没回答 "这个 NID 之前有没有闯过祸"。具体：

- 某个因滥用被吊销的 `L2` NID，必须能被任何考虑与之交易的 Node
  找到，不需要和吊销 CA 有预先关系。
- AaaS gateway 把某个 Agent 因爬虫行为踢出去时，应该能把这个
  信号发布出来，让其他 gateway 预先降级或拒绝该 NID。
- Node 应该能区分 "有 2 年干净历史的守规矩 L1 Agent" 和 "刚签
  出来的 L1，过去 24 小时里 40 次限速违规"。

现有域名声誉体系（Spamhaus、DNSBL）用私有名单。Certificate
Transparency 模型——可公开审计、append-only、防篡改、任何人可
查——直接映射到我们的问题，因为：

- NID 天生全球唯一（公钥）。
- AaaS 运营商天然产生这些信号，只是缺一个发布规范。
- 日志的 append-only 属性抑制审查：CA 不能悄悄删除一条吊销记录
  来恢复付费客户的声誉。

## 3. 非目标

- **不** 定义什么算滥用。日志运营方 / 审计方 MUST 公布各自标准；
  本 RFC 只定 wire 格式。
- **不** 要求每个 Node 都查日志。查询是建议性的；是否执行是
  per-Node 策略决定。
- **不** 强制唯一日志。预期会有多个独立日志；Node 选择信任哪些，
  对齐 CT 日志运营方模型。
- **不** 纳入人类操作员的个人数据。条目只涉及 NID 和元数据
  （时间戳、事件类型、证据 hash）；任何 doxx 级别的细节走带外。
- **不** 定义货币化或自定义封禁名单。商业威胁情报 feed 可在本
  日志之上构建，那是它们的层。

## 4. 详细设计

### 4.1 日志条目格式

条目是签名的 JSON 对象：

```json
{
  "v": 1,
  "log_id": "nid:ed25519:<log-operator-pubkey>",
  "seq": 42817,
  "timestamp": "2026-04-21T14:30:00Z",
  "subject_nid": "nid:ed25519:<agent-pubkey>",
  "incident": "rate-limit-violation",
  "severity": "moderate",
  "window": { "start": "2026-04-21T13:00:00Z", "end": "2026-04-21T14:00:00Z" },
  "observation": {
    "requests": 45000,
    "threshold": 300,
    "endpoint_hash": "sha256:<hash-of-node-origin>"
  },
  "evidence_ref": "https://log.example.com/evidence/42817",
  "evidence_sha256": "hex-of-blob",
  "issuer_nid": "nid:ed25519:<issuer-pubkey>",
  "signature": "base64url(Ed25519(canonical-entry-without-sig))"
}
```

字段语义：

| 字段 | 必填 | 说明 |
|------|------|------|
| `v` | 是 | Schema 版本；本 RFC 定义 `1` |
| `log_id` | 是 | append 本条目的日志运营方 NID |
| `seq` | 是 | 每日志单调递增序号 |
| `timestamp` | 是 | RFC 3339 UTC；日志运营方的时钟 |
| `subject_nid` | 是 | 本条目关于哪个 NID |
| `incident` | 是 | 枚举；见 §4.2 |
| `severity` | 是 | `info` / `minor` / `moderate` / `major` / `critical` |
| `window` | 否 | 观察覆盖的时间窗 |
| `observation` | 否 | 自由格式机读细节，按 incident 类型定义 |
| `evidence_ref` | 否 | 指向更丰富证据 blob 的 URL（日志、对话记录等） |
| `evidence_sha256` | 否 | 证据 blob 的 SHA-256，用于篡改检测 |
| `issuer_nid` | 是 | 主张该事件的一方 NID——MAY 等于 `log_id` |
| `signature` | 是 | `issuer_nid` 私钥对 canonical 形式的 Ed25519 签名 |

**签名规范化**：对条目对象（省略 `signature`）应用 JCS
（RFC 8785）。日志运营方 append 前 **先验证 issuer 签名**，然后用
**自己的密钥重签** 全条目以承诺序号和时间戳（双签名：issuer 证明
事件，日志运营方证明排序）。

### 4.2 事件词表

初始枚举（后续 RFC 可扩展）：

| 值 | 含义 |
|----|------|
| `cert-revoked` | CA 吊销了该 NID 的证书；subject_nid 与吊销对应 |
| `rate-limit-violation` | 持续违反已公布的速率限制 |
| `tos-violation` | 违反 AaaS gateway 已公布条款 |
| `scraping-pattern` | 观察到的行为匹配爬虫启发 |
| `payment-default` | 已承诺交易的 CGN / 法币支付违约 |
| `contract-dispute` | 异步 NOP 任务合约违约，未解决 |
| `impersonation-claim` | 第三方主张 subject_nid 冒充了自己 |
| `positive-attestation` | 显式正面信号（如审计通过） |

未知值 MUST 被日志运营方保留并返回给查询方——向前兼容。

### 4.3 日志运营方接口

#### 4.3.1 Phase 1 —— 提交与查询（当前）

日志运营方暴露两组 HTTP endpoint，经 NDP 发现：

```
POST /v1/log/entries        # 提交新条目（要 issuer 认证）
GET  /v1/log/entries?nid=<subject_nid>&since=<seq>  # 查询
```

#### 4.3.2 [Phase 2] —— Merkle 完整性证明（延后）

> **[Phase 2 —— 已延后]** 以下 endpoint 及 Merkle 结构不属于
> Phase 1 实现范围；按 §8.1 计划于 v1.0-alpha.5 交付。

```
GET  /v1/log/sth            # signed tree head（Merkle root + seq + timestamp）
GET  /v1/log/proof?seq=<n>&tree_size=<m>  # 包含证明
```

Merkle 结构对齐 RFC 9162（CT 2.0）：叶子是 canonical 条目；内部
节点是 SHA-256 hash；signed tree head（STH）承诺当前 root，
由 `log_id` 签名。这让查询方可以密码学证明条目被收录，不必下载
整个日志。

### 4.4 Manifest / NDP 变更

> **[Phase 2 —— 已延后]** 本节所有内容按 §8.1 计划于 v1.0-alpha.5
> 交付。`/.nid/reputation` 和 `reputation_policy` 均不在 Phase 1
> 实现范围内。

NDP 新增一个可选 well-known 路径用于声誉发现：

```
GET /.nid/reputation?nid=<nid>
    → 声称有该 NID 条目的日志运营方 URL 数组
```

这是**发现提示**，不是事实源——Node 仍从自己信任的日志运营方拉
条目。

NWM 可选声明：

```yaml
# /.nwm 片段
reputation_policy:
  required_logs: ["log:labacacia-primary", "log:some-industry-consortium"]
  reject_on:
    - { incident: "cert-revoked", severity: ">=minor" }
    - { incident: "scraping-pattern", severity: ">=major", within_days: 30 }
```

不查声誉的 Node 直接省略此字段。

### 4.5 STH Gossip 协议

> **[Phase 3 — v1.0-alpha.5]** 支持跨日志一致性验证与分叉检测。
> OQ-1 已决议采用轻量 NPS 原生变体（类比 RFC 9162 §8.1.4，但去掉
> HTTP-header 传输，改用 NPS 错误码）。

#### 4.5.1 Gossip 端点

每个日志运营方 MUST 暴露：

```
GET /v1/log/gossip/sth
```

**响应**（JSON）：

```json
{
  "own_sth": {
    "tree_size": 42817,
    "timestamp": "2026-05-01T10:00:00Z",
    "sha256_root_hash": "hex-of-merkle-root",
    "log_id": "nid:ed25519:<log-operator-pubkey>",
    "signature": "base64url(Ed25519(jcs(own_sth without signature)))"
  },
  "peer_sths": [
    {
      "log_id": "nid:ed25519:<peer-pubkey>",
      "received_at": "2026-05-01T09:59:30Z",
      "sth": { /* 结构同 own_sth */ }
    }
  ]
}
```

`peer_sths` 包含从每个已配置对端收到的最新已验证 STH。查询此端点
的客户端可无需直接联系对端即完成交叉核对。

#### 4.5.2 Gossip 推送周期

配置了 `peers` 列表的日志运营方 MUST 运行后台 gossip 周期：

1. **拉取** 每个对端的 `GET /v1/log/gossip/sth`（默认间隔 30 s）。
2. **验证** 对端 `own_sth.signature` 是否与其 `log_id` 公钥匹配。
3. **单调性检查**：对端新的 `tree_size` MUST ≥ 该 `log_id` 上次接受的
   `tree_size`。减小视为分叉尝试——运营方 SHOULD 向本地审计写入
   `LOG-FORK-DETECTED` 事件，并暂停与该对端的交互，等待人工审查。
4. **一致性证明**（SHOULD）：当对端 `tree_size` 增大时，运营方 SHOULD
   从 `GET /v1/log/proof?from=<prev_size>&to=<new_size>` 拉取 RFC 9162
   一致性证明并验证后再接受新 STH。验证失败即潜在分叉。
5. **缓存** 已接受的对端 STH 于内存中，并通过 `/gossip/sth` 对外服务。

#### 4.5.3 配置

日志运营方配置新增可选 `peers` 列表：

```json
{
  "peers": [
    { "log_id": "nid:ed25519:<peer>", "endpoint": "https://log2.example.com" }
  ],
  "gossip_interval_s": 30
}
```

`gossip_interval_s` 默认 30；最小 10；最大 3600。

#### 4.5.4 错误码（Phase 3 新增）

| 错误码 | NPS 状态 | 说明 |
|--------|---------|------|
| `NIP-REPUTATION-GOSSIP-FORK` | `NPS-SERVER-INTERNAL` | 跨节点 STH 一致性检查失败；可能检测到分叉 |
| `NIP-REPUTATION-GOSSIP-SIG-INVALID` | `NPS-CLIENT-BAD-FRAME` | gossip 交换中对端 STH 签名验证失败 |

### 4.7 错误码

`spec/error-codes.md` 新增：

| 错误码 | NPS 状态 | 说明 |
|--------|---------|------|
| `NWP-AUTH-REPUTATION-BLOCKED` | `NPS-AUTH-FORBIDDEN` | 声誉策略命中了 reject 规则 |
| `NIP-REPUTATION-LOG-UNREACHABLE` | `NPS-DOWNSTREAM-UNAVAILABLE` | 策略评估期间所需日志运营方不可达 |
| `NIP-REPUTATION-ENTRY-INVALID` | `NPS-CLIENT-BAD-FRAME` | 条目签名无效或 canonical 形式畸形 |

### 4.6 状态机 / 流程

声誉闸的准入：

```
Agent                      Node                         Log Operator
  │                          │                               │
  │── HelloFrame ──────────→ │                               │
  │── IdentFrame (NID) ────→ │                               │
  │                          │ ── GET entries?nid=... ─────→ │
  │                          │ ←── 200 + entries ──────────  │
  │                          │  evaluate reject_on rules     │
  │                          │                               │
  │ ←── (accept | 403) ───── │                               │
```

热路径上，Node SHOULD 用短 TTL（默认 60 s）缓存日志结果并异步
刷新。硬性 `reject_on: cert-revoked` SHOULD 每次连接都同步查，
可以用 OCSP-stapling 风格的预取满足。

### 4.8 向后兼容性

- 旧 Node（无 `reputation_policy`）？完全兼容——日志被忽略。
- 旧 Agent？wire 层永远不受影响；只是它们的 NID 会出现在日志里。
- NDP sub-path 是叠加式的（按 `spec/rfcs/README.md` 的 RFC 流程要求）。

---

## 5. 备选方案

### 5.1 Node 各自的私有名单

每个 Node 维护自己的 NID 黑名单/白名单。

- **代价**：每 Node 运维负担；重复劳动；生态级别无共享信号。
- **结论**：作为生态级方案拒绝。Node 仍可在此之上叠加私有名单。

### 5.2 只靠 CA 内部吊销（不要声誉）

完全依靠 NPS-RFC-0002 的 CRL / OCSP 做恶意方信号。

- **代价**：CRL 只带 "吊销 yes/no"，没有事件类型、严重程度、
  观察者。错过最常见的信号（限速违规、爬虫），这些够不上吊销
  但与策略相关。
- **结论**：作为"完备方案"拒绝；CRL 仍是吊销必备，但不是完整
  答案。

### 5.3 中心化单一声誉注册表

LabAcacia 运营一个单独注册表。

- **代价**：单点审查；LabAcacia 成为守门人；合规暴露；利益冲突
  （LabAcacia 同时跑 NIP CA）。
- **结论**：拒绝。LabAcacia SHOULD 运营一个参考日志，MUST NOT
  是唯一的。

### 5.4 不动

- **代价**：RFC-0003 单独给 Node 来源信息但没有行为历史。规模
  化爬虫问题部分未解。
- **结论**：拒绝。

---

## 6. 缺陷与风险

- **日志运营方滥权**：流氓运营方可以对诚实 NID 灌水。对策：Node
  只信任自己 opt-in 的日志；每条带 `issuer_nid`，伪造条目可检测；
  双签名模型让运营方无法静默把条目归咎给第三方。
- **隐私**：声誉给 NID 创造了永久历史。想重开的 Agent 必须轮换
  NID，同时也会重置保证等级。这是**故意的**——重置的代价是防止
  声誉洗白。
- **延迟**：同步日志查询每连接加一个 RTT。对策：缓存、异步刷新、
  OCSP 风格预取。
- **假阳性**：自动化爬虫检测并不完美。对策：严重程度分级 +
  per-Node 可调 reject 规则 + `evidence_ref` 用于申诉。
- **法律风险**：运营方发布不当行为指控可能招来诽谤诉讼。对策：
  优先客观、机器可观察的事件类型；强烈建议带 `evidence_ref`。
- **可逆性**：中等。日志格式有版本号；日志运营方可以逐个废弃
  而不改协议。

---

## 7. 安全性考量

- **日志篡改**：Merkle 树 + 定期 STH 公布阻断。**[Phase 2]** Node
  SHOULD 在下载条目时拉 STH 并验证包含证明。
- **条目伪造**：不拿到 `issuer_nid` 私钥不可能（canonical 形式的
  签名）。日志运营方再签一次承诺排序。
- **重放**：`seq` + `timestamp` + log_id 唯一标识条目；包含证明
  绑定特定 tree head。
- **日志运营方 DoS**：运营方 MUST 按 `issuer_nid` 限速提交，并
  要求写入认证账户。
- **审查**：Merkle 树阻断静默删除。运营方尝试分叉日志可以通过
  STH 不一致检测——这是 CT "gossip 协议" 要求，SHOULD 由查询方
  互相对账 STH 实现。
- **跨日志关联**：同一个 NID 可能出现在多个日志。Node 按自己
  NWM 策略独立评估。

---

## 8. 实施计划

### 8.1 阶段划分

| 阶段 | 范围 | 出口条件 |
|------|------|----------|
| 1 | .NET 参考日志运营方；条目格式；submit + query HTTP API；暂无 Merkle 证明 | 单测绿；另一 SDK 可查询 |
| 2 | Merkle 树 + STH + 包含证明；所有 SDK 实现 NDP `/.nid/reputation`；NWM `reputation_policy` 解析 | Interop：2 家独立日志 + 2 SDK 客户端互相对账 |
| 3 | AaaS Profile L2 层默认 `reputation_policy`；参考日志之间 STH gossip（§4.5）| 日志 60s 内 STH 一致 ✅ v1.0-alpha.5 |
| 4 | 若有未签名条目的实验 flag 则废弃 | 清理 |

### 8.2 SDK 覆盖矩阵

| SDK | 负责人 | 状态 | 备注 |
|-----|--------|------|------|
| .NET | Ori Lynn | Phase 1+2 ✅ / Phase 3 ✅ | 参考日志运营方；Phase 3 gossip 在 alpha.5 落地 |
| Python | _待定_ | pending | Phase 1 只实现 client；Phase 2 运营方 |
| TypeScript | _待定_ | pending | — |
| Java | _待定_ | pending | — |
| Rust | _待定_ | pending | — |
| Go | _待定_ | pending | — |

### 8.3 测试计划

1. 条目 round-trip：issuer 签 → 日志运营方 append → 查询方验证
   双签名。
2. **[Phase 2]** 包含证明：查询 `seq=N` 的条目，对 `tree_size >= N+1` 的
   STH 验证。
3. **[Phase 2]** 篡改检测：存储中改条目字节 → STH 证明失败。
4. NWM `reject_on` 匹配：`severity: ">=major"` 匹配 `major`、
   `critical`，不匹配 `moderate`。
5. 策略评估时日志不可达：`NIP-REPUTATION-LOG-UNREACHABLE`，
   Node 按策略 fallback（`fail-open` / `fail-closed` 可配）。
6. 未知 incident 值被保留并返回给查询方。

### 8.4 基准

- 日志查询延迟：缓存命中目标 ≤ 20 ms；冷拉包含 STH 验证 ≤ 200 ms。
- 条目提交吞吐：参考运营方 MUST 在普通硬件上维持 1000 条/s。
- 内存：10M 条目 Merkle 树 ≤ 1 GiB 常驻。

---

## 9. 实测数据

暂无。在 `Accepted` 前：
- 参考 .NET 日志运营方实现。
- 端到端场景：AaaS gateway 发布 `scraping-pattern` 条目；第二家
  gateway 查询并应用 `reject_on`。
- Merkle 包含证明基准。

| 指标 | 基线 | 提议 | 差值 | 方法 |
|------|------|------|------|------|
| 日志查询延迟（热缓存） | _N/A_ | ≤ 20 ms | — | 墙钟 |
| 条目提交吞吐 | _N/A_ | ≥ 1000/s | — | 压测 |
| Merkle 树体积（10M 条目） | _N/A_ | ≤ 1 GiB | — | 常驻集 |

---

## 10. 未决问题

- [x] **OQ-1**：STH gossip 协议——复用 CT gossip（RFC 9162
  §8.1.4）还是定义一个更轻变体？**已决议**：采用轻量 NPS 原生变体（§4.5）。
- [ ] **OQ-2**：申诉机制——`subject_nid` 能否对自己被指控的条目
  发布一个 `dispute` 条目？默认：可以，作为 `incident: self-dispute`
  引用原 `seq`。
- [ ] **OQ-3**：日志是存证据 blob 还是只存 hash？默认：只存 hash；
  blob 由 issuer 在 `evidence_ref` 自行托管。
- [ ] **OQ-4**：条目 TTL / 过期。与 GDPR 式"被遗忘权"的交互？
  默认：无过期；Merkle 证明要求保留。合规处理延后。

---

## 11. 未来工作

- 后续 RFC：标准化自动化爬虫检测标准，使 `scraping-pattern` 条目
  可复现。
- 后续 RFC：CGN 质押 / 担保原语——`issuer_nid` 对条目质押 CGN；
  条目被证伪时 slash。
- 后续 RFC：日志运营方分级认证。

---

## 12. 参考

- RFC 9162 —— "Certificate Transparency Version 2.0"
- RFC 8785 —— "JSON Canonicalization Scheme (JCS)"
- RFC 3339 —— "Date and Time on the Internet"
- NPS-RFC-0002 —— NID 证书 X.509 + ACME（证书吊销来源）
- NPS-RFC-0003 —— 保证等级（互补：来源 vs 行为）
- 讨论：2026-04-20 关于反爬的评审意见

---

## 附录 A. 修订记录

| 日期 | 作者 | 变更 |
|------|------|------|
| 2026-04-21 | Ori Lynn | 初稿 |
| 2026-04-26 | Ori Lynn | 走 1.0 之前快速通道 Accept。Phase 1 spec 改动已落地：NPS-3 §5.1.2 声誉日志条目（12 字段签名 JSON、8 项 `incident` 枚举、5 级 `severity` 枚举、JCS 双签名规则），错误码 `NIP-REPUTATION-ENTRY-INVALID` / `NIP-REPUTATION-LOG-UNREACHABLE` / `NWP-AUTH-REPUTATION-BLOCKED`，新增状态码 `NPS-DOWNSTREAM-UNAVAILABLE`。Phase 1 .NET 参考类型已落地于 `NPS.NIP.Reputation.*`。Phase 2（Merkle 树 + STH + inclusion proof + NDP `/.nid/reputation` 发现 + NWM `reputation_policy` 解析）按 RFC §8.1 推迟到 v1.0-alpha.4。Phase 3（AaaS Profile L2 默认策略 + STH gossip）推迟到 alpha.5+。|
| 2026-05-01 | Ori Lynn | Phase 3 落地（v1.0-alpha.5）：新增 §4.5 STH Gossip 协议（gossip 端点 + 推送周期 + 配置 + 错误码）；§4.7（原 §4.5）错误码表新增 `NIP-REPUTATION-GOSSIP-FORK` / `NIP-REPUTATION-GOSSIP-SIG-INVALID`；原 §4.7 向后兼容性改编号为 §4.8；OQ-1 已关闭；SDK 矩阵更新 .NET Phase 3 ✅；nps-ledger GossipState + GossipService + `GET /v1/log/gossip/sth` 已落地。|
