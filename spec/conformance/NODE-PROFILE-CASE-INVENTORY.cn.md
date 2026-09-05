[English](./NODE-PROFILE-CASE-INVENTORY.md) | 中文版

# Node L1/L2 用例证据盘点

**状态**：Alpha.19 技债收口证据图
**日期**：2026-09-05
**机器可读源**：[`node-profile-case-inventory.json`](./node-profile-case-inventory.json)

本盘点明确区分 catalog 中的用例标题与可执行 conformance 证据，不构成实现
认证。认证仍必须满足套件定义的完整 IUT、peer、环境、结果 manifest 与签名规则。

## 分类

| 分类 | 含义 |
|---|---|
| `executable_iut` | 可运行测试针对指定实现边界执行该用例。|
| `component_executable` | 有可运行组件证据，但没有完整 paired-peer IUT fixture。|
| `partial` | 部分验收条件已执行；至少一个条件或 fixture 缺失。|
| `catalog_only` | 已写入规范和 catalog，但没有 case-complete 可执行证据。|
| `not_applicable_reference_iut` | 仅参考 IUT 明确拒绝该可选能力。|

`component_executable` 与 `partial` 都不等于通过。Catalog entry 是元数据，
不是测试结果。

## 结果

| Profile | 规范标题 | .NET catalog | IUT 可执行 | 仅组件 | 部分 | 仅 catalog | 参考 IUT N/A |
|---|---:|---:|---:|---:|---:|---:|---:|
| Node L1 v0.1 | 20 | 20 | 8 | 0 | 6 | 5 | 1 |
| Node L2 v0.7 | 38 | 38 | 6 | 16 | 11 | 5 | 0 |

L1 套件实际有 20 个 case 标题，不是 21 个；旧正文计数属于编辑错误，修正时
没有改变用例集合。旧 .NET L2 catalog 只包含 16 个 topology/TLS 用例；v0.6
先把漂移关闭到 31 个。v0.7 又为当前合同 AaaS L2-01..L2-07 加入
`TC-N2-AaaS-01..07`，现覆盖全部 38 个标题。

## 近期可执行缺口

- L1：通用 frame echo、隔离环境中的真实默认端口、完整 root identity 与 peer
  Ident、ResolveFrame 未知目标、可选 Graph、指定 100-QPS 环境，以及逐方向
  structured frame log。
- L2 topology：depth cap 拒绝、真实 NDP 驱动的 join/TTL leave、snapshot filter
  拒绝，以及外部 paired-peer Anchor fixture。
- L2 Bridge：带 conformance ID 的 harness、歧义拒绝、完整 foreign error mapping，
  以及未声明方向/协议拒绝。
- L2 HA：终态 failover、quorum 驱动只读、standby 写拒绝、旧 leader 围栏，
  以及完整的单 Anchor 无 HA event 用例。

JSON inventory 记录逐 case 分类、证据路径与缺口，并机械校验两份 Markdown 套件
和 .NET catalog 的集合一致性。

## 非声明

- 本盘点不推导完整 Node L1 或 L2 认证。
- 组件测试不会被改称 paired-peer IUT pass。
- npsd 对 L1 可选 push 的 N/A 处置不豁免其他 IUT。
- 不隐含版本、tag、包、镜像或 release 发布。
