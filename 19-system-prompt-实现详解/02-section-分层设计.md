# Section 分层设计

OpenClaw 的 system prompt section 不是按“人设、规则、背景资料”这种传统 prompt 分类方式组织的，而是按 agent 运行时问题组织的。

如果把它拆成几层，可以看成：

```text
身份层
  -> 工具与行动层
  -> 安全与控制层
  -> 知识/技能发现层
  -> workspace 与环境层
  -> channel 交付层
  -> 运维协议层
  -> runtime 诊断层
```

## 1. 身份层

顶层身份句很短：

```text
You are a personal assistant running inside OpenClaw.
```

它没有把模型塑造成复杂角色，而是声明“你运行在 OpenClaw 里”。这句话的作用是确定宿主边界：模型不是裸聊天机器人，而是 OpenClaw runtime 内的 agent。

源码还可能追加当前 model identity。这样当用户问“你现在是什么模型”时，模型可以按当前 run 的 runtime 信息回答，而不是凭 provider 默认自我认知猜。

## 2. 工具与行动层

这一层主要包含：
- `Tooling`
- `Tool Call Style`
- `Execution Bias`
- `Sub-Agent Delegation`

### 2.1 Tooling

`Tooling` 的作用不只是列工具。它同时说明：
- 工具是 policy-filtered 的
- 工具名大小写敏感
- `TOOLS.md` 只是 usage guidance，不决定工具可用性
- 长任务不要用高频轮询
- future follow-up 应该用 cron
- 大任务可以用 `sessions_spawn`
- 子 agent 完成是 push-based，不应循环查状态

这体现了一个设计选择：把“工具如何正确使用”放在 system prompt 中，把“工具是否真的可用”留给工具 schema 和 policy。

### 2.2 Tool Call Style

`Tool Call Style` 主要压低模型在常规工具调用前后的废话，鼓励低风险调用直接执行。它还会写入 exec approval 的处理规则：
- native approval UI 可用时优先用按钮/card
- 需要 chat approval 时才展示 `/approve`
- `/approve` 是用户可见命令，不是 shell 命令
- approval 是单条命令粒度，不能把旧 approval 当成新命令的许可

这部分其实是工具治理的一部分。它把 approval 这种宿主协议显式投射给模型，减少模型走错通道。

### 2.3 Execution Bias

`Execution Bias` 是对抗“只给计划、不推进”的 section。它要求：
- 可执行请求就在当前 turn 推进
- 非最终 turn 要用工具产生实际进展
- 真正阻塞时才问缺失决策
- 弱工具结果要换查询、路径、命令或来源继续验证
- 可变事实必须实时检查
- 最终回答需要证据

这不是个性化语气，而是工作流约束。

### 2.4 Sub-Agent Delegation

当配置倾向于委派且 `sessions_spawn` 可用时，prompt 会出现更强的子 agent 委派指导。它把主 agent 定义为 responsive coordinator：
- 简单问题直接答
- 非平凡工作交给 `sessions_spawn`
- spawn 时说明 objective、output、files、write scope、verification
- completion 走 push-based 事件，不轮询

这说明 OpenClaw 把 multi-agent orchestration 也放进 system prompt 的运行协议里，而不是只靠工具名称让模型自行摸索。

## 3. 安全与控制层

这一层主要包含：
- `Safety`
- `OpenClaw Control`
- `OpenClaw Self-Update`
- `Sandbox`
- `Authorized Senders`

### 3.1 Safety

源码里的 `Safety` 很短，但目标明确：
- 不发展独立目标
- 不自我保存、复制、扩权、谋求资源
- 安全与监督优先于任务完成
- 用户 stop/pause/audit 时要服从
- 不说服别人放宽权限或关闭保护
- 非明确请求下不改 prompt、安全规则或工具策略

注意：官方文档也强调这类 safety guardrail 是 advisory。真正的硬边界要靠 tool policy、exec approvals、sandbox 和 channel allowlist。

### 3.2 OpenClaw Control / Self-Update

`OpenClaw Control` 告诉模型不要编造命令，配置与 restart 优先走 `gateway` tool。`OpenClaw Self-Update` 只有在 gateway 工具可见且非 minimal prompt 时出现，核心规则是：
- 只有用户明确要求才做 update/config 写入
- 改配置前先查 schema 精确路径
- 使用 `config.get`、`config.patch`、`config.apply`、`update.run`
- restart 优先用 `restart`，不要 stop+start

这块约束的是“agent 能不能改宿主本身”。OpenClaw 允许 agent 维护自己，但把入口、前置检查和用户授权写得很窄。

### 3.3 Sandbox

`Sandbox` section 只有启用 sandbox 时出现。它会告诉模型：
- 工具在 Docker sandbox 中执行
- file tools 和 exec 的路径解析可能不同
- sandbox container workdir 是什么
- host mount source 是否只是 file-tools bridge
- workspace access 是什么
- elevated exec 是否可用
- `/elevated full` 是否被 host policy、channel 或 sandbox 限制

这块的关键价值是边界可视化。模型通常不知道自己在容器、宿主还是混合路径里运行；OpenClaw 通过 prompt 把这些边界显式化。

### 3.4 Authorized Senders

当存在 owner/sender 配置时，prompt 会注入授权 sender 信息，并且可以选择 raw 或 hash 显示。这里有一个细节：即使 sender 在 allowlist，prompt 仍提醒模型不要把它直接等同于 owner。

这说明 OpenClaw 对消息渠道的身份边界比较谨慎：allowlisted sender 是访问控制信号，不是无限授权信号。

## 4. 知识/技能发现层

这一层主要包含：
- `Skills`
- `Memory`
- `Documentation`
- `Model Aliases`

### 4.1 Skills

Skills section 注入的是技能索引，而不是技能正文。模型看到 name、description、location 后，只有在明确适用时才读对应 `SKILL.md`。

这解决两个问题：
- 避免所有技能正文挤爆 system prompt
- 让技能加载变成可观察的 tool read 行为

它也降低了错误技能污染主上下文的概率。

### 4.2 Memory

Memory section 由 memory 子系统构建，通常依赖可用工具和 citations 配置。它不是直接把所有历史记忆塞入 prompt，而是告诉模型如何按需查。

和 Skills 一样，Memory 的设计方向也是“索引常驻，正文按需”。

### 4.3 Documentation

Documentation section 会指向本地 docs 或公共镜像，并告诉模型：
- OpenClaw 行为、配置、架构问题优先读本地 docs
- config 字段用 `gateway config.schema.lookup`
- docs 不足时再查源码
- 诊断问题时能跑 `openclaw status` 就自己跑

这块的目的不是提供业务知识，而是降低模型对 OpenClaw 自身机制的幻觉。

### 4.4 Model Aliases

如果配置了模型别名，prompt 会说明优先使用 alias，同时接受完整 `provider/model`。这服务于多 provider、多模型环境下的引用一致性。

## 5. Workspace 与环境层

这一层主要包含：
- `Workspace`
- `Workspace Files (injected)`
- `Project Context`
- `Dynamic Project Context`
- `Current Date & Time`
- `Runtime`

### 5.1 Workspace

Workspace section 会说明当前工作目录，并根据 sandbox 状态给出路径规则。非 sandbox 情况下，它告诉模型把这个目录当作 file operations 的默认 workspace。sandbox 情况下，它会区分 host workspace 和 container workdir。

这个 section 是文件工具和 shell 工具的一致性基础。

### 5.2 Workspace Files / Project Context

Workspace 文件会被放到 `Project Context` 下。源码里对 context files 做固定排序，并区分稳定文件与动态文件：
- `AGENTS.md`、`SOUL.md`、`IDENTITY.md`、`USER.md`、`TOOLS.md`、`BOOTSTRAP.md`、`MEMORY.md` 等偏稳定
- `HEARTBEAT.md` 被视为动态文件

稳定文件尽量放在 cache boundary 上方；动态文件放在 boundary 下方。

### 5.3 Current Date & Time

当前实现偏 cache-friendly。官方文档说明 system prompt 主要保留用户时区，而不是每轮注入动态时钟；如果需要当前时间，模型应调用 `session_status`。

这是一个典型取舍：牺牲一点直接可见信息，换取 prompt cache 稳定。

### 5.4 Runtime

Runtime section 在末尾注入 agent、host、repo、OS、node、model、default model、shell、channel、capabilities、thinking 等摘要，并附带 reasoning 可见性说明。

它被放在 dynamic suffix 的末端，说明这些运行态信息经常随 run 或 channel 改变，不适合放入 cache-stable prefix。

## 6. Channel 交付层

这一层主要包含：
- `Assistant Output Directives`
- `Messaging`
- `Control UI Embed`
- `Voice (TTS)`
- `Reactions`
- `Silent Replies`

### 6.1 Assistant Output Directives

不同 channel 对附件、语音、引用回复有不同交付协议。OpenClaw 会根据 source reply delivery mode 注入不同规则：
- message-tool-only 模式下，用 message tool attachment 字段
- 普通模式下，可以用 `MEDIA:`、`[[audio_as_voice]]`、`[[reply_to_current]]` 等指令

这让模型的最终文本可以被 OpenClaw 渲染层正确解释。

### 6.2 Messaging

Messaging section 是 OpenClaw 区别于纯 coding agent 的关键。它说明：
- 当前 session 回复如何路由
- 何时使用 `message(action=send)`
- 跨 session 用 `sessions_send`
- 子 agent 用 `sessions_spawn`
- 不要用 `exec/curl` 直接调用 provider messaging
- 用 message tool 发了可见内容后，最终 answer 如何避免重复

也就是说，OpenClaw 把“说话”也建模为工具化交付，而不仅是模型返回文本。

### 6.3 Control UI / Voice / Reactions

这些 section 都是条件注入：
- webchat 下才有 Control UI embed 指导
- 有 TTS hint 才有 Voice section
- channel 开启 reactions 才有 reaction guidance

这说明 prompt 是按 channel capability 塑形的，而不是所有 agent 共用一套固定输出协议。

### 6.4 Silent Replies

Silent Replies 约束模型在无需输出时只返回静默 token，并且不能把它混进正常回复或 code block。

它服务于几个场景：
- background housekeeping
- memory flush
- message tool 已经发送可见回复后的去重
- heartbeat 正常无事时的低噪声行为

## 7. 运维协议层

这一层主要包含：
- `Heartbeats`
- `Bootstrap Pending`
- `Group Chat Context`
- `Subagent Context`

### 7.1 Heartbeats

当 heartbeat 生效时，prompt 会定义 heartbeat poll 的特殊协议：无事时返回固定 OK token，有事时返回 alert text。这样 OpenClaw 可以定期唤醒 agent，又不让它每次都发一段无意义回复。

### 7.2 Bootstrap Pending

全新 workspace 有 `BOOTSTRAP.md` 时，OpenClaw 会加入 bootstrap pending 指导。它要求模型先处理 bootstrap，而不是普通问候，并且不能假装 bootstrap 已完成。

这把初始配置流程也纳入了 prompt 协议。

### 7.3 Group Chat Context / Subagent Context

`extraSystemPrompt` 在主 agent 下叫 `Group Chat Context`，在 minimal prompt 下叫 `Subagent Context`。这个命名差异不只是标题变化，它提示模型当前上下文属于主对话还是子 agent 执行环境。

## 8. 小结

OpenClaw 的 section 设计可以概括为：

```text
先定义能做什么和怎么做，
再定义不能越过什么边界，
再给出按需加载知识的方法，
再说明当前 workspace/channel/runtime 的真实状态，
最后把可变运维信息放到 cache boundary 后面。
```

这种设计比“堆一堆规则”更工程化，因为每个 section 都对应一个明确的运行时问题。
