# OpenClaw Context 上下文管理

这个目录单独维护 OpenClaw 的 `context` 相关实现，不再混在 `agent loop` 专题里。

主题范围：
- 用户输入如何进入 session 与 transcript
- assistant / tool call / tool result 如何进入模型可见上下文
- provider-aware transcript hygiene
- context assemble / compaction / memory flush
- 默认 system prompt 的模块设计与运行时注入

建议阅读顺序：
1. [01-context-维护机制.md](./01-context-维护机制.md)
2. [02-system-prompt-设计.md](./02-system-prompt-设计.md)

一句话结论：

```text
OpenClaw 的 context 不是“聊天历史字符串”，
而是由 transcript、context engine、system prompt builder、tool schemas、provider hygiene、compaction
共同维护的一份“本轮模型可见状态”。
```

快照时间：
- 本目录整理时间：2026-05-03
