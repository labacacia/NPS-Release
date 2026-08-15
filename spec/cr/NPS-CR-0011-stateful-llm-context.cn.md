[English](./NPS-CR-0011-stateful-llm-context.md) | 中文

# NPS-CR-0011：有状态 LLM Context 与增量 Completion

**状态**：Implemented
**目标版本**：v1.0.0-alpha.18  
**日期**：2026-08-12  
**作者**：Ori Lynn / INNO LOTUS PTY LTD  
**跟踪 Issue**：[NPS-Dev#90](https://github.com/labacacia/NPS-Dev/issues/90)  
**涉及范围**：NPS-2 NWP（§4.2a LLM profile、§7.5 `llm.complete`、§7.6 stateful context）、NPS-3 NIP（§5.1 capabilities）、统一错误/状态码、NWM、六语言 SDK、NWP 合规向量

---

## 1. 摘要

在现有 `llm.complete` action 上增加可选、与 provider 无关的服务端 context
contract。客户端先创建一个不透明 context，后续只追加新的消息或工具结果，
服务端复用已经提交的模型前缀。该变更是纯增量的：不含 `context` 的普通请求
继续沿用现有 stateless 完整消息列表语义。

本 CR 不新增 NCP frame type，而是新增：

1. `LlmCompleteActionRequest.context`，支持 `create`、`append`、`fork`、`reset`；
2. unary、async 最终结果和 streaming 终块中的 context receipt；
3. `llm.context.status` 与 `llm.context.release` 生命周期 action；
4. CAS 版本、身份所有权、binding 校验、原子提交和确定性失败语义；
5. NWM 发现元数据与 NIP `llm:context` capability；
6. 区分逻辑输入、复用输入、新求值输入与序列化请求字节的 usage 字段。

最重要的安全规则是：**禁止静默降级**。请求一旦携带 context operation，节点
必须精确执行，或者返回 ErrorFrame；不得偷偷重建完整 prompt 后仍把请求报告为
成功的 stateful/cache hit。

## 2. 动机

持久 NCP 连接可以消除重复握手，MessagePack 可以减少传输字节，但两者都不能
阻止客户端重发完整对话，也不能保证模型 runtime 不重复求值。因此，仅靠传输
压缩不能真实地声称节省了模型输入 token。

各 provider 的私有 session 也不能形成互操作 contract：其所有权、陈旧更新、
工具 binding、重连行为和 token 统计都不同。没有统一 NWP contract，客户端只能
继续保留 provider-specific codec，或者从成功响应中猜测状态，最终会重新产生
官方 `llm.complete` DTO 已经消除过的漂移。

## 3. Wire Contract

### 3.1 沿用现有 action，增量添加字段

`ActionFrame.action_id` 仍为 `llm.complete`，`ActionFrame.params` 仍为
`LlmCompleteActionRequest`，只新增一个可选字段：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `context` | `LlmContextRequestDto` | 否 | 有状态 context operation；缺省时为现有 stateless contract。 |

出现 `context` 时，必须同时提供 `ActionFrame.idempotency_key`。沿用普通
ActionFrame 的 24 小时 replay window，且 key 绑定完整 canonical request；同一
key 携带不同 params 时返回 `NWP-ACTION-IDEMPOTENCY-CONFLICT`。

原 stateful 请求仍在运行时，重复请求返回该 conflict，且不得加入现有 live stream。
完成后，unary/async replay 沿用现有 cached-result 规则。Streaming replay 必须使用
新的 StreamFrame sequence 返回 cached logical chunks 与同一 terminal context
receipt，不得再次运行模型或 mutation。服务端可以合并 replay chunk，但有序文本、
tool calls、stop reason、usage 与 receipt 必须在逻辑上完全一致。

`messages` 仍然必填，但含义按 operation 区分：

- stateless：完整有序对话；
- `create` / `reset`：完整初始或替换对话；
- `append` / `fork`：`base_version` 之后的有序增量。

`fork` 可以使用空 `messages` 克隆已提交前缀；`create`、`append`、`reset`
不得使用空数组。

### 3.2 `LlmContextRequestDto`

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `operation` | enum | 是 | `create`、`append`、`fork` 或 `reset`。 |
| `context_id` | string | 条件必填 | `append`、`fork`、`reset` 必填；`create` 禁止。大小写敏感的不透明值。 |
| `base_version` | uint64 | 条件必填 | `append`、`fork`、`reset` 必填，必须等于当前已提交版本。 |
| `ttl_seconds` | uint32 | 否 | 请求的空闲生命周期；服务端可以按 NWM 上限收窄，receipt 报告实际到期时间。 |

服务端以至少 128 bit 密码学安全随机数生成无 padding base64url `context_id`；
生产者发出 22–128 个 `[A-Za-z0-9_-]` ASCII 字符。其中不得编码 NID、租户、
模型、数据库 key 或其他安全敏感信息。Context ID 只是定位符，绝不是授权凭据。

### 3.3 Context receipt

`LlmCompleteActionResponse` 与终结
`LlmCompleteStreamChunkDto` 新增可选 `context` 字段，类型为
`LlmContextReceiptDto`：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `context_id` | string | 是 | 服务端生成的不透明标识。 |
| `version` | uint64 | 是 | 本次 completion 后的已提交版本；从 1 开始，每次成功 mutation 恰好加 1。 |
| `operation` | enum | 是 | 本结果提交的 operation：`create`、`append`、`fork`、`reset` 或 `release`；completion 请求只接受前四种。 |
| `state` | enum | 是 | completion mutation 成功时为 `active`；release 成功时为 `released`。 |
| `expires_at` | RFC 3339 timestamp | 否 | 有期限时的实际到期时间。 |
| `parent_context_id` | string | 条件必填 | `fork` 时出现，标识源 context。 |
| `parent_version` | uint64 | 条件必填 | `fork` 时出现，标识不可变的源快照。 |

Stateless 响应必须省略 `context`，非终结 stream chunk 也必须省略。Async ack
不代表 context 已发生 mutation；只有任务完成后的 `system.task.status.result`
携带 receipt。

### 3.4 生命周期 action

释放 context 不应伪装成模型 completion，因此使用独立 action：

| Action | 所需 capability | 请求 | 成功结果 |
|---|---|---|---|
| `llm.context.status` | `llm:context` | `context_id` / `idempotency_key` 二选一 | `LlmContextStatusDto` |
| `llm.context.release` | `llm:context` | `context_id`、`base_version` | `state = "released"` 的 `LlmContextReceiptDto` |

`status` 可按 `idempotency_key` 查询，以便客户端丢失 `create` 终结响应后在
ActionFrame replay window 内找回服务端生成的 context ID。Status 只观察状态，
不延长 TTL。Release 也必须提供 ActionFrame `idempotency_key`；它对相同
owner/key 幂等，replay 返回原 release receipt。

`LlmContextStatusDto` 包含 `state`（`busy`、`active`、`released`、`expired`、
`failed`）、可选 `context_id`、可选 `version`、可选 `expires_at`、可选活跃
`request_id` 与可选 `error_code`。Active/tombstone state 必须携带 context ID 与
version；运行中的 create 报告 `busy`，但 commit 前必须省略二者；失败的 create
报告 `failed`、省略二者并携带终结 error code。released/expired tombstone 建议
至少保留 NWM 声明的 `tombstone_seconds`；超过该期限后同一查询返回 not-found。

## 4. 状态与提交语义

### 4.1 状态机

```text
                  create success
  ABSENT --------------------------------> ACTIVE(v1)
                                             |
                 append/reset success        | fork success
                 ACTIVE(vN+1) <---------------+------------> ACTIVE-child(v1)
                     |                                        (parent unchanged)
                     | release / idle expiry
                     v
              RELEASED / EXPIRED tombstone ----> ABSENT
```

`append`、`reset`、`release` 都是 CAS operation；`base_version` 必须等于当前
已提交版本。节点必须按 context 串行化 mutation，同一时刻至多一个请求持有
mutation reservation。陈旧请求或并发失败方收到
`NWP-LLM-CONTEXT-VERSION-CONFLICT`，ErrorFrame hint 携带当前版本。

`fork` 从 `base_version` 的不可变快照创建新 context，父 context 不变。
`reset` 保留 context ID，但原子替换 transcript 与 binding，并递增版本。

Fork 在 admission 时原子获取 parent 快照；之后 parent append 不会使已接纳的快照
失效。Release 把 tombstone version 从 vN 增至 vN+1；自动 expiry 记录最后已提交
版本，但不递增。

成功的 create、append、fork、reset 会开始或刷新实际 idle TTL；status 不刷新。
失败或取消的 mutation 也不刷新。有效 mutation reservation 执行期间 context
不 expiry；如果旧 TTL 已经过期，reservation abort 并释放后立即进入 expired。

`ttl_seconds` 出现时必须大于零。省略时，`create` 使用节点默认值，`append` 与
`reset` 保留 context 当前的实际 TTL，`fork` 继承源 context 的剩余 TTL。节点可将
显式值、继承值或默认值收窄到广告上限，并在 TTL 有界时通过 receipt 返回实际
expiry。

### 4.2 原子 completion 边界

只有模型 action 以非 `error` stop reason 成功终结时 mutation 才提交。提交状态
同时包含请求增量和最终 assistant 消息或 tool calls。以普通 `end_turn` 表示的
结构化 refusal 可以提交；`stop_reason = "error"` 不提交。参数校验失败、授权失败、provider error、
timeout、cancel 或 stream 异常终止都必须：

- 保持原已提交版本和 transcript 不变；
- 释放 mutation reservation；
- 返回普通 ErrorFrame 或 terminal stream error。

Streaming receipt 只出现在成功终块。服务端必须先按其广告的 persistence level
原子提交，再让该终结 receipt
可见。断线可能使“客户端是否收到响应”不确定，但状态本身不能不确定：客户端用
`llm.context.status` 判定；append 使用已知 `context_id`，丢失 create 响应时使用
原 `idempotency_key`。

### 4.3 重启与持久级别

NWM 声明一种 persistence level：

- `connection`：只在当前 NCP 连接内跨请求存活；
- `process`：重连仍路由到同一进程时存活；进程重启或路由到其他实例后丢失；
- `durable`：在同一 logical node NID 与 endpoint identity 下跨进程重启存活。

状态丢失后必须返回 `NWP-LLM-CONTEXT-NOT-FOUND` 或
`NWP-LLM-CONTEXT-EXPIRED`，不得静默新建替代 context。声明 `process` 或
`durable` 的节点必须使用稳定 endpoint identity。connection/process context ID
只在 node instance 范围内有效，客户端应固定到创建它的 endpoint。跨实例共享
context store 与 context migration 不在本 CR 内；除非所有 target 都属于同一
durable logical node，否则 load balancer 必须提供 affinity。

## 5. Binding 与授权

### 5.1 不可变 binding

create/reset 时，服务端把 context 绑定到：

- alias/routing 解析后的模型 ID；
- 有序 system message 集合；
- canonical tool definitions（含参数 schema）；
- 复用前缀所需的 provider/runtime compatibility revision。

`append` 和 `fork` 必须保持 binding 不变。模型、system message、tool definition
发生变化时，必须返回 `NWP-LLM-CONTEXT-BINDING-MISMATCH`。客户端要有意修改
这些输入时，应使用 `reset`（保留 lineage）或 stateless/create（新 lineage）。
Runtime 私有 cache handle 与 binding fingerprint 不上 wire。append/fork 省略
`tools` 表示复用已绑定定义；若出现则必须 canonical 相等。append/fork delta
禁止 system-role message；必填 `model` 必须解析为已绑定模型。

### 5.2 NIP 所有权

有状态 `llm.complete` mutation 同时要求 `llm:complete` 与 `llm:context`；
streaming 仍额外要求 `llm:stream`，tools 仍额外要求 `llm:tool_call`。
Status/release 要求 `llm:context` 加 owner 授权，但不要求 `llm:complete`，因此
principal 失去模型调用权后仍能检查和清理其保留状态。Coordinator 必须在 admission
与 commit 时把完整的必需 capability 集合交给部署侧 authorizer；若未配置
authorizer，有状态 surface 必须以 `NWP-LLM-CONTEXT-FORBIDDEN` fail closed，
不得将缺少 hook 视为 allow。Context owner 由认证 caller NID
与节点认证后的 tenant/workspace security scope 共同构成。Scope 来自已接纳身份和
部署 policy，不来自客户端可控的 context 字段。

每次 lookup/mutation 都必须重新执行 NIP expiry、revocation、assurance、scope 和
capability 检查。不同 owner 返回 `NWP-LLM-CONTEXT-FORBIDDEN`。实现应尽量采用
不透明 ID 查询与固定顺序授权；ID 本身必须不可猜，但不能把“知道 ID”当作授权。
长时间生成还必须在 commit 前再次检查 revocation 与 authorization。Caller 在
admission 后被撤销或失去授权时，返回普通 NIP/NWP authorization ErrorFrame，
并 abort reservation。

## 6. 发现

`profiles.llm.profile_version` 升至 `0.2`。支持本 CR 的节点必须在顶层 NWM
`actions` registry 注册 `llm.context.status` 与 `llm.context.release`，并令
`required_capability = "llm:context"`；同时新增：

```json
{
  "profiles": {
    "llm": {
      "profile_version": "0.2",
      "actions": [
        "llm.complete",
        "llm.context.status",
        "llm.context.release"
      ],
      "context": {
        "supported": true,
        "operations": ["create", "append", "fork", "reset", "release"],
        "persistence": "durable",
        "max_contexts_per_principal": 32,
        "max_ttl_seconds": 3600,
        "tombstone_seconds": 86400
      }
    }
  }
}
```

客户端必须先发现 `context.supported = true`、所需 operation 与 `llm:context`
capability，再发送 stateful 请求。已经声明但运行时无法提供时，返回
`NWP-CAPABILITY-ADVERTISED-UNIMPLEMENTED`；`operations` 未声明的 operation
返回 `NWP-LLM-CONTEXT-OPERATION-UNSUPPORTED`。

## 7. Usage 统计

`LlmUsageDto` 新增：

| 字段 | 类型 | 说明 |
|---|---|---|
| `wire_input_bytes` | uint64 | NWP decoder 实际接收的完整序列化 ActionFrame payload 字节数；采用协商 JSON/MessagePack 表示，位于 NCP 解密之后且不计 frame header。 |

现有字段语义保持：

- `input_tokens`：已提交前缀与本次增量共同表示的逻辑模型输入；
- `reused_tokens`：本次不经求值、直接来自保留 prefix/KV state 的逻辑输入 token；
- `evaluated_tokens`：本次实际新求值的输入 token；
- `output_tokens`：本次新生成 token。

三个输入计数均可得时，必须满足
`reused_tokens + evaluated_tokens = input_tokens`。只有 `reused_tokens > 0`
时才可令 `cache_hit = true`。`wire_input_bytes` 必须在 NWP decoder 边界实测，
不得从 DTO 重新序列化；它包含 ActionFrame envelope（包括 context 与 idempotency
metadata），排除 NCP header、TLS record 和响应字节，使 JSON 与 MessagePack
可比较。Provider 无法观测时不得按 prompt 长度编造计数，应省略对应字段。

## 8. 错误

下列失败全部走 ErrorFrame（或 terminal stream error），不得放进
`LlmCompleteActionResponse.error`：

| 错误 | NPS status | 含义 |
|---|---|---|
| `NWP-LLM-CONTEXT-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | context 不存在，或 idempotency lookup 已超过保留期。 |
| `NWP-LLM-CONTEXT-EXPIRED` | `NPS-CLIENT-GONE` | 存在 tombstone，证明 context 因 idle expiry 到期。 |
| `NWP-LLM-CONTEXT-VERSION-CONFLICT` | `NPS-CLIENT-CONFLICT` | `base_version` 陈旧，或另一 mutation 持有 reservation。 |
| `NWP-LLM-CONTEXT-BINDING-MISMATCH` | `NPS-CLIENT-CONFLICT` | 模型、system prompt、tools 或 runtime compatibility revision 不一致。 |
| `NWP-LLM-CONTEXT-FORBIDDEN` | `NPS-AUTH-FORBIDDEN` | caller 不是 owner，或缺少 scope/capability。 |
| `NWP-LLM-CONTEXT-LIMIT-EXCEEDED` | `NPS-LIMIT-RESOURCE` | 达到每 principal 活跃 context 上限；hint 应报告上限。 |
| `NWP-LLM-CONTEXT-OPERATION-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | 节点支持 context，但不支持请求的 operation。 |

字段组合不合法仍使用 `NWP-ACTION-PARAMS-INVALID`；provider/runtime 失败继续使用
现有 NWP server error。Owner 可通过 `llm.context.status` 观察 released/expired
tombstone。对 released context 发起 completion mutation 返回 not-found；对 expiry
tombstone 发起 mutation 返回 gone；tombstone 清除后两者都返回 not-found。原
release key 的 replay 仍遵循 ActionFrame replay 规则。

本 CR 新增通用 LIMIT status `NPS-LIMIT-RESOURCE`（HTTP 429），用于 context 等
有上限的活跃资源。统一状态表已包含该映射；活跃对象数量上限不是请求速率违规。

## 9. SDK 改动

六语言 SDK 提供完全一致的 wire DTO 与 enum 值：

- `LlmContextRequestDto`、`LlmContextReceiptDto`、`LlmContextStatusDto`；
- context operation/state enum；
- 两个生命周期 action 的 typed builder/parser；
- unary response 与 terminal stream chunk 的 context 字段；
- `LlmUsageDto.wire_input_bytes`；
- NWM `context` 发现元数据。

Server library 还需提供具备 atomic reserve/commit/abort、owner 校验、expiry 和
status lookup 的 context-store interface。内存 store 足以声明 `connection` /
`process`；只有通过 restart 测试的 backend 才能声明
`durable`。SDK 仅能编码 DTO，并不代表可以广告 context support。

## 10. 合规计划

候选规范变更随 NWP 0.21 新增
`spec/conformance/nwp/llm_context_vectors.json`。下方保留稳定 case ID；六语言
SDK 全部执行该文件后，本 CR 才能从 Draft 进入 Implemented。

| ID | 必须判定 |
|---|---|
| `nwp.llm-context.001` | Stateless 路径保持 wire/行为兼容，不含 receipt。 |
| `nwp.llm-context.002` | Create 只在成功终结时提交 v1 并返回不透明 ID。 |
| `nwp.llm-context.003` | Append 只发送 delta，提交 vN+1。 |
| `nwp.llm-context.004` | 陈旧/并发 append 返回 version conflict，状态不变。 |
| `nwp.llm-context.005` | Fork 创建 child v1，parent 不变。 |
| `nwp.llm-context.006` | Reset 原子替换 binding/transcript。 |
| `nwp.llm-context.007` | 模型/工具/system 不一致时 fail closed，禁止 stateless fallback。 |
| `nwp.llm-context.008` | owner/capability 不符时 forbidden。 |
| `nwp.llm-context.009` | cancel/timeout/stream error 中止 reservation 与版本变更。 |
| `nwp.llm-context.010` | 丢失 create 响应后按 idempotency key 找回状态。 |
| `nwp.llm-context.011` | Release/expiry/tombstone 错误迁移确定。 |
| `nwp.llm-context.012` | Usage 等式与实测 wire bytes 自洽。 |
| `nwp.llm-context.013` | NWM 只声明真实实现的 operation/persistence。 |
| `nwp.llm-context.014` | 重启行为与 `connection`/`process`/`durable` 一致。 |
| `nwp.llm-context.015` | 幂等 streaming replay 不重新生成或再次 commit。 |
| `nwp.llm-context.016` | commit 前发生 revocation 时 abort reservation 与 mutation。 |
| `nwp.llm-context.017` | 每 principal context 上限使用 `NPS-LIMIT-RESOURCE` 且不分配状态。 |
| `nwp.llm-context.018` | 未广告 operation 返回 operation-unsupported。 |
| `nwp.llm-context.019` | Stateful 请求缺少 idempotency key 时在 dispatch 前 params-invalid。 |

端到端 benchmark 必须用同一段多轮对话比较 stateless 与 stateful。第二轮
stateful 必须只发送 delta，`wire_input_bytes` 更少，`evaluated_tokens` 更少，且
保持 role/tool 的有序语义。Benchmark 必须使用禁用协议 fallback 的 strict native
模式。

## 11. 迁移与兼容性

- 现有 client、server、manifest 与 `llm.complete` payload 继续有效。
- Client 只有显式发送 `context` 才 opt in；不得从持久连接或 `cache_hit` 推断。
- Context-capable client 只能在提交 context operation 之前按本地显式 policy
  降级为 stateless；server 不得代替 client 执行降级。
- Context ID 在 NWP 边界与 provider 无关，但不能跨无关 logical node identity
  移植。

## 12. 范围之外

- 标准化模型内部 KV-cache 布局，或在 provider 间搬运 cache tensor。
- 定义共享长期记忆、检索或 NDP node discovery 语义。
- 暴露隐藏 chain-of-thought 或 provider 私有 reasoning state。
- 保证 stateless/stateful 两次生成逐 bit 相同。
- 跨 owner 共享或委托 context。未来可用显式 grant object 扩展；本 CR 禁止
  bearer-style context sharing。
- 跨实例 context migration 或分布式 context-store 协议。
- 在 alpha.18 发布 NPS LLM Framework 或 Willow。

## 13. 接受标准

- [x] 独立 design review 接受 operation 名、提交边界、所有权、持久级别与错误。
- [x] NWP/NIP/error/status 英中正文落地。
- [x] 六语言 SDK 提供一致 DTO 与 NWM metadata。
- [x] 六语言 CI 均执行共享 context vectors。
- [x] 至少一个 native server 通过 cancel、reconnect、restart、并发更新集成测试。
- [x] Ivy strict-native 集成删除私有 stateful payload codec。
- [x] Stateless 路径无行为回归。
- [x] Benchmark 同时证明 transport byte 与 evaluated-token 节省。
- [x] 分发前 source-of-truth 与 version-consistency gate 全绿。

实现验证于 2026-08-14 完成。六语言 SDK suite 均执行 19 条共享向量与 Action
Server 集成场景。Strict-native 第二轮 benchmark 得到 `wire_input_bytes`
756 -> 586（降低 22.5%），确定性 runtime `evaluated_tokens` 157 -> 59
（降低 62.4%），同时保持 role/tool 有序语义一致并禁用协议 fallback。Ivy 已基于
alpha.18 源码项目验证官方 DTO usage、并发乱序 unary correlation，以及缺失/未知
响应 ID 的严格拒绝。同步到 NPS-Release 与 standalone SDK repo 属于 release workflow，
不纳入本 CR 的实现 commit。

## 14. 建议 CHANGELOG 文案

> **NWP 有状态 LLM context（NPS-CR-0011，Implemented）**：为 `llm.complete` 增加
> opt-in context/delta contract、owner-bound 不透明 context ID、CAS 版本、
> create/append/fork/reset/status/release 生命周期、原子 stream completion、
> NWM/NIP 发现与授权、确定性错误，以及 logical/reused/evaluated/wire 实测 usage。
> Stateless completion 不变；stateful 请求绝不静默降级。

## 15. 已确定的设计选择

丢失响应恢复沿用现有强制 24 小时 ActionFrame idempotency window。本 CR 不新增
第二个 retention 配置：即使 context TTL 更短，节点也必须在完整 24 小时内保留
owner-scoped key-to-outcome 记录。该记录可以指向 expired/released tombstone；
24 小时后按 key 查询可以返回 not-found。
