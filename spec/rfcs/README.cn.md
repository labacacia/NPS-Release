[English Version](./README.md) | 中文版

# NPS RFC 流程

本目录收录 **Request for Comments (RFC)**——NPS 协议族非平凡变更从
提案、讨论到决议的正式记录。RFC 是 **思考文档，不是实现计划**：
存在的目的是让"为什么这样决定"的论证比决定本身活得更久。

---

## 什么情况下必须写 RFC

任何 PR 在动手之前 **必须** 先开 RFC，如果它：

- 占用 Reserved 位 / 码 / 命名空间（例如 encoding tier `0x02`、新帧类型、新的 NPS 状态码段）
- 在 `spec/frame-registry.yaml` 里增删帧
- 引入新节点类型或新 endpoint sub-path
- 改变现有帧的 on-the-wire 语义（不管是显式 breaking 还是静默变更）
- 定义新的鉴权方式或密钥算法
- 向协议族新增一层（`NPS-6-*`）
- 抬高 `min_agent_version`

以下 **建议** 开 RFC（shepherd 同意可以免）：

- 跨 SDK 的大型重构
- 新的兼容层桥接（例如接第三方协议）
- 策略类变更：版本号、废弃窗口、发布节奏

以下 **不需要** RFC：

- Bug 修复与规范澄清（错别字、歧义消除）
- SDK 非破坏性追加（新的便利函数、已可扩展结构的新可选字段）
- 文档、基准、示例、CI 变更
- 工具实现（例如新增一个 NIP CA Server 后端）

> **经验法则：** 如果合上 PR 就意味着 6 个 SDK 要么跟进要么显式拒绝，它就要走 RFC。

---

## 生命周期

```
  Draft ─────► Accepted ─────► Active ─────► Superseded
   │              │
   │              └──► Withdrawn   （作者撤回）
   └──► Rejected
```

| 状态 | 含义 | 如何切换 |
|------|------|----------|
| `Draft` | 作者起草/修订中；PR 已开，对 `dev` | 作者继续推提交 |
| `Accepted` | 合入 `dev`；各 SDK 可以开始实现 | shepherd 达到批准阈值后合 PR |
| `Active` | 至少一个参考 SDK 已落地；帧/规范变更进主线 | 后续 PR 切状态；要求 ≥1 SDK + CI 绿 |
| `Superseded` | 被后续 RFC 取代 | 取代它的 RFC 填 `Supersedes:`，本 RFC 填 `Superseded-By:` |
| `Withdrawn` | 作者不再推进；作为历史保留 | 作者改头部 |
| `Rejected` | 讨论达成"拒绝"共识；作为历史保留 | shepherd 改头部并写拒绝原因 |

Rejected 与 Withdrawn 的 RFC **永不删除**——论证过程本身就是产出。

---

## 批准阈值

Accept 的最低门槛：

- **1 位 shepherd**（非作者的 maintainer）显式在 PR 上批准
- **公开讨论窗口 ≥ 7 天**（破坏性变更窗口更长，见下）
- **Open questions** 全部解决或显式延后（有 tracking issue）
- **SDK 覆盖矩阵**每格要么负责人签字，要么声明一个非阻塞的 follow-up

加长窗口：

| 变更类型 | 最短讨论窗口 |
|----------|-------------|
| 新帧 / 新 Reserved 位分配 | 14 天 |
| 破坏性 wire-format 变更 | 21 天 |
| 抬高 `min_agent_version` | 21 天 + 1 个月废弃通告期 |

---

## 编号与文件命名

- `NPS-RFC-{NNNN}-{kebab-slug}.md`（英文主稿）
- `NPS-RFC-{NNNN}-{kebab-slug}.cn.md`（中文副本）

`NNNN` 是四位数字补零，在 **PR 开立时分配**，单调递增。即使是
Withdrawn / Rejected 的 RFC，编号也永不复用。

---

## 如何提交 RFC

1. 复制 `template.md` → `NPS-RFC-{下一个编号}-{slug}.md`
2. 复制 `template.cn.md` → `NPS-RFC-{下一个编号}-{slug}.cn.md` 并翻译
3. 填头部信息（`Status: Draft`）
4. 对 `dev` 开 PR，title 格式：
   `rfc: NPS-RFC-{NNNN} — {一句话摘要}`
5. 指派至少一位 shepherd 审阅
6. 在 PR 上迭代；squash-merge 无所谓，RFC 文件内部的修订史才是关键

合入 `dev` 且状态为 `Accepted` 后，各 SDK 可开始实现工作。等任一
参考 SDK 发版包含该变更，再提一个 follow-up PR 把 `Status:`
切到 `Active`。

---

## 索引

| 编号 | 标题 | 状态 | 接受日期 | 取代 |
|------|------|------|----------|------|
| [0001](./NPS-RFC-0001-ncp-connection-preamble.cn.md) | 为 NCP 原生模式加入连接前导，用于流量识别 | Draft | _—_ | _—_ |
| [0002](./NPS-RFC-0002-x509-acme-nid-certs.cn.md) | NID 证书改用 X.509 + ACME | Draft | _—_ | _—_ |
| [0003](./NPS-RFC-0003-agent-identity-assurance-levels.cn.md) | Agent 身份三级保证等级（反爬 / 信任分流） | Draft | _—_ | _—_ |
| [0004](./NPS-RFC-0004-nid-reputation-log.cn.md) | NID 声誉日志（Agent 版 Certificate Transparency） | Draft | _—_ | _—_ |

---

## 参考资料

- Rust RFC 流程 —— https://github.com/rust-lang/rfcs
- Python PEP —— https://peps.python.org/
- Kubernetes KEP —— https://github.com/kubernetes/enhancements
- IETF RFC 7282（《On Consensus and Humming in the IETF》）
