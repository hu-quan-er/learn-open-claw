# OpenClaw Gateway 协议与 API 面

## 1. 为什么要补这一篇

前面的文档已经把 `gateway` 定位为控制面核心，但还没有把它真正暴露给 UI、CLI、节点和自动化客户端的协议面展开。

这一篇补的是：
- Gateway WebSocket 协议的基本形状
- `operator` / `node` 两类连接角色
- RPC method family 的能力分布
- 事件流与订阅模型
- 认证、配对、scope、idempotency 对架构边界的意义

说明：
- 当前本地 `openclaw-repo` 没有完整源码检出
- 本篇基于官方 Gateway Architecture 与 Gateway Protocol 文档整理
- 涉及源码文件名时，只作为后续源码阅读指引

## 2. Gateway 协议的定位

官方 Gateway Protocol 文档把 Gateway WS 协议定义为 OpenClaw 的单一控制面与节点传输协议。

这句话很关键。它说明 Gateway 不是只给 Web UI 用的后端接口，而是同时服务：
- CLI
- Web UI / Control UI
- macOS app
- iOS / Android nodes
- headless nodes
- 自动化或运维客户端

因此 Gateway 的协议面必须同时解决两类问题：
- 控制面客户端如何读写系统状态
- 能力节点如何声明和执行本机能力

## 3. 基础帧模型

Gateway 使用 WebSocket，消息是 JSON text frame。

协议上有三类基本帧：

```text
Request
  { type: "req", id, method, params }

Response
  { type: "res", id, ok, payload | error }

Event
  { type: "event", event, payload, seq?, stateVersion? }
```

第一帧必须是 `connect` request。  
这表示 Gateway 把连接建立也纳入协议状态机，而不是“连接上就默认可信”。

有副作用的方法需要 idempotency key。  
这对 `send`、`agent` 这类操作尤其重要，因为客户端重连或超时重试时，不能让同一条消息或同一次 agent run 被重复执行。

## 4. Handshake 的含义

连接建立大致是：

```text
Gateway -> client
  connect.challenge

client -> Gateway
  req: connect
    minProtocol / maxProtocol
    client metadata
    role
    scopes
    caps / commands / permissions
    auth
    device identity + signature

Gateway -> client
  hello-ok
    protocol
    policy
    optional deviceToken
```

这里至少有四层边界：

1. 协议版本边界  
   客户端声明 `minProtocol` / `maxProtocol`，服务端不接受不兼容版本。

2. 身份边界  
   客户端需要提供 device identity，并签名 Gateway 发出的 challenge。

3. 角色边界  
   客户端连接时声明自己是 `operator` 还是 `node`。

4. 能力边界  
   operator 走 scopes，node 走 caps / commands / permissions。

这比普通 WebSocket chat API 重很多，但符合 OpenClaw 的平台定位。

## 5. `operator` 与 `node`

Gateway 协议显式区分两类角色：

```text
operator
  控制面客户端，例如 CLI / UI / automation

node
  能力宿主，例如 iOS / Android / headless node
```

`operator` 关注的是“我能调用哪些控制面方法”。  
典型 scope 包括：
- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`node` 关注的是“我能提供哪些设备能力”。  
它会声明：
- `caps`：能力大类，例如 camera、canvas、screen、location、voice
- `commands`：可被 invoke 的具体命令
- `permissions`：更细颗粒度的开关

分析判断：
- OpenClaw 把“控制系统的人”和“提供设备能力的端”从协议层就分开了
- 这能减少远端节点被误当作管理客户端的风险
- UI 展示 presence 时，也可以按 device identity 聚合不同角色连接

## 6. Scope 不是装饰字段

Gateway Protocol 文档明确提到，某些核心管理 namespace 会强制要求 admin 级别：

```text
config.*
exec.approvals.*
wizard.*
update.*
```

即使插件注册 Gateway RPC 方法时请求了更窄的 scope，保留的核心 admin namespace 仍然解析为 `operator.admin`。

这说明 OpenClaw 对协议扩展有一个重要原则：

```text
插件可以扩展 API 面，
但不能重新定义核心管理 API 的权限等级。
```

这点对长期插件生态很重要。否则插件或扩展一旦能“降级”核心管理方法，整个控制面权限模型就会被绕开。

## 7. RPC method family 地图

官方协议文档列出的 Gateway method family 很多。按平台层次可以整理成：

```text
系统与身份
  health / status / gateway.identity.get / system-presence / heartbeat

模型与用量
  models.list / usage.status / usage.cost / sessions.usage

渠道与登录
  channels.status / channels.logout / web.login.start / web.login.wait

消息与日志
  send / logs.tail

配置、密钥、更新、向导
  secrets.* / config.* / update.run / wizard.*

agent 与 workspace
  agents.* / agents.files.* / agent.identity.get / agent.wait

session 控制
  sessions.* / chat.history / chat.send / chat.abort / chat.inject

设备配对与 token
  device.pair.* / device.token.*

node 配对与调用
  node.pair.* / node.invoke / node.pending.*

approval
  exec.approval.* / plugin.approval.*

自动化、技能、工具
  wake / cron.* / commands.list / skills.* / tools.*
```

这个分布可以反向证明一件事：

```text
Gateway 是 OpenClaw 的真实平台 API 面。
```

它不是单纯转发 agent 消息，而是把配置、模型、渠道、会话、设备、审批、自动化都纳入统一 RPC 边界。

## 8. Event family 地图

常见事件 family 包括：
- `chat`
- `session.message`
- `session.tool`
- `sessions.changed`
- `presence`
- `tick`
- `health`
- `heartbeat`
- `cron`
- `shutdown`
- `node.pair.requested`
- `node.pair.resolved`
- `device.pair.requested`
- `device.pair.resolved`
- `exec.approval.requested`
- `exec.approval.resolved`
- `plugin.approval.requested`
- `plugin.approval.resolved`

这说明 UI 或客户端不应该只靠轮询 `status`。  
更自然的模型是：

```text
首次拉取 snapshot
  -> subscribe / receive events
  -> 根据 stateVersion / seq 判断是否需要 refresh
```

官方 Gateway Architecture 文档也提到 events 不 replay，客户端遇到 gap 要自行 refresh。

## 9. Agent 与 Session API 的位置

从 Gateway Protocol 的 method family 看，agent 和 session 不是同一层东西：

```text
agent
  agent.wait 等运行级能力

sessions
  list / subscribe / messages.subscribe / preview / resolve / create /
  send / steer / abort / patch / reset / delete / compact / get

chat
  history / send / abort / inject
```

这和前面 Agent Loop 文档的判断一致：
- `agent` 更偏运行请求
- `sessions` 更偏持久会话状态
- `chat` 更偏 UI/display-normalized 交互面

读源码时不要把这三者混成一个“聊天 API”。

## 10. 协议 typing 与 codegen

官方文档指出：
- `PROTOCOL_VERSION` 在 `src/gateway/protocol/schema.ts`
- 协议 schema 来自 TypeBox
- JSON Schema 和 Swift models 可由脚本生成

这解释了为什么 [02-启动流程与控制面.md](./02-启动流程与控制面.md) 里提到 TypeBox 很重要。

TypeBox 在这里不是普通校验库，而是承担：
- 运行时入参校验
- 协议文档化
- 客户端模型生成
- UI / macOS / mobile node 的跨语言契约

## 11. 安全边界

Gateway 协议至少有这些安全边界：

1. Shared-secret auth  
   `connect.params.auth.token` 或 password。

2. Device token  
   配对成功后 Gateway 可签发 device token，后续重连复用。

3. Device identity + challenge signature  
   防止简单复制连接参数就冒充设备。

4. Pairing approval  
   新设备 ID 需要配对审批，本地 loopback 可有更顺滑的自动批准路径。

5. Scope enforcement  
   方法 scope 是第一层门禁，某些 slash command 还有更严格的命令级检查。

6. Node command allowlist  
   node 声明能力，但 Gateway 仍做服务端 allowlist enforcement。

这些机制说明 OpenClaw 没有把 Gateway 当成可信内网里的裸 API。  
它从协议层就假设有远端、节点、代理、隧道和多设备接入。

## 12. 对源码阅读的建议

后续如果拿到完整源码，建议优先读：

1. `src/gateway/protocol/schema.ts`
2. `src/gateway/server-methods-list.ts`
3. `src/gateway/server-methods/*`
4. `src/gateway/server/*`
5. `src/gateway/pair*` 或 device pairing 相关路径
6. UI 中 Gateway client / WS client 封装

重点问题：
- 哪些 method 被标记为 side-effecting，如何要求 idempotency key
- scope 到 server method 的映射表在哪里
- plugin 注册的 RPC method 如何进入 `hello-ok.features.methods`
- event 的 `seq` / `stateVersion` 在哪些状态更新里递增
- `chat.history` 的 display normalization 和 raw session transcript 如何分开

## 13. 结论

OpenClaw Gateway 的 API 面不是 REST 风格资源集合，而是一个带角色、scope、设备身份、事件流和 codegen 契约的控制面协议。

可以把它记成：

```text
Gateway Protocol =
  WebSocket RPC
  + typed schema
  + operator/node role split
  + device pairing
  + session/agent/control APIs
  + server-push event stream
```

这层如果读通，OpenClaw 的 UI、CLI、移动节点、配对、审批和 session 控制都会更容易串起来。

## 14. 参考

- Gateway Architecture：<https://docs.openclaw.ai/concepts/architecture>
- Gateway Protocol：<https://docs.openclaw.ai/gateway/protocol>

