# Alpha.19 路线图历史对账

**状态**：已冻结为 alpha.19 债务清零候选

**范围**：P19-S03、P19-D03 与 P19-T03 中可能被误读为当前产品缺口的
历史声明。原 milestone 事实原样保留；本记录补充其时间边界、关闭版本和
当前证据。

机器可读记录见
[`alpha19-roadmap-history-reconciliation.json`](../spec/conformance/alpha19-roadmap-history-reconciliation.json)。
运行 `python3 tools/scripts/check-alpha19-roadmap-history.py` 可校验路线图标记与
证据路径。

## 声明处置

| ID | 历史声明 | 当时成立于 | 当前处置 |
|---|---|---|---|
| RH-01 | RFC-0001/0002/0003 使用 Accepted/Draft lifecycle 标签 | Phase 1／alpha.5 | 保留为有日期的快照；当前 lifecycle 见[状态记录](alpha19-p19-4-status-reconciliation.cn.md)。 |
| RH-02 | 所有 SDK 都缺 NDP DNS TXT 解析 | alpha.6 规划 | 已被 alpha.6 的六 SDK 实现和下方证据取代。 |
| RH-03 | RFC-0004 完整 client 仍不完整 | alpha.7 规划 | 已在 alpha.13–alpha.15 取代；六 SDK 现有 client/proof 行为，但并不把每个 SDK 都声明为 log operator。 |
| RH-04 | `nps-ingress` 仅是骨架 | alpha.5／alpha.11 | 已被 claim-scoped TLS 1.3、ALPN、mTLS、NID binding 与 proxy 证据取代；仍不声称完整 L2 certification。 |
| RH-05 | `nps-runner` 仅是骨架，OCI／renewal 仍开放 | alpha.5／alpha.17 | 已由 portable OCI SpawnSpec 与 lease renewal 实现取代；严格 L3 manifest 仍暴露真实 partial／unexecuted 用例。 |
| RH-06 | RFC-0002 被临时 PEN/OID 阻塞 | alpha.5 规划 | IANA 于 2026-05-08 分配 PEN 65715 后关闭；`1.3.6.1.4.1.99999` 仅是历史。 |
| RH-07 | alpha.13 详细实现计划把工作写成待开展 | alpha.11／alpha.13 规划 | 文件继续作为历史记录，并在相邻位置增加 supersession map，不静默改写原计划。 |

## 六 SDK DNS TXT 证据

| SDK | 源码 | 测试 |
|---|---|---|
| .NET | [`InMemoryNdpRegistry.cs`](https://github.com/labacacia/NPS-Dev/blob/main/impl/dotnet/src/NPS.NDP/Registry/InMemoryNdpRegistry.cs) | [`NdpRegistryTests.cs`](https://github.com/labacacia/NPS-Dev/blob/main/impl/dotnet/tests/NPS.Tests/Ndp/NdpRegistryTests.cs) |
| Python | [`registry.py`](https://github.com/labacacia/NPS-Dev/blob/main/impl/python/nps_sdk/ndp/registry.py) | [`test_ndp.py`](https://github.com/labacacia/NPS-Dev/blob/main/impl/python/tests/test_ndp.py) |
| TypeScript | [`ndp-registry.ts`](https://github.com/labacacia/NPS-Dev/blob/main/impl/typescript/src/ndp/ndp-registry.ts) | [`ndp.test.ts`](https://github.com/labacacia/NPS-Dev/blob/main/impl/typescript/tests/ndp.test.ts) |
| Go | [`registry.go`](https://github.com/labacacia/NPS-Dev/blob/main/impl/go/ndp/registry.go) | [`ndp_test.go`](https://github.com/labacacia/NPS-Dev/blob/main/impl/go/ndp/ndp_test.go) |
| Java | [`InMemoryNdpRegistry.java`](https://github.com/labacacia/NPS-Dev/blob/main/impl/java/src/main/java/com/labacacia/nps/ndp/InMemoryNdpRegistry.java) | [`NdpTest.java`](https://github.com/labacacia/NPS-Dev/blob/main/impl/java/src/test/java/com/labacacia/nps/ndp/NdpTest.java) |
| Rust | [`registry.rs`](https://github.com/labacacia/NPS-Dev/blob/main/impl/rust/nps-ndp/src/registry.rs) | [`ndp_tests.rs`](https://github.com/labacacia/NPS-Dev/blob/main/impl/rust/nps-ndp/tests/ndp_tests.rs) |

## Daemon 关闭证据

- `nps-ingress`：[当前架构](daemons/architecture.cn.md) 与
  [`NPS-NODE-L2-TLS-EVIDENCE.json`](https://github.com/labacacia/NPS-Dev/blob/main/tools/daemons/nps-ingress/conformance/NPS-NODE-L2-TLS-EVIDENCE.json)
  定义已实现 transport 边界，但不声明完整 profile。
- `nps-runner`：[当前架构](daemons/architecture.cn.md)、
  [`NPS-RUNNER-CAPABILITIES.json`](https://github.com/labacacia/NPS-Dev/blob/main/tools/daemons/nps-runner/conformance/NPS-RUNNER-CAPABILITIES.json)、
  [`SpawnSpecResolverTests.cs`](https://github.com/labacacia/NPS-Dev/blob/main/tools/daemons/nps-runner/tests/SpawnSpecResolverTests.cs)
  与 [`LeaseRenewalLoopTests.cs`](https://github.com/labacacia/NPS-Dev/blob/main/tools/daemons/nps-runner/tests/LeaseRenewalLoopTests.cs)
  证明 OCI／renewal 声明已关闭，同时保留真实 L3 限制。

## 边界

本次对账改变历史文字的解释方式，不改变文字写成时真实成立的事实；不把
component 证据转换为 deployment certification，不激活未来协议阶段，也不
发布 release。
