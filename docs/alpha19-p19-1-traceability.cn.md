# NPS alpha.19 P19-1 规范与 fixture 冻结

**状态**：评审候选
**日期**：2026-08-31
**父清单**：[P19-0 债务清单](alpha19-debt-inventory.cn.md)
**治理**：[ChangeControl EPIC-004](https://github.com/innolotus/ChangeControl/issues/4)

## 目的

P19-1 在 runtime 开始前，把已接受的 alpha.19 债务边界转换为可测试的协议要求。
本轮冻结 46 个稳定 requirement ID 与五组共享语义向量。fixture 文件是后续六 SDK
runner 的契约；文件存在不代表 runner 已经执行它。

## 可追踪性矩阵

| 协议 | 候选版本 | 债务 ID | 规范要求 | 共享向量 | 产品 Issue |
|---|---:|---|---|---|---|
| NCP | 0.12 | P19-P01、P19-P06 | NCP-P19-01–NCP-P19-08 | `spec/conformance/ncp/runtime_hardening_vectors.json` | [#92](https://github.com/labacacia/NPS-Dev/issues/92) |
| NWP | 0.22 | P19-P04、P19-S01 | NWP-P19-01–NWP-P19-10 | `spec/conformance/nwp/alpha19_hardening_vectors.json` | [#95](https://github.com/labacacia/NPS-Dev/issues/95) |
| NIP | 0.15 | P19-P05、P19-T01、P19-S01 | NIP-P19-01–NIP-P19-10 | `spec/conformance/nip/renewal_revocation_vectors.json` | [#95](https://github.com/labacacia/NPS-Dev/issues/95) |
| NDP | 0.13 | P19-P02、P19-S01、P19-S03 | NDP-P19-01–NDP-P19-08 | `spec/conformance/ndp/recovery_fence_vectors.json` | [#93](https://github.com/labacacia/NPS-Dev/issues/93) |
| NOP | 0.10 | P19-P03、P19-S01 | NOP-P19-01–NOP-P19-10 | `spec/conformance/nop/replay_retention_vectors.json` | [#94](https://github.com/labacacia/NPS-Dev/issues/94) |

## 兼容性决策

- NCP 0.12 改变 runtime policy，不改变 frame layout。QUIC/TLS 0-RTT NPS 数据因
  replay 敏感而被拒绝；握手确认后仍可重试。
- NWP 0.22 新增可选 manifest 与 SubscribeFrame 字段。v0.21 producer 不声明
  `subscription_policy` 时保留无 lease 的兼容行为。
- NIP 0.15 不新增 frame 字段；它冻结 CA/verifier 的时序与结果策略，并继续关闭
  Phase-3 flag day。
- NDP 0.13 不新增 frame 字段；只有两个要求持久化的 Registry profile 强制 durable
  recovery，`local-dev` 仍明确为 volatile。
- NOP 0.10 在不改变既有 TaskFrame layout 的前提下新增 server-side replay policy 与
  error；既有 result TTL 继续作为 result retention 输入。
- minimum-compatible floor 保持不变，除非六 SDK 可执行证据证明必须提高。

## P19-1 评审关口

满足以下全部条件后，P19-1 才可评审：

- 每个 requirement ID 在 EN/CN 各出现一次，规范强度等价；
- 每个 requirement ID 至少被一个合法 JSON vector 引用；
- version matrix、EN/CN header、dependency、README badge/table、
  `frame-registry.yaml`、error namespace 与 maintainer table 一致；
- 适用处覆盖 positive、negative、boundary、restart/partition 与 failure case；
- 规范不得声称六 SDK 已执行新增向量；
- 不创建 tag/package/image，不提高 compatibility floor，也不引入 alpha.20-only 设计。

本评审单元接受后才进入 P19-2。runtime 实现和各语言 vector adapter 属于 P19-2，
不属于本文件。
