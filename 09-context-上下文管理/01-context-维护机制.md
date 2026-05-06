# Context 维护机制

这个文件只讲 OpenClaw 如何维护 `context` 本身，不展开默认 `system prompt` 的 section 设计。

回答的问题：

1. OpenClaw 的 context 到底由哪些部分组成
2. 用户输入、assistant 返回、tool call / tool result 如何进入上下文
3. context 是怎样从 transcript 组装出来的
4. compaction 如何触发、切块、持久化

说明：
- 这里基于 2026-05-03 可见的公开文档与公开源码路径整理
- 若某条来自官方文档，直接按官方表述说明
- 若某条来自运行时路径与实现还原，会明确写出

## 1. 先给一句总定义

OpenClaw 里的 `context` 不是“最近几轮聊天消息”，而是：

```text
system prompt
+ tool schemas
+ 会话历史（user / assistant / toolResult / custom_message / compaction）
+ 附件与外部内容（图片、音频、文件读取结果、命令输出）
+ provider 为兼容性附加的包装
```

官方文档把它定义为“这次 run 发给模型的全部内容”，而不是“磁盘上存过的全部历史”。

所以：
- `memory` 是可持久化、可检索、可按需读回的长期材料
- `context` 是这一轮真正塞进模型窗口里的东西

## 2. Context 的三层结构

更适合工程上理解的拆法是三层。

### 2.1 持久化层

OpenClaw 用两层持久化维护会话状态：

1. `sessions.json`
   - `sessionKey -> SessionEntry`
   - 记录当前 `sessionId`、更新时间、toggle、token counters、override、compactionCount 等

2. `<sessionId>.jsonl`
   - append-only transcript
   - 第一行是 session header
   - 后续行是树状 entry，带 `id + parentId`
   - 记录真实 user/assistant/toolResult/custom_message/compaction 等内容

这意味着“上下文状态”不是纯内存对象，而是 session store + transcript 两层共同维护。

### 2.2 运行时组装层

每次 run 开始时，OpenClaw 会基于当前 session：
- 打开 transcript
- 修复或清洗必要的历史
- 构建 system prompt
- 让 context engine 决定这轮要送哪些 message
- 再把这些内容连同 tool schemas 一起交给模型

也就是：

```text
磁盘 transcript
  -> 运行时 sanitize / assemble
  -> 本轮模型上下文
```

### 2.3 交付与回写层

模型返回后，OpenClaw 不是只把一段文本吐给用户，而是会：
- stream assistant delta
- stream tool start/end
- 把 tool call / tool result 写回 transcript
- 把最终 assistant reply 写回 transcript
- 更新 `sessions.json` 里的 token 计数、compactionCount、lastActivity 等元数据

## 3. 用户输入如何进入 context

### 3.1 先过 Gateway 和 session 路由

用户消息不会直接喂给模型，而是先经过：

```text
入口 channel / UI / CLI
  -> Gateway
  -> sessionKey 路由
  -> sessionId 选择
  -> transcript append
  -> runEmbeddedPiAgent
```

其中：
- `sessionKey` 决定“这条消息属于哪个会话桶”
- `sessionId` 决定“当前续写哪份 transcript”

### 3.2 Slash commands 与 directives 不一定进入模型

官方 `Context` 文档明确区分了几种情况：

1. 纯命令消息
   - 比如只发 `/compact`
   - Gateway 直接执行，不进模型

2. directives
   - `/think`、`/verbose`、`/trace`、`/reasoning`、`/elevated`、`/model`、`/queue`
   - 会被 Gateway 剥离后再决定是否作为 hint 生效

3. 普通自然语言消息
   - 作为新的 user turn 进入 transcript 和本轮 context

所以用户看到的是“我发了一条消息”，但模型实际看到的可能是“剥离了 control directive 的有效消息文本”。

### 3.3 某些 user turn 会被打 provenance 标记

官方 `Transcript Hygiene` 文档提到：
- 如果一个 agent 通过 `sessions_send` 往另一个 session 发消息
- OpenClaw 仍然会把它存成 `role: "user"`，以兼容 provider
- 但会给该 turn 加 `message.provenance.kind = "inter_session"`
- 在后续重建 context 时，会在内存里给这类 turn 加一个 `[Inter-session message]` 标记

这说明 OpenClaw 并没有把“所有 user role”都当作“真人用户直接输入”。

### 3.4 bootstrap pending 时还会插入 user prompt prefix

从 `src/agents/system-prompt.ts` 可见，OpenClaw 在 bootstrap 未完成的 workspace 上会构造额外的 user prompt prefix：
- `full bootstrap pending`
- `limited bootstrap pending`

它的作用不是修改 system prompt，而是在当前用户 turn 前面补一段“先按 `BOOTSTRAP.md` 干活”的前缀。

所以当前轮的“用户输入”在送模型前，可能是：

```text
[Bootstrap pending]
请先按 BOOTSTRAP.md 完成初始化...

用户原始输入
```

## 4. 模型返回与工具结果如何维护

### 4.1 assistant 输出不是一段裸文本

在 OpenClaw 里，assistant 的输出通常分为三层：

1. Pi runtime 内部事件
   - `message_start` / `message_update` / `message_end`
   - `tool_execution_start` / `tool_execution_update` / `tool_execution_end`
   - `turn` / `agent` lifecycle

2. OpenClaw stream
   - `assistant`
   - `tool`
   - `lifecycle`

3. transcript 持久化 entry
   - assistant message
   - toolResult message
   - compaction entry

因此“模型返回”实际上同时存在于：
- 实时事件流里
- transcript 里
- 最终 delivery payload 里

### 4.2 tool call / tool result 会进入 transcript

官方 session deep dive 明确说 transcript 会存：
- 实际 conversation
- tool calls
- compaction summaries

常见链路可以近似理解为：

```text
user
-> assistant(tool_call: exec)
-> toolResult(exec output)
-> assistant(tool_call: read)
-> toolResult(file contents)
-> assistant(final answer)
```

下一轮组装 context 时，这些 entry 会被重新读出并送回模型。

### 4.3 `custom_message` 和 `custom` 不一样

官方文档区分了两类扩展注入：

1. `custom_message`
   - 会进入模型 context
   - 可以隐藏出现在 UI 上

2. `custom`
   - 仅作为扩展状态
   - 不进入模型 context

这说明 OpenClaw 允许插件把“要不要进上下文”作为单独决策。

### 4.4 返回给用户前还有 delivery 整形

OpenClaw 不会机械地把模型最后一个文本块直接发出去。它还会做：
- `NO_REPLY` / `no_reply` 静默回复抑制
- message tool 已发出的内容去重
- partial chunk 的静默抑制
- tool summary 与最终 answer 的整形

所以：

```text
模型最终可见输出
!=
用户最终看到的消息
```

## 5. Context 是怎样拼接起来的

最稳妥的理解不是“字符串拼接”，而是“多层结构组装”。

### 5.1 高层组装顺序

在默认 `legacy` context engine 下，本轮上下文可以近似理解为：

```text
system prompt
+ provider prompt additions
+ tool schemas
+ assembled transcript messages
+ 当前用户 turn / 当前 toolResult turn
```

官方 `Context Engine` 文档把生命周期拆成：
- ingest
- assemble
- compact
- after turn

其中默认 `legacy` engine 的行为是：
- ingest: no-op
- assemble: 走现有 sanitize -> validate -> limit 流水线
- compact: 调用内建 summarization
- after turn: no-op

### 5.2 真正参与“拼接”的几个步骤

更细一点可以拆成：

1. 读取 `sessionKey -> sessionId`
2. 打开 `<sessionId>.jsonl`
3. 必要时修复 session file
4. 执行 transcript hygiene
5. context engine assemble 选出本轮要带的 messages
6. 构建 system prompt
7. 附加 tool schemas
8. provider transport 再做一层请求整形
9. 发模型请求

这个过程里最关键的不是哪一步“concat string”，而是谁有权决定：
- 什么历史要进
- 什么历史只保留在磁盘
- 什么需要修复后才能进 provider API

### 5.3 transcript hygiene 是“拼接前的清洗层”

官方 `Transcript Hygiene` 文档明确说，这些修复发生在“构建模型上下文之前”，并且是内存态修复，不会直接改写磁盘 transcript。

当前公开行为包括：
- image sanitization
- malformed tool call 丢弃
- tool result pairing repair
- turn validation / ordering
- thought signature cleanup
- inter-session provenance marker

并且这些修复是 provider-aware 的：
- OpenAI / Codex：基本只做图片和一部分 reasoning 清理
- Google：会做严格 tool call id、turn alternation、synthetic tool results 等
- Anthropic-compatible：会做 tool result pairing 和 turn alternation 修复

所以“同一份 transcript 发给不同 provider”前，实际可见上下文可能不完全一样。

## 6. 什么真正算 context 开销

官方 `Context` 文档特别强调：不是只有你看得见的文字才占窗口。

会计入 context 的包括：
- system prompt 全部 section
- conversation history
- tool calls
- tool results
- 图片、音频、附件
- compaction summary
- pruning artifacts
- provider wrappers / hidden headers
- tool schemas JSON

这也是为什么 OpenClaw 要专门提供：
- `/status`
- `/context list`
- `/context detail`

来拆分这些开销。

很多人第一次会低估的一项，是：

```text
tool schemas 也算 token，而且经常非常大
```

## 7. Compaction 是怎么做的

### 7.1 Compaction 改的是“下轮可见上下文”，不是删除历史

官方 `Compaction` 文档明确说：
- 老历史会被总结成一个 compact entry
- summary 会写回 transcript
- 最近消息保持原样
- 完整历史仍然保留在磁盘

所以 compaction 不是删历史，而是把“下轮发给模型看的形状”重写。

### 7.2 持久化后的 transcript 形状

compaction 后，后续 run 看到的是：

```text
compaction summary
+ firstKeptEntryId 之后的原始消息
```

官方 deep dive 里提到 `compaction` entry 会带：
- `firstKeptEntryId`
- `tokensBefore`

这保证系统知道“从哪里开始保留原始 tail”。

### 7.3 触发条件

embedded Pi runtime 下，auto-compaction 主要有两个触发条件：

1. overflow recovery
   - provider 返回 context overflow 一类错误
   - 先 compact，再 retry

2. threshold maintenance
   - 成功一轮后
   - 若 `contextTokens > contextWindow - reserveTokens`
   - 则进入压缩维护

注意这里官方文档明确说：这套触发语义由 Pi runtime 决定，OpenClaw 消费这些事件并配合状态维护。

### 7.4 切块时会保护 tool call / toolResult 配对

这是 compaction 最重要的细节之一。

官方文档明确说：
- assistant tool call 与 matching `toolResult` 必须成对保留
- 如果切点落在二者中间，OpenClaw 会把边界移回 assistant tool-call message
- 如果尾部还有未闭合的 tool-result block，会保留该未压缩 tail
- aborted/error tool-call block 不会强行保持 split

这背后的目的很直接：
- 避免模型下轮看到“只看到了工具结果，不知道是谁调用的”
- 或者“只看到了调用，没有看到结果”

### 7.5 关键参数

官方 deep dive 给出的 Pi compaction 典型设置是：

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

OpenClaw 还会额外做一层安全兜底：
- 如果 `reserveTokens` 小于 `reserveTokensFloor`
- 会向上 bump
- 默认 floor 是 `20000`

目的不是“更早压缩”，而是给多轮 housekeeping 和 memory flush 留足余量。

### 7.6 pre-compaction memory flush

这是 OpenClaw 很平台化的一点。

在真正 auto-compaction 前，它还可能先跑一轮“静默记忆落盘”：
- 检测到接近 soft threshold
- 发一个 silent turn
- 要求 agent 把关键状态写进 `memory/YYYY-MM-DD.md` 等 durable storage
- 用 `NO_REPLY` 保证用户看不到这一轮

这说明 OpenClaw 不只是“上下文快满了就压缩”，而是尽量先把长期价值高的内容落到 memory。

### 7.7 Context engine 可以替换 compaction

默认是 `legacy` engine，但官方文档明确支持 context-engine plugin：
- `assemble()`
- `compact()`
- `systemPromptAddition`

也就是说，OpenClaw 允许你换掉：
- 上下文如何选 message
- 旧消息如何总结
- 是否额外给 system prompt 注入动态 recall 提示

## 8. 一个完整的 context 维护心智模型

如果把这整块压缩成一个足够准确的模型，可以记成：

```text
用户输入
  -> Gateway 处理命令/指令/路由
  -> sessionKey / sessionId 定位 transcript
  -> transcript append
  -> hygiene / assemble
  -> build system prompt
  -> attach tool schemas
  -> provider request
  -> assistant/tool/lifecycle streaming
  -> transcript append assistant + toolResult
  -> token/session metadata update
  -> threshold / overflow 时 compact
  -> compaction summary 进入 transcript
```

真正关键的不是“消息最终拼成了什么字符串”，而是：

```text
OpenClaw 用 transcript、context engine、provider hygiene、tool schemas、compaction
共同维护一份可持续运行的模型可见上下文。
```

## 9. 最后给一个判断

OpenClaw 的 context 维护比普通聊天代理复杂得多，关键原因不在模型，而在状态治理：
- 它把 context 当成一级运行时对象来管理
- 它明确区分“磁盘持久化状态”和“当前轮模型可见状态”
- 它允许 provider-aware 清洗和 context-engine 替换
- 它把 compaction 做成 transcript 生命周期的一部分

## 10. 参考

- Context: <https://docs.openclaw.ai/concepts/context>
- Compaction: <https://docs.openclaw.ai/concepts/compaction>
- Session Management Deep Dive: <https://docs.openclaw.ai/reference/session-management-compaction>
- Context Engine: <https://docs.openclaw.ai/concepts/context-engine>
- Transcript Hygiene: <https://docs.openclaw.ai/reference/transcript-hygiene>
