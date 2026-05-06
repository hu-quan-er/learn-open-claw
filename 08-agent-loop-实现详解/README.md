# OpenClaw Agent Loop 实现详解

这个目录专门维护 OpenClaw `agent loop` 的实现说明，重点不是抽象概念，而是“真实执行链路到底怎么跑”。

适用范围：
- 以公开文档 `docs.openclaw.ai` 与公开仓库 `openclaw/openclaw` 当前暴露的 `docs/pi.md`、相关源码路径为基础
- 当前工作区不是 OpenClaw 源码本体，因此这里记录的是“基于官方文档与公开路径的实现还原”
- 若某条是分析推断，会明确写出

一句话结论：

```text
OpenClaw 负责外层编排（session、queue、stream、delivery、persistence），
Pi SDK 负责单次 agent turn 的内层循环（模型推理 -> tool call -> tool result -> 再推理）。
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
  -> runEmbeddedPiAgent
  -> runEmbeddedAttempt
  -> createAgentSession
  -> subscribeEmbeddedPiSession
  -> session.prompt(...)
  -> Pi 内层 agent loop
  -> assistant/tool/lifecycle 事件回流
  -> payloads / usage / transcript 持久化
```

这里最值得记住的不是“它也会 tool calling”，而是三件事：
- 同一个 `sessionKey` 的 run 会被串行化，不会并发踩 transcript
- OpenClaw 自己接管了工具注入、事件桥接、回复整形和静默策略
- transcript 不是一串普通聊天消息，而是带树结构、compaction、tool pairing 的持久状态

快照时间：
- 本目录整理时间：2026-05-03
