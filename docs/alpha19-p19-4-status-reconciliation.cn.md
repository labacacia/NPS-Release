# Alpha.19 CR/RFC 状态对账

**状态**：已冻结为 alpha.19 债务清零候选

**范围**：当前 lifecycle、SDK coverage 与问题处置；历史 roadmap 订正属于
另一项 P19-4 任务。

机器可读记录见
[`alpha19-status-reconciliation.json`](../spec/conformance/alpha19-status-reconciliation.json)。
运行 `python3 tools/scripts/check-alpha19-status-reconciliation.py` 可对规范、
索引和源码证据做机械校验。

## Lifecycle 真值

| 记录 | 当前状态 | 原因 |
|---|---|---|
| NPS-CR-0011 | Implemented | NWP 0.21、19 条共享向量、六 SDK 与 server/benchmark 证据均已存在；两份 CR 索引现与正文一致。 |
| NPS-RFC-0001 | Active | .NET 参考 helper 已在 Phase 1 落地，六 SDK 均有 preamble helper/test；Phase 3/4 兼容迁移未激活。 |
| NPS-RFC-0002 | Active | 六 SDK 与六 CA surface 均有 X.509 / ACME `agent-01` 源码和测试；v2 default flip 与删除 v1 未激活。 |
| NPS-RFC-0003 | Active | 六 SDK 已实现 Phase 1–2 assurance 类型、签发、验证与 opt-in enforcement；Phase 3 flag day 不进入 alpha.19。 |
| NPS-RFC-0004 | Active | 六 SDK 都有 client/proof 行为；仅 .NET + `nps-ledger` 声明参考 operator 与 gossip 实现。 |
| NPS-RFC-0005 | Active | Accepted 的可用性、进程内 ban 和 fail-most-restrictive 默认值保持不变。 |
| NPS-RFC-0006 | Accepted | 英中状态现已一致；daemon 候选证据不会暗中把 RFC 提升为 Active。 |

## SDK matrix 真值

RFC-0001 至 RFC-0004 不再保留匿名 `_TBD_` owner 或过期 `pending` 单元格；
每个 implemented cell 都能追到受维护的 source/test surface。表格按能力声明：
RFC-0004 client/proof 行不等同于 log-operator 声明。

CR-0003 与 CR-0005 已记录六种 CA surface 的实现。CR-0006 记录六 SDK 的
`SubscribeFrame` 与 cursor 支持，同时保留真实的非规范缺口：Python 与 Rust
尚未暴露高层 subscribe client 便利 API。

## 问题处置

Proposal 阶段的默认值现已变成明确决议或明确未来范围。关键当前契约选择为：

- RFC-0001 保留八字节、client-to-server preamble 与固定 10 秒读取超时；
  ALPN `nps/1.0` 由 RFC-0006 管理。
- RFC-0003 在 Phase 1–2 不标准化法律实体 X.509 字段；实际执行 action 的 Node
  独立强制 `min_assurance_level`。
- RFC-0004 只保存 signed entry 与证据 hash/reference，不托管 blob；append-only
  协议没有 TTL。
- RFC-0005 保留 `on_log_unavailable=allow`、进程内可移植 ban state 与多日志
  最严格结果规则。
- RFC-0006 当前 QUIC 契约限定为 QUIC v1；QUIC v2 需要未来 RFC 修订。

这些决议不会激活机器记录中明确延后的 phase transition，也不构成 release 发布声明。
