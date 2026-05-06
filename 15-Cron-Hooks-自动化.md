# OpenClaw Cron、Hooks 与自动化

## 1. 为什么自动化层重要

OpenClaw 不只是被动响应用户消息。  
官方文档里有 cron、hooks、webhooks、plugin hooks、gateway lifecycle hooks 等能力。

这意味着系统存在一类特殊入口：

```text
不是用户即时发送消息，
但仍会触发 agent 或平台行为。
```

这类入口如果不分析清楚，会影响：
- agent 什么时候主动运行
- 运行时用哪个 session
- 是否需要 approval
- 自动化能否调用工具
- 插件能否改变消息或工具调用
- 出错如何重试和观测

## 2. 自动化入口地图

可以把 OpenClaw 的自动化入口分成四类：

```text
cron
  定时触发 Gateway 或 agent 行为

webhook
  外部 HTTP 事件进入 Gateway

built-in hooks
  平台内部事件，例如 message_received、agent_before_run

plugin hooks
  插件注册的运行时 hook，例如 before_tool_call、after_tool_call
```

这四类入口都可能影响 agent loop，但位置不同：

```text
cron / webhook
  更像外部触发源

built-in hooks / plugin hooks
  更像管线中的拦截点和扩展点
```

## 3. Cron 的定位

官方 cron 文档说明，Gateway 级 scheduler 可以按计划运行任务。

cron 的关键不是“定时器”，而是：

```text
计划任务如何安全进入 agent runtime。
```

一个 cron job 可能只是发消息，也可能触发 agent 做检查、总结、维护、提醒或后台工作。

这类任务通常没有用户实时盯着，因此必须额外关注：
- session 选择
- approval fallback
- 工具权限
- 输出去向
- 重试和幂等
- 是否能长期运行

## 4. Cron session 模型

OpenClaw 的 cron 通常需要决定：

```text
用 main session
还是用 isolated session
```

两者差异很大：

```text
main session
  cron 内容进入主会话上下文，后续用户可见

isolated session
  cron 在独立上下文运行，避免污染主会话
```

分析判断：
- 例行检查、后台整理、监控类任务更适合 isolated session
- 需要主动提醒用户的任务，可以把结果送回 main session 或 channel
- 如果 cron 每次都写 main session，长期会显著增加 context 噪音

## 5. Cron 与 Queue

cron 触发的 agent run 仍然应该进入 queue。  
否则它可能和用户即时消息、工具结果、渠道事件并发修改同一 session。

高层链路：

```text
schedule fires
  -> cron job resolved
  -> target agent/session selected
  -> agent command enqueued
  -> agent loop runs
  -> stream/persistence/delivery
```

这说明 queue 不只是用户消息的并发控制，也是后台任务的状态收敛点。

## 6. Cron 与 Approval

cron 最大的问题之一是：触发时可能没有人在场。

因此只要 cron 能调用 `exec` 或其他高风险工具，就必须定义 approval fallback：

```text
可无人批准时：
  deny
  pending
  allow by policy
  route to operator approval
```

这和 [13-Tools-Sandbox-Exec审批.md](./13-Tools-Sandbox-Exec审批.md) 的 `askFallback` 是同一问题。

结论很直接：

```text
自动化任务的工具权限应该比交互式本地 agent 更窄。
```

## 7. Built-in Hooks

官方 hooks 文档列了不少平台事件。可以按管线阶段整理：

```text
消息入站
  message_received
  message_transcribed
  message_preprocessed

agent 运行
  agent_before_run
  agent_after_run

消息出站
  message_sent

生命周期
  gateway_start
  gateway_stop
```

这说明 hooks 不是单点 callback，而是贯穿：
- channel ingress
- media/transcription
- preprocessing
- agent loop
- delivery
- gateway lifecycle

## 8. Hook 的结构意义

hooks 的价值在于把横切逻辑从主流程里拆出来：

- 日志和审计
- 消息过滤
- 入站标准化
- 安全策略
- 自动回复
- 用户画像更新
- 工具调用治理
- 交付后的副作用

但 hooks 也会带来风险：

```text
主链路变得不透明。
```

当一个消息行为异常时，不能只看 agent loop。  
还要看有没有 hook 改写、拒绝、追加、延迟或转发了消息。

## 9. Plugin Hooks

插件 hooks 是扩展体系最强也最危险的一部分。

常见插件 hook 类型可以按阶段理解：

```text
gateway lifecycle
  gateway_start
  gateway_stop

message pipeline
  message_received
  message_preprocessed
  message_sent

agent pipeline
  agent_before_run
  agent_after_run

tool pipeline
  before_tool_call
  after_tool_call
```

其中 tool pipeline 尤其关键。  
如果插件可以拦截 tool call，就意味着插件能：
- 观察工具参数
- 拒绝工具调用
- 修改工具调用
- 记录工具结果
- 触发额外行为

这必须有严格的 hook order、权限和审计。

## 10. Terminal Decision

hooks 系统一般会出现几种返回语义：

```text
continue
  继续后续 hooks 和主流程

modify
  修改 payload 后继续

stop / reject
  终止当前流程

respond
  直接返回某个结果
```

官方插件 hook 文档提到 terminal decision 这类概念。  
它的架构含义是：

```text
某些 hook 可以成为流程终点。
```

这会影响调试和安全：
- 为什么 agent 没跑？可能某个 hook 终止了
- 为什么工具没调用？可能 before_tool_call 拒绝了
- 为什么用户收到了不同回复？可能 hook 直接 respond 了

## 11. Hook Order

hook 顺序必须稳定。  
如果多个插件同时注册同一 hook，平台需要定义：

- 谁先运行
- 谁可以修改 payload
- 谁可以终止流程
- 终止后后续 hook 是否还运行
- 错误是否中断主流程

否则插件生态会很快不可预测。

建议后续源码核查：

```text
hook priority
  -> registration order
  -> plugin enable order
  -> terminal decision
  -> error handling
```

## 12. Webhooks

webhook 是外部系统主动打进 Gateway 的入口。

它和 channel 消息有相似点，但信任模型不同：

```text
channel message
  来自已配置 channel/account/peer

webhook
  来自外部 HTTP caller，通常依赖 secret/signature/path/token
```

webhook 应该重点分析：
- 如何鉴权
- payload 如何 schema validate
- 是否映射到 agent/session
- 是否进 queue
- 是否可触发 tool/exec
- 是否有重放保护
- 是否记录审计日志

## 13. 自动化与 Context 污染

cron、hooks、webhooks 都可能把内容写进 session。

风险是：
- 大量自动化消息挤占 context
- hook 生成的 `custom_message` 被误当成用户输入
- 系统维护消息污染主会话语义
- cron 的工具结果长期进入 transcript

因此自动化最好明确区分：

```text
可见给用户的内容
  进入 chat/session transcript

运行审计内容
  进入 logs / events

长期可复用知识
  进入 curated memory

临时状态
  不进入模型上下文
```

这和 context 文档里的 `custom_message` / `custom` 区分相关。

## 14. 自动化与 Observability

自动化任务没有人在场时，观测能力非常重要。

至少应该能回答：
- 哪个 cron 触发了 run
- 哪个 hook 改写或终止了流程
- 哪个插件注册了 hook
- 哪个 webhook caller 触发了事件
- 最终写入哪个 session
- 是否触发 approval
- approval 是谁批准或拒绝
- 工具调用结果是否成功

这也说明 logs、events、health、status 不应被视为边缘功能。

## 15. 常见风险

### 15.1 Cron 权限过宽

无人值守任务拿到 coding/full profile，容易造成不可控执行。

### 15.2 Hook 静默改写消息

hook 如果修改用户输入但没有审计，排查 agent 行为会很困难。

### 15.3 Plugin hook 越权

插件通过 `before_tool_call` 观察或修改高风险工具参数，相当于进入执行安全边界。

### 15.4 Webhook 缺少重放保护

外部事件可能被重复发送。没有幂等或签名时间窗会导致重复 agent run。

### 15.5 自动化污染 main session

每天的 cron 结果都写入主会话，会让后续用户对话带上大量无关历史。

## 16. 对源码阅读的建议

拿到完整源码后建议读：

1. cron scheduler 实现
2. Gateway `cron.*` server methods
3. webhook ingress handler
4. built-in hook registry
5. plugin hook registry
6. hook runner / terminal decision 处理
7. message pipeline 入口
8. agent_before_run / agent_after_run 接入点
9. before_tool_call / after_tool_call 接入点
10. logs / event emission

重点问题：
- cron 是否持久化 nextRun / lastRun / failure
- cron run 是否总是进入 queue
- isolated session 如何命名和清理
- hook payload 是否 immutable 或可变
- 多插件 hook 顺序如何确定
- hook 抛错是否中断主流程
- webhook 是否要求签名或 secret
- webhook 到 sessionKey 的映射规则在哪里

## 17. 结论

OpenClaw 的自动化层可以这样理解：

```text
cron/webhook
  是外部或时间触发源

hooks
  是消息、agent、tool、gateway 生命周期的拦截点

plugins
  可以把 hooks 变成扩展生态

queue/session/context
  决定自动化结果如何进入状态系统
```

真正要守住的是：

```text
自动化入口必须和用户入口一样经过身份、权限、queue、工具策略和审计。
```

如果这层设计清楚，OpenClaw 就不只是聊天 agent，而是可以长期运行的 agent automation platform。

## 18. 参考

- Cron jobs：<https://docs.openclaw.ai/cron/>
- Automation hooks：<https://docs.openclaw.ai/automation/hooks>
- Plugin hooks：<https://docs.openclaw.ai/plugins/hooks>
- Gateway Protocol：<https://docs.openclaw.ai/gateway/protocol>
- Exec Approvals：<https://docs.openclaw.ai/tools/exec-approvals>

