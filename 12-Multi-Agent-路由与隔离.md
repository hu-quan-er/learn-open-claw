# OpenClaw Multi-Agent 路由与隔离

## 1. 为什么要补 Multi-Agent

前面的文档已经分析了 session、group、queue 和 agent loop，但还没有把“一个 Gateway 同时托管多个 agent”讲清楚。

官方 Multi-agent Routing 文档明确说，OpenClaw 支持在一个运行中的 Gateway 里管理多个隔离 agent，并通过 bindings 把入站消息路由到正确 agent。

这意味着：

```text
OpenClaw 的 agent 不是单例。
Gateway 才是宿主进程，agent 是可被路由和隔离的运行域。
```

## 2. 什么算“一个 agent”

官方文档把一个 agent 定义成完整的 per-persona scope。

一个 agent 至少拥有：

```text
workspace
  AGENTS.md / SOUL.md / USER.md / local notes / persona rules

agentDir
  auth profiles / model registry / per-agent config

session store
  chat history + routing state
```

默认路径大致是：

```text
workspace
  ~/.openclaw/workspace
  或 ~/.openclaw/workspace-<agentId>

agentDir
  ~/.openclaw/agents/<agentId>/agent

sessions
  ~/.openclaw/agents/<agentId>/sessions
```

这说明 `agentId` 不只是 UI 上的名字，而是磁盘状态、身份、模型、技能和会话的隔离键。

## 3. Single-agent 模式只是默认值

如果不配置多 agent，OpenClaw 默认运行一个 agent：

```text
agentId = main
session key = agent:main:<mainKey>
workspace = ~/.openclaw/workspace
state = ~/.openclaw/agents/main/agent
```

这解释了为什么前面很多文档里会看到 `agent:main:main` 一类 key。  
它不是系统只能有 main agent，而是默认情况下 agentId 和 mainKey 都落在 main。

## 4. `agentId`、`accountId`、`binding`

Multi-agent 路由里最重要的三个词：

```text
agentId
  一个完整 agent brain：workspace、auth、model、sessions

accountId
  某个 channel 的一个账号实例，例如 WhatsApp personal / biz

binding
  把 channel/account/peer/guild/team 等匹配条件映射到 agentId
```

高层关系：

```text
inbound message
  -> channel account
  -> binding match
  -> agentId
  -> sessionKey
  -> agent loop
```

## 5. Binding 匹配规则

官方文档说明 bindings 是 deterministic，且 most-specific wins。

当前规则可以整理为：

```text
1. peer match
   精确 DM / group / channel id

2. parentPeer match
   thread inheritance

3. guildId + roles
   Discord role routing

4. guildId
   Discord guild 级别

5. teamId
   Slack team 级别

6. accountId match
   某 channel 的账号级 fallback

7. channel-level match
   accountId: "*"

8. default agent
   agents.list[].default，或者第一个 agent，最后 fallback 到 main
```

如果同一级别有多个 binding 命中，按配置顺序取第一个。  
如果一个 binding 同时设置多个 match 字段，语义是 AND，不是 OR。

这意味着 binding 配置顺序本身是路由逻辑的一部分。

## 6. 多账号与多 agent

一个典型场景：

```text
WhatsApp personal account -> home agent
WhatsApp biz account      -> work agent
Telegram account          -> coding agent
```

`accountId` 让同一个 channel 类型可以同时有多个账号实例。  
这样一个 Gateway 可以维护多个 WhatsApp、Telegram、Discord 或 Slack 账号，并把它们路由到不同 agent。

这比“每个 agent 启一个 Gateway”更集中：
- Gateway 统一管理 channel 连接
- agent 状态仍可隔离
- 配置和运维面更收敛

代价是 binding 与权限配置必须足够清楚。

## 7. Direct chat 与 main session

官方文档提醒：direct chats 会 collapse 到该 agent 的 main session key。

这意味着：
- 同一个 agent 下的 direct chat 不一定天然形成完全隔离的 session 空间
- 如果要让不同人真正隔离，应该用不同 agent
- 只靠 peer routing 到同一个 agent，仍可能共享 agent workspace、auth、memory 和 main session 行为

所以多用户场景下的隔离建议是：

```text
不同人 / 不同组织 / 不同信任域
  -> 不同 agentId

同一人不同渠道或不同账号
  -> 可以考虑同一 agentId + 不同 binding
```

## 8. Per-agent workspace 与 auth

每个 agent 都可以有自己的 workspace 和 auth profiles。

官方文档强调不要复用 `agentDir`，否则会造成 auth/session collision。

这点非常重要：

```text
workspace 隔离不等于 agent 隔离。
agentDir 复用会让认证、模型 registry、session store 发生混线。
```

如果二开或部署时手动配置 agent，至少要保证：
- `id` 唯一
- `workspace` 分开
- `agentDir` 分开
- session store 分开
- channel binding 明确

## 9. Skills 与 memory search

Multi-agent 文档提到，skills 会从每个 agent workspace 和共享 roots 加载，再按 agent 的 skill allowlist 过滤。

这意味着技能可见性也是 agent-scoped 的一部分。

另外，官方文档还提到 cross-agent QMD memory search。  
如果一个 agent 需要搜索另一个 agent 的 QMD session transcripts，要显式配置 extra collections。

分析判断：
- 默认隔离是主线
- 跨 agent 记忆访问需要显式配置
- 这比默认共享所有记忆更安全

## 10. Per-agent sandbox 与 tools

官方文档支持 per-agent sandbox 和工具限制，例如：
- 某个私人 agent 不启 sandbox
- 某个家庭/群组 agent 始终 sandbox
- 某个 agent 只允许 read，禁止 exec/write/edit/apply_patch

这说明 agent 隔离不只在状态层，还可以延伸到执行能力层。

高层模型：

```text
agentId
  -> workspace
  -> agentDir
  -> auth profiles
  -> sessions
  -> model defaults
  -> skills
  -> sandbox policy
  -> tools allow/deny
```

这才是完整的 agent boundary。

## 11. 与 Session / Group 的关系

Multi-agent 不替代 session/group，而是在它们外面再加一层 agent scope。

可以这样看：

```text
Gateway
  -> agentId
    -> sessionKey
      -> sessionId
        -> transcript / run state
      -> groupId
        -> session family / branching
```

其中：
- `agentId` 决定哪个 brain 和状态目录
- `sessionKey` 决定某条对话或渠道线程如何落到会话入口
- `sessionId` 决定具体会话实例
- `groupId` 决定更高层的会话族或分支关系

如果只看 session，会漏掉 agent 级隔离。  
如果只看 agent，又会看不懂会话分支和 transcript lifecycle。

## 12. 常见配置风险

### 12.1 复用 `agentDir`

这是最危险的一类配置错误。  
结果可能是 auth profile、sessions、model registry 混在一起。

### 12.2 Binding 过宽

例如给某个 channel 设置宽泛 fallback，但忘了更具体的 peer binding。  
消息可能落到默认 agent，而不是预期 agent。

### 12.3 同一信任域内共享 workspace

如果两个 agent 使用同一 workspace，就会共享 `AGENTS.md`、`SOUL.md`、`USER.md`、memory 和 workspace skills。  
除非刻意设计，否则不建议这样做。

### 12.4 以为 channel allowlist 等于 agent allowlist

官方文档提醒，某些 DM access control 是 channel/account 级的，不一定是 per-agent 的。  
所以 channel 入口控制和 agent 路由控制要分开核查。

## 13. 对源码阅读的建议

后续拿到完整源码后，建议重点读：

1. agent config schema
2. binding resolver
3. channel inbound message 到 `agentId` 的路由入口
4. `sessionKey` 构造逻辑
5. per-agent workspace / agentDir 解析
6. auth profile 读取路径
7. per-agent skills 解析与 allowlist
8. per-agent sandbox / tools policy 合并逻辑

重点问题：
- binding 的 most-specific wins 在代码中如何排序
- `accountId` 省略时如何选择 default account
- direct chat collapse 到 main session key 的逻辑在哪里
- `agentId` 如何进入 system prompt、tools、session store 和 model auth
- 跨 agent memory search 有没有默认关闭

## 14. 结论

OpenClaw 的 multi-agent 设计说明它不是“一个 assistant 多开几个会话”，而是：

```text
一个 Gateway
  托管多个 agent brain
  每个 brain 有自己的 workspace / agentDir / auth / sessions
  入站消息通过 binding 路由
  执行能力可按 agent 限制
```

这层补上以后，OpenClaw 的状态模型可以更完整地理解为：

```text
Gateway
  -> channel/account/binding
  -> agentId
  -> sessionKey/group/sessionId
  -> agent loop
```

## 15. 参考

- Multi-agent Routing：<https://docs.openclaw.ai/concepts/multi-agent>
- Configuration agents：<https://docs.openclaw.ai/gateway/config-agents>
- Agent Workspace：<https://docs.openclaw.ai/concepts/agent-workspace>

