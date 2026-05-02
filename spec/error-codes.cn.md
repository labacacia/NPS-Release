[English Version](./error-codes.md) | 中文版

# NPS 统一错误码命名空间

**Version**: 1.2  
**Date**: 2026-05-01  

错误码格式：`{PROTOCOL}-{CATEGORY}-{DETAIL}`

NPS 采用两级错误体系：
1. **协议错误码**（本文档）：具体错误标识，前缀标识所属协议
2. **NPS 状态码**：传输层状态分类与 HTTP 映射，见 [status-codes.cn.md](status-codes.cn.md)

---

## NCP 错误码

| 错误码 | NPS 状态码 | 描述 |
|--------|-----------|------|
| `NCP-ANCHOR-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | anchor_ref 引用的 Schema 不存在，Agent 须重新获取 AnchorFrame |
| `NCP-ANCHOR-SCHEMA-INVALID` | `NPS-CLIENT-BAD-FRAME` | AnchorFrame 中 Schema 格式不合法 |
| `NCP-ANCHOR-ID-MISMATCH` | `NPS-CLIENT-CONFLICT` | 相同 anchor_id 收到不同 Schema（锚点污染攻击防御）|
| `NCP-FRAME-UNKNOWN-TYPE` | `NPS-CLIENT-BAD-FRAME` | 未知帧类型码 |
| `NCP-FRAME-PAYLOAD-TOO-LARGE` | `NPS-LIMIT-PAYLOAD` | Payload 超过协商的 max_frame_payload |
| `NCP-FRAME-FLAGS-INVALID` | `NPS-CLIENT-BAD-FRAME` | Flags 字段中保留位非零 |
| `NCP-STREAM-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | StreamFrame 序号不连续 |
| `NCP-STREAM-NOT-FOUND` | `NPS-STREAM-NOT-FOUND` | stream_id 引用的流��存在 |
| `NCP-STREAM-LIMIT-EXCEEDED` | `NPS-STREAM-LIMIT` | 超出单连接最大并发流数 |
| `NCP-ENCODING-UNSUPPORTED` | `NPS-SERVER-ENCODING-UNSUPPORTED` | 请求的编码 Tier 不被支持 |
| `NCP-ANCHOR-STALE` | `NPS-CLIENT-CONFLICT` | anchor_ref 存在但 Schema 已更新；响应通过 CapsFrame.inline_anchor 携带最新 AnchorFrame |
| `NCP-DIFF-FORMAT-UNSUPPORTED` | `NPS-CLIENT-BAD-FRAME` | DiffFrame 使用了接收方不支持的 patch_format（如 binary_bitset）|
| `NCP-VERSION-INCOMPATIBLE` | `NPS-PROTO-VERSION-INCOMPATIBLE` | 客户端 min_version 高于 Server 支持的最高版本（握手失败）|
| `NCP-STREAM-WINDOW-OVERFLOW` | `NPS-STREAM-LIMIT` | 发送方在应用层流量控制窗口耗尽后继续发送 StreamFrame |
| `NCP-ENC-NOT-NEGOTIATED` | `NPS-CLIENT-BAD-FRAME` | 收到 ENC=1 帧，但会话未协商 E2E 加密算法（HelloFrame 中未声明）|
| `NCP-ENC-AUTH-FAILED` | `NPS-CLIENT-BAD-FRAME` | E2E 加密 Auth Tag 验证失败，帧可能被篡改 |
| `NCP-PREAMBLE-INVALID` | `NPS-PROTO-PREAMBLE-INVALID` | 原生模式连接首 8 字节非常量前导 `b"NPS/1.0\n"`；服务端静默关闭连接，不返回 ErrorFrame（NPS-RFC-0001） |

---

## NWP 错误码

| 错误码 | NPS 状���码 | 描述 |
|--------|-----------|------|
| `NWP-AUTH-NID-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` | Agent scope 不覆盖目标节点路径 |
| `NWP-AUTH-NID-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` | NID 证书已过期 |
| `NWP-AUTH-NID-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` | NID 已被吊销 |
| `NWP-AUTH-NID-UNTRUSTED-ISSUER` | `NPS-AUTH-UNAUTHENTICATED` | NID 颁发者不在 trusted_issuers 中 |
| `NWP-AUTH-NID-CAPABILITY-MISSING` | `NPS-AUTH-FORBIDDEN` | Agent 缺少节点要求的能力（如 nwp:query）|
| `NWP-AUTH-ASSURANCE-TOO-LOW` | `NPS-AUTH-FORBIDDEN` | Agent 保证等级低于节点 `min_assurance_level`（NPS-2 §4.1）或 ActionSpec 的 per-action 覆盖（§4.6）；响应 SHOULD 在 `hint` 字段附 CA 注册 URL（NPS-RFC-0003）|
| `NWP-AUTH-REPUTATION-BLOCKED` | `NPS-AUTH-FORBIDDEN` | 接收 Node 的声誉策略命中了对发起方 `subject_nid` 的 `reject_on` 规则；响应 SHOULD 携带匹配的 `incident` + `severity` + 日志条目 `seq` 便于追溯（NPS-RFC-0004）|
| `NWP-QUERY-FILTER-INVALID` | `NPS-CLIENT-BAD-PARAM` | Filter 语法不合法或嵌套超过 8 层 |
| `NWP-QUERY-FIELD-UNKNOWN` | `NPS-CLIENT-BAD-PARAM` | fields 中引用了不存在的字段 |
| `NWP-QUERY-CURSOR-INVALID` | `NPS-CLIENT-BAD-PARAM` | cursor 值无法解码或已过期 |
| `NWP-QUERY-REGEX-UNSAFE` | `NPS-CLIENT-BAD-PARAM` | `$regex` 模式被拒绝（ReDoS 风险、超长或嵌套量词）|
| `NWP-QUERY-VECTOR-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | 节点不支持向量搜索 |
| `NWP-QUERY-AGGREGATE-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | 节点不支持聚合查询（capabilities.aggregate=false）|
| `NWP-QUERY-AGGREGATE-INVALID` | `NPS-CLIENT-BAD-PARAM` | aggregate 结构不合法（未知 func、alias 重复、缺少必填字段等）|
| `NWP-QUERY-STREAM-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | 节点不支持流式查询（capabilities.stream_query=false）|
| `NWP-ACTION-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | action_id 不存在于节点注册表 |
| `NWP-ACTION-PARAMS-INVALID` | `NPS-CLIENT-UNPROCESSABLE` | 操作参数 Schema 校验失败 |
| `NWP-ACTION-IDEMPOTENCY-CONFLICT` | `NPS-CLIENT-CONFLICT` | 相同 idempotency_key 的请求正在进行中 |
| `NWP-TASK-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | task_id 引用的异步任务不存在 |
| `NWP-TASK-ALREADY-CANCELLED` | `NPS-CLIENT-CONFLICT` | 任务已被取消，无法继续操作 |
| `NWP-TASK-ALREADY-COMPLETED` | `NPS-CLIENT-CONFLICT` | 任务已完成，无法取消 |
| `NWP-TASK-ALREADY-FAILED` | `NPS-CLIENT-CONFLICT` | 任务已失败，无法取消 |
| `NWP-SUBSCRIBE-STREAM-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | unsubscribe 引用的 stream_id 不存在 |
| `NWP-SUBSCRIBE-LIMIT-EXCEEDED` | `NPS-LIMIT-EXCEEDED` | 超出节点允许的最大并发订阅数 |
| `NWP-SUBSCRIBE-FILTER-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | 节点不支持带 filter 的订阅 |
| `NWP-SUBSCRIBE-INTERRUPTED` | `NPS-SERVER-UNAVAILABLE` | 订阅流因底层数据源中断而终止 |
| `NWP-SUBSCRIBE-SEQ-TOO-OLD` | `NPS-CLIENT-CONFLICT` | resume_from_seq 超出节点缓冲范围（推荐缓冲 10 分钟或 10,000 条）；Agent 须全量重查后重新订阅 |
| `NWP-BUDGET-EXCEEDED` | `NPS-LIMIT-BUDGET` | 响应将超过 X-NWP-Budget 限制 |
| `NWP-DEPTH-EXCEEDED` | `NPS-CLIENT-BAD-PARAM` | X-NWP-Depth 超过节点允许的 max_depth |
| `NWP-GRAPH-CYCLE` | `NPS-CLIENT-UNPROCESSABLE` | 节点图谱中存在循环引用 |
| `NWP-NODE-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | 底层数据源暂不可用 |
| `NWP-MANIFEST-VERSION-UNSUPPORTED` | `NPS-CLIENT-BAD-PARAM` | Agent NPS 版本低于节点要求的 min_agent_version |
| `NWP-MANIFEST-NODE-TYPE-REMOVED` | `NPS-CLIENT-BAD-FRAME` | NWM `node_type` 包含已移除的遗留值 `"gateway"`（NPS-CR-0001）；请改用 `"anchor"` 或 `"bridge"`。响应 SHOULD 携带指向 NPS-CR-0001 的 `hint` 字段。|
| `NWP-MANIFEST-NODE-TYPE-UNKNOWN` | `NPS-CLIENT-BAD-FRAME` | NWM `node_type` 包含无法识别的值（非已知遗留移除值——遗留移除情况请用 `NWP-MANIFEST-NODE-TYPE-REMOVED`）。|
| `NWP-RATE-LIMIT-EXCEEDED` | `NPS-LIMIT-RATE` | 超出频率限制，X-NWP-Rate-Reset 头包含重置时间 |
| `NWP-RESERVED-TYPE-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | `QueryFrame` 或 `SubscribeFrame` 的 `type` 字段包含不可识别的 reserved-type 标识符；当前节点未实现所请求的保留操作（NWP §12）。与 `NWP-ACTION-NOT-FOUND` 不同——当 `type` 字段是未知操作数（而非 `action_id`）时使用此码。|
| `NWP-TOPOLOGY-UNAUTHORIZED` | `NPS-AUTH-FORBIDDEN` | 调用方无权读取该 Anchor 的拓扑信息（NPS-2 §12）；授权策略由实现方按 §12.4 自定义（NPS-CR-0002）|
| `NWP-TOPOLOGY-UNSUPPORTED-SCOPE` | `NPS-CLIENT-BAD-PARAM` | `topology.scope` 的值该 Anchor Node 未实现（NPS-CR-0002）|
| `NWP-TOPOLOGY-DEPTH-UNSUPPORTED` | `NPS-CLIENT-BAD-PARAM` | 请求的 `topology.depth` 超出该 Anchor Node 配置的最大值（NPS-CR-0002）|
| `NWP-TOPOLOGY-FILTER-UNSUPPORTED` | `NPS-CLIENT-BAD-PARAM` | `topology.filter` 包含未识别的键或不支持的运算符（NPS-CR-0002）|

---

## NIP 错误码

| 错误码 | NPS 状态码 | 描述 |
|--------|-----------|------|
| `NIP-CERT-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` | 证书已过期（expires_at < now）|
| `NIP-CERT-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` | 证书已被吊销（在 CRL 或 OCSP 中）|
| `NIP-CERT-SIGNATURE-INVALID` | `NPS-AUTH-UNAUTHENTICATED` | 证书签名验证失败 |
| `NIP-CERT-UNTRUSTED-ISSUER` | `NPS-AUTH-UNAUTHENTICATED` | 颁发者��在 trusted_issuers 列表中 |
| `NIP-CERT-CAPABILITY-MISSING` | `NPS-AUTH-FORBIDDEN` | 证书缺少节���要求的能力 |
| `NIP-CERT-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` | 证书 scope 不覆盖目标路径或操作 |
| `NIP-CA-NID-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | NID 不存在于 CA 数据库 |
| `NIP-CA-NID-ALREADY-EXISTS` | `NPS-CLIENT-CONFLICT` | NID 已存在（重复注册）|
| `NIP-CA-SERIAL-DUPLICATE` | `NPS-CLIENT-CONFLICT` | 证书序列号已��在 |
| `NIP-CA-RENEWAL-TOO-EARLY` | `NPS-CLIENT-BAD-PARAM` | 距到期超过 7 天，尚未到续期窗口 |
| `NIP-CA-SCOPE-EXPANSION-DENIED` | `NPS-AUTH-FORBIDDEN` | 请求的 scope 超出父级 scope（委托链违规）|
| `NIP-OCSP-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | OCSP 服务暂时不可用 |
| `NIP-TRUST-FRAME-INVALID` | `NPS-CLIENT-BAD-FRAME` | TrustFrame 签名或格式不合法 |
| `NIP-ASSURANCE-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | `IdentFrame.assurance_level` 与证书扩展 `id-nid-assurance-level` 不一致（防 downgrade 攻击）—— 见 NPS-3 §5.1.1（NPS-RFC-0003）|
| `NIP-ASSURANCE-UNKNOWN` | `NPS-CLIENT-BAD-FRAME` | `assurance_level` 取值不在定义枚举（`anonymous` / `attested` / `verified`）—— 见 NPS-3 §5.1.1（NPS-RFC-0003）|
| `NIP-REPUTATION-ENTRY-INVALID` | `NPS-CLIENT-BAD-FRAME` | 声誉日志条目签名校验失败或规范化（RFC 8785 JCS）形式不合法 —— 见 NPS-3 §5.1.2（NPS-RFC-0004）|
| `NIP-REPUTATION-LOG-UNREACHABLE` | `NPS-DOWNSTREAM-UNAVAILABLE` | 准入评估时无法到达 Node `reputation_policy` 引用的某个日志运营方 —— 见 NPS-3 §5.1.2（NPS-RFC-0004）|
| `NIP-REPUTATION-GOSSIP-FORK` | `NPS-SERVER-INTERNAL` | 跨节点 STH 一致性检查失败；可能检测到 Merkle 树分叉 —— 见 NPS-RFC-0004 §4.5 |
| `NIP-REPUTATION-GOSSIP-SIG-INVALID` | `NPS-CLIENT-BAD-FRAME` | gossip 交换中对端 STH 签名验证失败 —— 见 NPS-RFC-0004 §4.5 |
| `NIP-CERT-FORMAT-INVALID` | `NPS-CLIENT-BAD-FRAME` | `IdentFrame.cert_chain` 不是 DER 编码 X.509，或 ASN.1 解析失败 —— 见 NPS-RFC-0002 §4.3 |
| `NIP-CERT-EKU-MISSING` | `NPS-CLIENT-BAD-FRAME` | leaf 证书缺少必需的 NPS EKU（`agent-identity` 或 `node-identity`）或未标 critical —— 见 NPS-RFC-0002 §4.1 / §4.3 |
| `NIP-CERT-SUBJECT-NID-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | X.509 leaf 证书 subject CN / SAN URI 与 `IdentFrame.nid` 字段不一致 —— 见 NPS-RFC-0002 §4.3 |
| `NIP-ACME-CHALLENGE-FAILED` | `NPS-CLIENT-BAD-FRAME` | ACME `agent-01` challenge 校验失败（token 不匹配、签名验证失败或 replay）—— 见 NPS-RFC-0002 §4.4 |

---

## NDP 错误码

| 错误码 | NPS 状态码 | 描述 |
|--------|-----------|------|
| `NDP-RESOLVE-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | nwp:// 地址无法解析到物理端点 |
| `NDP-RESOLVE-AMBIGUOUS` | `NPS-CLIENT-CONFLICT` | 解析结果存在冲突（多个不一致的注册）|
| `NDP-RESOLVE-TIMEOUT` | `NPS-SERVER-TIMEOUT` | 解析请求超时 |
| `NDP-ANNOUNCE-SIGNATURE-INVALID` | `NPS-AUTH-UNAUTHENTICATED` | AnnounceFrame 签名验证失�� |
| `NDP-ANNOUNCE-NID-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | AnnounceFrame 中 NID 与签名证书不一致 |
| `NDP-ANNOUNCE-ROLE-REMOVED` | `NPS-CLIENT-BAD-FRAME` | AnnounceFrame `node_roles` 包含已移除的遗留值 `"gateway"`（NPS-CR-0001）；请改用 `"anchor"` 或 `"bridge"`。响应 SHOULD 携带指向 NPS-CR-0001 的 `hint` 字段。|
| `NDP-ANNOUNCE-ROLE-UNKNOWN` | `NPS-CLIENT-BAD-FRAME` | AnnounceFrame `node_roles` 包含无法识别的值（非已知遗留移除值——遗留移除情况请用 `NDP-ANNOUNCE-ROLE-REMOVED`）。|
| `NDP-GRAPH-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | GraphFrame 序号不连续 |
| `NDP-REGISTRY-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | NDP Registry 暂��不可用 |

---

## NOP 错误码

| 错误码 | NPS 状态码 | 描述 |
|--------|-----------|------|
| `NOP-TASK-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | task_id 不存在 |
| `NOP-TASK-TIMEOUT` | `NPS-SERVER-TIMEOUT` | 整体任务执行超时 |
| `NOP-TASK-DAG-INVALID` | `NPS-CLIENT-BAD-FRAME` | DAG 格式不合法（缺少起点/终点节点、字段错误等）|
| `NOP-TASK-DAG-CYCLE` | `NPS-CLIENT-BAD-FRAME` | DAG 中存在环路 |
| `NOP-TASK-DAG-TOO-LARGE` | `NPS-CLIENT-BAD-FRAME` | DAG 节点数超过上限（默认 32）|
| `NOP-TASK-ALREADY-COMPLETED` | `NPS-CLIENT-CONFLICT` | 任务已完成，无法重复提交 |
| `NOP-TASK-CANCELLED` | `NPS-CLIENT-CONFLICT` | 任务已被取消 |
| `NOP-DELEGATE-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` | delegated_scope 超出父 Agent scope |
| `NOP-DELEGATE-REJECTED` | `NPS-CLIENT-UNPROCESSABLE` | 目标 Agent 拒绝接受委托（能力不足或超负载），响应含 retry_after_ms |
| `NOP-DELEGATE-CHAIN-TOO-DEEP` | `NPS-CLIENT-BAD-PARAM` | 委托链深度超过上限（默认 3 层）|
| `NOP-DELEGATE-TIMEOUT` | `NPS-SERVER-TIMEOUT` | 子任务在 deadline_at 前未完成 |
| `NOP-SYNC-TIMEOUT` | `NPS-SERVER-TIMEOUT` | SyncFrame 等待依赖任务完成超时 |
| `NOP-SYNC-DEPENDENCY-FAILED` | `NPS-CLIENT-UNPROCESSABLE` | 等待的依赖子任务已失败（且失败数超出 K-of-N 容忍范围）|
| `NOP-STREAM-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | AlignStream 序号不连续 |
| `NOP-STREAM-NID-MISMATCH` | `NPS-AUTH-UNAUTHENTICATED` | AlignStream sender_nid 与连接身份不一致 |
| `NOP-RESOURCE-INSUFFICIENT` | `NPS-SERVER-UNAVAILABLE` | 预检（preflight）发现一个或多个 Worker Agent 资源不足（CGN 不够或缺少能力）|
| `NOP-CONDITION-EVAL-ERROR` | `NPS-CLIENT-BAD-PARAM` | DAG 节点 condition 表达式求值失败（语法错误或引用了不存在的字段）|
| `NOP-INPUT-MAPPING-ERROR` | `NPS-CLIENT-UNPROCESSABLE` | input_mapping JSONPath 无法解析或目标字段不存在 |

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
