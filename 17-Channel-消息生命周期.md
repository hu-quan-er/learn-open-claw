# OpenClaw Channel 消息生命周期

## 1. 为什么要单独看消息生命周期

前面的文档已经讲了 channel plugins、Gateway、agent loop 和 multi-agent routing。  
但真实消息从外部平台进入 OpenClaw，再到回复发出，中间还有一条完整 delivery pipeline。

这条链路包含：
- 入站去重
- 入站 debouncing
- channel/account/peer/thread 解析
- mention gating
- routing/bindings
- session key 选择
- queue/followup/steer
- streaming/chunking
- reply threading
- silent reply
- delivery retry

如果只看 agent loop，会漏掉大量渠道侧语义。

## 2. 高层消息流

官方 Messages 文档给出的高层流是：

```text
Inbound message
  -> routing/bindings
  -> session key
  -> queue if a run is active
  -> agent run with streaming + tools
  -> outbound replies with channel limits + chunking
```

这说明 channel 消息不是直接送进模型。  
它先被归一化成 OpenClaw 的路由和 session 语义，然后才进入 agent loop。

## 3. Channel plugin 的职责

官方 Channel Plugins 文档说明，一个 channel plugin 主要负责：

- Config
- Security
- DM policy and allowlists
- Pairing
- Session grammar
- Outbound
- Threading
- Heartbeat typing

它不需要为普通聊天动作注册独立 send/edit/react tools。  
OpenClaw core 保留一个共享 `message` tool，channel plugin 负责提供 channel-specific action discovery 和最终发送实现。

这是一条非常关键的边界：

```text
core owns shared message tool
channel plugin owns platform semantics
```

## 4. 入站消息接入

channel plugin 可以通过 webhook、长连接、平台 SDK 或其他方式接收外部消息。  
典型路径：

```text
platform event
  -> plugin verifies request
  -> parse provider-specific payload
  -> normalize sender/account/peer/thread/message id
  -> dispatch inbound message to OpenClaw
```

插件要在这一层处理：
- 平台签名/secret 校验
- service/system message 排除
- raw message text
- attachment/media metadata
- thread/reply context
- platform message id

这一步如果做错，后面 session routing 和 dedupe 都会不稳。

## 5. Inbound dedupe

官方 Messages 文档说明，channel 可能在重连后重复投递同一条消息。  
OpenClaw 会用短期 cache 按 channel/account/peer/session/message id 去重，避免重复触发 agent run。

这说明入站消息必须有足够稳定的 identity：

```text
channel id
account id
peer/conversation id
session/thread id
message id
```

如果某个 channel 没有稳定 message id，插件需要构造合理的幂等键，否则重复投递会变成重复回复。

## 6. Inbound debouncing

官方 Messages 文档说明，快速连续消息可以通过 `messages.inbound` 合并成一个 agent turn。  
debouncing 按 channel + conversation 维度隔离，并使用最近一条消息作为 reply threading/id。

这解决的是移动聊天常见场景：

```text
用户连续发三条短消息
  -> OpenClaw 等待 debounce 窗口
  -> 合并成一轮 agent 输入
```

但这也有语义代价：
- 最后一条消息决定 reply thread
- 合并后的 prompt 需要保留顺序
- 等待窗口会增加响应延迟
- group chat 里 debounce 范围必须非常清楚

## 7. Mention gating

官方 Channel Plugins 文档建议把 inbound mention handling 拆成两层：

```text
plugin-owned evidence gathering
  平台本地事实收集

shared policy evaluation
  OpenClaw 统一判断是否应该响应
```

插件适合判断：
- 是否 reply to bot
- 是否 quoted bot
- bot 是否参与过 thread
- 是否 service/system message
- 平台原生 mention 结构

共享 helper 适合判断：
- requireMention
- explicit mention
- implicit mention allowlist
- command bypass
- final skip decision

这能避免每个 channel 自己写一套不一致的 mention 规则。

## 8. Routing 与 bindings

入站消息通过 routing/bindings 找到 agent。  
这和 [12-Multi-Agent-路由与隔离.md](./12-Multi-Agent-路由与隔离.md) 的链路一致：

```text
inbound message
  -> channel/account/peer/thread
  -> binding match
  -> agentId
  -> sessionKey
```

这一步决定：
- 消息属于哪个 agent
- 用哪个 workspace/auth/tools/model
- 落到哪个 session key
- 是否进入 main session 或 thread session

如果 binding 过宽，消息可能落到错误 agent。  
如果 session grammar 不稳定，同一个外部 thread 可能分裂成多个 session。

## 9. Session grammar

官方 channel plugin 文档提到，插件要负责 provider-specific conversation ids 如何映射到 base chats、thread ids 和 parent fallbacks。

不同平台差异很大：
- Slack 有 team/channel/thread
- Discord 有 guild/channel/thread
- Telegram 有 chat/message/reply
- WhatsApp 有个人/群聊/业务账号
- Matrix 有 room/event

OpenClaw core 不应该硬编码所有平台的 thread 规则。  
插件负责把平台语义折成统一可路由的 session grammar。

## 10. Queueing 与 followups

官方 Messages 文档说明，如果 run 已经 active，入站消息可以：

```text
interrupt
steer
followup
collect
backlog variants
```

配置通过 `messages.queue` 和 `messages.queue.byChannel` 控制。

这说明“用户在 agent 正在跑时又发消息”不是单一行为。  
不同 channel 和场景应该有不同策略：
- CLI 可以更倾向 interrupt 或 steer
- 群聊可能适合 collect/followup
- 任务型 channel 可能适合 backlog
- 低延迟聊天可能适合 interrupt

## 11. Agent run 与 streaming

消息进入 agent loop 后，会产生：
- assistant stream
- tool stream
- lifecycle stream

但 channel 不一定直接把每个 token 发出去。  
官方 Messages 文档说明 block streaming、chunking、coalesce、人类延迟和 channel overrides 都会影响最终发出的消息。

高层模型：

```text
model output stream
  -> OpenClaw block/chunk policy
  -> channel delivery constraints
  -> outbound send
```

## 12. Chunking 与 channel limits

不同平台有不同文本长度限制和格式限制。  
OpenClaw 的 chunking 需要：
- 遵守 channel text limits
- 尽量不拆 fenced code block
- 合并小块输出
- 控制 idle-based batching
- 按 channel 覆盖 streaming 设置

这说明 delivery pipeline 不是简单 `send(text)`。  
它要把模型输出转换成平台能接受、用户能读的消息片段。

## 13. Prefix 与 reply threading

官方 Messages 文档说明 outbound formatting 集中在 `messages` 配置：

- `messages.responsePrefix`
- `channels.<channel>.responsePrefix`
- `channels.<channel>.accounts.<id>.responsePrefix`
- WhatsApp inbound `messagePrefix`
- `replyToMode`
- per-channel defaults

这说明回复格式有 cascade：

```text
global messages config
  -> channel override
  -> channel account override
  -> concrete outbound message
```

reply threading 则需要 channel plugin 把 OpenClaw 的 reply intent 映射到平台 API。

## 14. Silent replies

官方 Messages 文档说明，精确 silent token：

```text
NO_REPLY
no_reply
```

表示不发用户可见回复。  
但不同 conversation type 行为不同：
- direct conversations 默认不允许 silence，会改写成短 fallback
- groups/channels 默认允许 silence
- internal orchestration 默认允许 silence
- 有 pending spawned subagent runs 时，bare silent reply 会被丢弃，等待子 agent 完成事件

这说明 silent reply 是 delivery policy，不只是模型输出文本。

## 15. Shared `message` tool

官方 Plugin Internals 文档说明，channel plugin 不应该注册单独的普通聊天 send/edit/react tools。  
core 拥有共享 `message` tool host、prompt wiring、session/thread bookkeeping 和 execution dispatch。  
channel plugin 拥有 scoped action discovery、capability discovery、schema fragments 和最终 action adapter。

这带来两个好处：
- 模型只看到统一消息工具
- channel-specific 能力仍由插件拥有

但也要求 core 把当前 runtime scope 传给插件，例如：
- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- trusted inbound `requesterSenderId`

否则插件无法正确暴露当前 turn 可用的 action。

## 16. Outbound media

channel plugin 负责平台侧媒体发送。  
官方 Plugin Internals 文档提到 channel-specific message-tool param 如果携带 local path 或 remote media URL，插件应通过 discovery 返回 `mediaSourceParams`。

这让 core 可以做：
- sandbox path normalization
- outbound media access hints
- action-scoped media 参数识别

关键点是 action-scoped，而不是 channel-wide flat list。  
否则某个 action 的媒体参数可能误影响另一个 action。

## 17. Typing 与 heartbeat

Channel plugin 文档把 heartbeat typing 作为 channel 能力之一。  
这说明 typing/busy signal 不是 UI 装饰，而是 delivery experience 的一部分。

对长任务来说：
- heartbeat 可以避免用户以为 agent 掉线
- typing indicator 可以表达“还在处理”
- channel 限制决定是否能持续刷新

但 typing 不能替代真实状态。  
实际 run lifecycle 仍要由 Gateway/session stream 表达。

## 18. Delivery retry 与失败

官方 Messages 文档把 retry 作为相关主题。  
虽然这里不展开具体实现，但 delivery retry 至少要考虑：

- 平台 API rate limit
- 短暂网络失败
- 消息长度/格式错误
- 权限失效
- channel account logout
- media upload 失败
- thread 已关闭或不可回复

失败不一定应该重跑 agent。  
很多情况下只需要重试 delivery，或把 delivery failure 记录到 logs/events。

## 19. 与 Hooks 的关系

消息生命周期会经过 hooks：
- `message_received`
- `message_transcribed`
- `message_preprocessed`
- `message_sent`
- plugin hooks

这意味着最终进入 agent 的消息可能不是原始平台消息。  
排查问题时要同时检查：
- channel plugin normalize 逻辑
- hooks 是否改写
- mention gating 是否跳过
- queue mode 是否影响
- delivery shaping 是否改变输出

## 20. 常见风险

### 20.1 Dedupe key 不稳定

同一条平台消息重投后变成多次 agent run。

### 20.2 Mention gating 不一致

某些群聊里 agent 意外响应无关消息，或该响应时被跳过。

### 20.3 Thread grammar 漂移

同一个外部 thread 被映射到多个 session，导致上下文断裂。

### 20.4 Streaming 过度

把模型小块输出直接发到 channel，造成刷屏或平台限流。

### 20.5 Silent reply 策略错误

direct chat 中裸 `NO_REPLY` 让用户以为系统无响应。

## 21. 对源码阅读的建议

拿到完整源码后建议读：

1. `src/channels/*`
2. `src/agents/channel-tools.ts`
3. `src/agents/openclaw-tools.ts`
4. `src/agents/pi-embedded-messaging.ts`
5. `src/agents/pi-embedded-block-chunker.ts`
6. channel plugin SDK inbound helpers
7. bundled Telegram/Slack/Discord/WhatsApp channel implementations
8. message delivery retry 实现
9. hooks message pipeline 接入点
10. config schema 中 `messages.*` 与 `channels.*`

重点问题：
- inbound dedupe cache 的 key 具体包含哪些字段
- debounce 合并后的 transcript 形状
- mention gating 的 shared helper 是否所有 channel 都使用
- sessionKey 如何从 channel conversation grammar 构造
- queue mode 在 active run 时如何决定
- delivery failure 是否会重放 assistant output
- media path normalization 是否受 sandbox 影响

## 22. 结论

OpenClaw 的 channel 消息生命周期可以概括成：

```text
platform event
  -> channel plugin normalize/security
  -> dedupe/debounce/mention
  -> binding/session routing
  -> queue/agent run
  -> stream/chunk/reply shaping
  -> channel plugin outbound delivery
```

真正要记住的是：

```text
channel 不是 agent loop 的边缘输入，
而是把外部平台语义转换成 OpenClaw session/runtime 语义的关键适配层。
```

## 23. 参考

- Messages：<https://docs.openclaw.ai/concepts/messages>
- Building Channel Plugins：<https://docs.openclaw.ai/plugins/sdk-channel-plugins>
- Plugin Internals：<https://docs.openclaw.ai/plugins/architecture>
- Multi-agent Routing：<https://docs.openclaw.ai/concepts/multi-agent>
- Gateway Protocol：<https://docs.openclaw.ai/gateway/protocol>

