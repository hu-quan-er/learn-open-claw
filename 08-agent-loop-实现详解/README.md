# OpenClaw Agent Loop 实现详解

这个目录专门维护 OpenClaw `agent loop` 的实现说明，重点不是抽象概念，而是“真实执行链路到底怎么跑”。

适用范围：
- 以公开文档 `docs.openclaw.ai` 与公开仓库 `openclaw/openclaw` 当前暴露的源码路径为基础
- 当前工作区不是 OpenClaw 源码本体，因此这里记录的是“基于官方文档与公开路径的实现还原”
- 若某条是分析推断，会明确写出

> **最新实现校正（2026-06-07）**：本目录初版（2026-05-03）把内层循环称为 “Pi SDK / Pi AgentSession”。当前实现已把运行时层**可插拔化**为「agent runtimes / agent harness」：
> - 内置嵌入式 runtime 现在叫 **`openclaw`**（`auto` 模式下未被其它 runtime 认领时的兼容默认；“pi” 是它的旧内部名）。
> - 并列的其它 runtime：**`codex`**（Codex app-server 原生 harness，OpenAI 模型）、**`claude-cli`**（Anthropic CLI 后端）、**`copilot`**、**`acp`**。通过 `agentRuntime.id` 在 model/provider 作用域选择。
> - 外层编排函数去 Pi 化：`runEmbeddedPiAgent` → **`runEmbeddedAgent`**、`subscribeEmbeddedPiSession` → **`subscribeEmbeddedAgentSession`**。
> - 旧的 `docs/pi.md` 已删除，改看 `concepts/agent-runtimes`、`plugins/codex-harness`、`plugins/sdk-agent-harness`。
>
> 下文已据此更新；凡涉及 runtime 归属差异（尤其 Codex 自管 thread/compaction）的地方都会标注。

一句话结论：

```text
OpenClaw 负责外层编排（session、queue、stream、delivery、persistence），
一个可插拔的 agent runtime（默认内置 openclaw，亦可换 codex / claude-cli 等）
负责单次 agent turn 的内层循环（模型推理 -> tool call -> tool result -> 再推理）。
```

建议阅读顺序：
1. [01-主执行链路.md](./01-主执行链路.md)
2. [02-状态-事件-队列.md](./02-状态-事件-队列.md)
3. [03-模拟数据流实例.md](./03-模拟数据流实例.md)
4. [04-源码路径与参考.md](./04-源码路径与参考.md)

相关专题已拆分：
- `context` 与 `system prompt` 已单独迁出到 [../09-context-上下文管理/README.md](../09-context-上下文管理/README.md)

最小心智模型：

```text
agent / CLI 入口
  -> agentCommand
  -> runEmbeddedAgent
  -> runEmbeddedAttempt
  -> createAgentSession
  -> subscribeEmbeddedAgentSession
  -> session.prompt(...)
  -> agent runtime 内层循环（默认 openclaw harness；codex 时交给 app-server）
  -> assistant/tool/lifecycle 事件回流
  -> payloads / usage / transcript 持久化
```

这里最值得记住的不是“它也会 tool calling”，而是三件事：
- 同一个 `sessionKey` 的 run 会被串行化，不会并发踩 transcript
- OpenClaw 自己接管了工具注入、事件桥接、回复整形和静默策略
- transcript 不是一串普通聊天消息，而是带树结构、compaction、tool pairing 的持久状态

快照时间：
- 本目录整理时间：2026-05-03
- 最近一次按最新实现校正：2026-06-07（runtime 可插拔化 + 函数去 Pi 化）
