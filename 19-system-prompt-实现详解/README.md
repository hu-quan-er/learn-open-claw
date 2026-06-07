# OpenClaw System Prompt 实现详解

这个目录单独维护 OpenClaw `system prompt` 的设计分析。前面的 `09-context-上下文管理` 已经讲过 context 的整体构成；这里不再把 system prompt 当作 context 的一个子章节，而是把它当成 OpenClaw agent 运行时协议来拆。

适用范围：
- 基于 2026-05-17 可见的官方文档与公开源码路径整理
- 重点关注 `src/agents/system-prompt.ts`、`system-prompt-contribution.ts`、`system-prompt-cache-boundary.ts` 等实现
- 当前工作区不是 OpenClaw 源码本体，因此源码细节以公开 GitHub 与官方文档为准
- 若某条是分析推断，会明确写出

一句话结论：

```text
OpenClaw 的 system prompt 不是固定人设词，而是一个按运行时能力、工具策略、workspace 上下文、
channel 交付协议、安全边界和 prompt cache 边界动态渲染出来的执行契约。
```

建议阅读顺序：
1. [01-构建入口与数据来源.md](./01-构建入口与数据来源.md)
2. [02-section-分层设计.md](./02-section-分层设计.md)
3. [03-运行时条件注入.md](./03-运行时条件注入.md)
4. [04-cache-provider-安全边界.md](./04-cache-provider-安全边界.md)
5. [05-源码路径与参考.md](./05-源码路径与参考.md)

最小心智模型：

```text
agent run
  -> resolve model / auth / workspace / sandbox / tools / skills
  -> collect bootstrap files + runtime metadata + channel capabilities
  -> buildAgentSystemPrompt(...)
  -> stable prefix
       identity + tooling + safety + workspace + docs + stable project context
  -> cache boundary
  -> dynamic suffix
       dynamic project context + messaging/channel guidance + runtime status
  -> provider request
```

这里最值得记住的是三件事：
- `system prompt` 是 OpenClaw owned，不直接复用底层 provider 或 agent runtime 的默认提示词
- prompt 的 section 是按运行时问题组织的，不是按知识目录组织的
- prompt cache 是一等约束，稳定内容和易变内容会被刻意拆开

> **最新实现校正（2026-06-07）**：两点补充。
> 1. **构建是三层**：`buildAgentSystemPrompt`（纯渲染器）+ `resolveAgentSystemPromptConfig`（解析 owner 显示、TTS、model alias、memory citation、delegation 等 config 旋钮）+ runtime adapters（采集 live 事实后调用 prompt facade）。
> 2. **Codex harness 例外**：当 runtime 为 `codex` 时，它不在每轮重复粘贴稳定文件——Codex 内部加载 `AGENTS.md`，其余 workspace 文件（`SOUL.md` / `IDENTITY.md` / `TOOLS.md` / `USER.md`）作为 developer instructions 下发；非 Codex harness 才把 bootstrap 文件正常拼进 prompt。另外 native Codex approval-card 已集成，prompt 会优先引导用原生审批 UI。详见 [04-cache-provider-安全边界.md](./04-cache-provider-安全边界.md) 与记忆专题 [../20-memory-记忆系统/02-加载-vs-检索.md](../20-memory-记忆系统/02-加载-vs-检索.md)。

快照时间：
- 本目录整理时间：2026-05-17
- 最近一次按最新实现校正：2026-06-07
