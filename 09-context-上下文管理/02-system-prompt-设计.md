# System Prompt 设计

这个文件只讲 OpenClaw 默认 `system prompt` 的设计，不再展开 transcript、toolResult 回写、compaction 的生命周期细节。

回答的问题：

1. OpenClaw 的 system prompt 由谁拥有、何时重建
2. 默认 prompt 有哪些 section
3. 各 section 分别解决什么运行时问题
4. 为什么它要围绕 prompt caching 做结构设计

说明：
- 这里基于 2026-05-03 可见的公开文档与 `src/agents/system-prompt.ts` 等公开路径整理
- 若某条来自官方文档，直接按官方表述说明
- 若某条来自源码路径的实现还原，会明确写出

## 1. 先给一句总定义

OpenClaw 的 system prompt 不是“写给模型的一段固定人设词”，而是一份运行时协议。

它要同时解决：
- 工具如何用
- 安全边界是什么
- 当前 workspace / sandbox / channel / runtime 能力是什么
- 如何回复、何时静默、何时发消息工具
- 如何把宿主能力暴露给模型且尽量保持 cache 稳定

所以这不是一个纯 prompt engineering 文本，而是执行层和宿主层之间的说明书。

## 2. 总设计原则

官方 `System prompt` 文档给出的原则非常明确：

1. prompt 是 OpenClaw-owned
   - 不使用 pi-coding-agent 默认 prompt

2. 每次 run 重建
   - 不是“启动时构一次以后复用”

3. 结构刻意做得紧凑、固定
   - 便于 provider cache 命中

4. provider 可以做小范围 overlay
   - 替换指定 section
   - 在 cache boundary 上下插入前后缀

5. sub-agent 可以使用更小的 promptMode
   - `full`
   - `minimal`
   - `none`

如果只记一句话：

```text
OpenClaw 的 system prompt 目标不是“写得像人”，而是“足够稳定、足够明确、足够能驱动执行”。
```

## 3. 默认 system prompt 的一级 section

官方 `System Prompt` 页面明确公开了这些 section：

1. `Tooling`
2. `Execution Bias`
3. `Safety`
4. `Skills`
5. `OpenClaw Self-Update`
6. `Workspace`
7. `Documentation`
8. `Workspace Files (injected)`
9. `Sandbox`
10. `Current Date & Time`
11. `Reply Tags`
12. `Heartbeats`
13. `Runtime`
14. `Reasoning`

这些 section 的设计思路并不是“按知识分类”，而是按运行时问题分类。

## 4. 一级 section 分别解决什么问题

### 4.1 `Tooling`

作用：
- 告诉模型当前有哪些工具
- 说明工具调用的基本规则
- 解释 approval、cron、subagent、长任务等运行时约束

重点不是“列出工具名”，而是把“如何正确用工具”写进去。

### 4.2 `Execution Bias`

作用：
- 约束模型偏向执行而不是空谈
- 能推进就这轮推进
- 工具结果弱时继续换路径验证
- 真阻塞时才提出缺失信息

这一节本质上是在对抗模型常见的“只给计划、不落地”的倾向。

### 4.3 `Safety`

作用：
- 约束高风险行为
- 防止主动扩权、绕监督、改安全边界
- 给工具使用和自修改能力设护栏

这节是 OpenClaw 宿主安全边界向模型的显式投影。

### 4.4 `Skills`

作用：
- 只注入 skill metadata
- 不把所有 `SKILL.md` 全量塞进 system prompt
- 让模型按需用 `read` 去读最相关技能

它解决的是技能数量增长后 prompt 膨胀的问题。

### 4.5 `OpenClaw Self-Update`

作用：
- 告诉模型如何安全查看 config schema
- 何时允许 `config.patch` / `config.apply` / `update.run`
- 防止 agent 乱改宿主配置

这节是“agent 能不能动宿主本身”的边界声明。

### 4.6 `Workspace`

作用：
- 说明当前工作目录
- 约束相对路径和默认工作区行为
- 让文件工具、exec 工具和项目上下文保持一致

### 4.7 `Documentation`

作用：
- 告诉模型本地 docs、镜像站、GitHub source 在哪
- 约束 OpenClaw 自身行为问题时优先读本地文档/源码
- 降低对平台自身机制的幻觉回答

### 4.8 `Workspace Files (injected)`

作用：
- 提前告诉模型，后续会注入 project context files
- 让模型知道这些文件是高优先级运行时背景

这节本身不承载业务知识，而是注入协议的说明。

### 4.9 `Sandbox`

作用：
- 告诉模型当前是否在 sandbox 中
- 是否允许 elevated exec
- workspace mount 与宿主访问边界是什么

这一节让模型明确“能做什么、不能做什么”。

### 4.10 `Current Date & Time`

作用：
- 提供用户时区
- 必要时时间感知
- 但尽量减少高波动时间字段，服务于 cache 稳定

### 4.11 `Reply Tags`

作用：
- 告诉模型某些 provider / channel 需要的回复标签
- 让最终输出能被宿主正确解释

### 4.12 `Heartbeats`

作用：
- 说明 heartbeat poll 的特殊回复协议
- 比如何时只回 `HEARTBEAT_OK`

这是一种把运维流程编码进 prompt 的做法。

### 4.13 `Runtime`

作用：
- 注入 host / os / node / model / repoRoot / thinking 等运行时摘要
- 让模型知道当前真实环境

### 4.14 `Reasoning`

作用：
- 告诉模型 reasoning 当前是否可见
- `/reasoning` toggle 是什么语义

这节主要服务于不同展示模式下的输出一致性。

## 5. 从 `src/agents/system-prompt.ts` 可见的内部细分 section

公开源码路径还能看到更多真实使用的 section 或 section builder。

### 5.1 顶层身份句

默认开头是：

```text
You are a personal assistant running inside OpenClaw.
```

这不是花哨人设，而是最基础的 runtime identity。

### 5.2 `Tooling`

除了工具名和摘要，它还塞进很多运行时约束，例如：
- 工具名区分大小写
- approval 的具体处理方式
- 长任务该用 cron 还是 process
- 有 subagent 时优先 `sessions_spawn`
- 不要用 `exec` 自己拼消息通道

它的设计目标不是“告诉模型有什么工具”，而是“减少工具使用上的错误姿势”。

### 5.3 `Execution Bias`

源码里这块非常实用，核心约束是：
- 可执行请求应当当轮推进
- 非最终轮也要靠工具实质推进
- 真阻塞时才提唯一缺失决策
- 结论要有证据

### 5.4 `Safety`

源码里的 Safety 比文档更具体，除了高层 guardrail，还有：
- 不追求自我保存、扩张资源、复制自己
- 不说服别人放宽权限
- 非明确请求下不要改 system prompt / safety rules / tool policy

### 5.5 `Skills (mandatory)` / Memory section

源码实现里：
- 有可用 skills 时，加入技能元信息和“只读最相关一个 `SKILL.md`”的规则
- Memory section 则通过 `buildMemoryPromptSection(...)` 条件注入

这说明 system prompt 也承担“何时读技能、何时查 memory”的路由职责。

### 5.6 `OpenClaw Self-Update`

源码可见这块非常强调：
- config 变更前先 `config.schema.lookup`
- 不要猜字段名
- `update.run` 只有用户明确要求才允许
- Gateway service lifecycle 命令也有限制

### 5.7 `Model Aliases`

如果配置里有 model aliases，会注入：
- 优先使用 alias 指定模型
- 完整 `provider/model` 也接受

它解决的是多模型环境下模型引用歧义问题。

### 5.8 `Workspace`

除了工作目录路径，还会告诉模型：
- 把这里当作默认 workspace
- 相对路径优先
- repoRoot / docsPath / sourcePath 的关系

这是文件工具和 exec 工具保持一致性的关键。

### 5.9 `Documentation`

源码里的文档 section 比公开文档更具体：
- local docs path
- mirror
- source repo
- Discord
- ClawHub
- 出现 OpenClaw 行为/配置问题时，优先读 local docs/source

这块的目的是减少模型对 OpenClaw 自身行为的幻觉回答。

### 5.10 `Sandbox`

源码里这块信息很多，包括：
- 是否 Docker sandbox
- container workdir
- host mount source
- workspace access
- browser bridge
- elevated `on/off/ask/full` 是否可用
- full access 被什么限制阻断

system prompt 在这里承担的是“边界可视化”的作用。

### 5.11 `Authorized Senders`

源码里有 owner / sender 相关 section builder：
- 允许显示 raw 或 hash 过的 sender id
- 告诉模型哪些 sender 在 allowlist

它主要服务于消息渠道和 sender-aware policy。

### 5.12 `Current Date & Time`

当前实现更偏 cache-friendly：
- 主要放时区
- 真要知道“现在几点”，提示模型去跑 `session_status`

这说明时间信息本身也在为 prompt caching 让路。

### 5.13 `Workspace Files (injected)` + `Project Context`

这是 OpenClaw system prompt 里最重要的运行时注入机制之一。

默认注入文件包括：
- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`
- `MEMORY.md`（存在时）

并且：
- 大文件会截断
- 有总注入上限
- 缺失文件会插入 missing marker
- sub-agent 默认只注入 `AGENTS.md` 与 `TOOLS.md`

### 5.14 `Assistant Output Directives`

源码里还有一层输出协议说明，例如：
- `MEDIA:`
- `[[audio_as_voice]]`
- `[[reply_to_current]]`

这类内容不是业务知识，而是告诉模型怎样生成带交付元数据的回复。

### 5.15 `Control UI Embed`

Webchat / Control UI 场景下，源码还会加：
- `[embed ...]` 只能在 webchat 用
- hosted embed 的 URL / ref 规范

这类 section 完全是面向特定前端宿主能力的 prompt 设计。

### 5.16 `Messaging`

当有 `message`、`sessions_send`、`sessions_spawn`、`subagents` 等工具时，system prompt 会告诉模型：
- 当前 session 回复如何自动路由
- 跨 session 应该用什么工具
- 子 agent 应该怎么起、怎么等
- 若 `message_tool_only`，最终 answer 与可见消息如何避免重复

这部分是 OpenClaw 区别于纯 coding agent 的重要 section。

### 5.17 `Voice (TTS)` / `Reactions`

源码里还有条件性的：
- `Voice (TTS)` section
- Telegram 等 channel 的 `Reactions` guidance

说明 system prompt 是按 channel capability 动态塑形的。

### 5.18 `Silent Replies`

这部分非常关键：
- 当确实不需要对用户说话时，必须只返回 `NO_REPLY`
- 不可混在正常回复里
- 不可包在 code block 里

这直接服务于：
- silent memory flush
- background housekeeping
- 使用 `message` 工具后避免重复回复

### 5.19 `Group Chat Context` / `Subagent Context`

当 runtime 需要注入额外系统提示时：
- 主 agent 走 `Group Chat Context`
- `promptMode=minimal` 时走 `Subagent Context`

这表明 OpenClaw 明确区分主对话和子 agent 的上下文角色。

### 5.20 `Heartbeats`

当 heartbeat 生效时，prompt 会明确：
- 若当前是 heartbeat poll 且一切正常，回复 `HEARTBEAT_OK`
- 若真有问题，不要带 `HEARTBEAT_OK`

这是一种用 prompt 约束 agent 运维行为的设计。

### 5.21 `Runtime` / `Reasoning`

最后还会注入运行时摘要：
- agentId
- host
- repo
- os/arch
- node
- model/default_model
- shell
- channel/capabilities
- thinking

以及 reasoning 可见性说明。

## 6. 为什么它要围绕 prompt cache 来设计

OpenClaw 这一块做得很工程化。

官方 `Prompt Caching` 文档明确说：
- system prompt 被切成 stable prefix + volatile suffix
- 两者之间有 internal cache boundary

设计目的：
- 尽量让工具定义、skills 元信息、稳定 workspace files 跨轮保持 byte-identical
- 把 `HEARTBEAT.md`、动态 runtime metadata 等易变内容放到 boundary 下方
- 减少不必要的 `cacheWrite`

几个关键点：
- 稳定 project context file 会排在 `HEARTBEAT.md` 之前
- system prompt fingerprint 会做 whitespace / line ending / capability ordering 归一化
- tool catalog 顺序也尽量 deterministic

因此 system prompt 不是只追求“表达清楚”，还要追求“高缓存复用率”。

## 7. 一个适合记忆的系统观

如果把这整块压缩成一句话：

```text
OpenClaw 的 system prompt 不是固定人设，而是对工具、安全、宿主能力、输出协议和缓存边界的模块化声明。
```

真正重要的不是某一段 wording，而是：
- section 化
- 条件注入
- provider overlay
- promptMode 分级
- cache-friendly 组织方式

## 8. 参考

- System Prompt: <https://docs.openclaw.ai/concepts/system-prompt>
- Prompt Caching: <https://docs.openclaw.ai/reference/prompt-caching>
- Source path: `src/agents/system-prompt.ts`

